---
services:
  - swap.quickswap.v1.QuickSwapService
  - swap.quickswap.v1.QuickSwapBrokerService
  - swap.quickswap.v1.QuickSwapReferralService
kind: changed
breaking: true
wire_compatible: true
---

## What and why

The QuickSwap request rules now publish limits the service already enforced
or documented, so clients and SDK validators catch a bad request before
sending it. No field, message or enum value changes.

`QuoteQuickSwap` and `CreateQuickSwap`:

- `affiliates`: at most 3, each vault named once.
- `AffiliateShare.account_id`: non-zero. `AffiliateShare.bps`: 1 to 300.
- `broker_bps`: at most 300.
- `broker_bps` plus the affiliates' `bps`: at most 300.
- `referral_code` cannot be combined with `affiliates`.

These used to fail with `INVALID_ARGUMENT` and the error code
`FEES_INVALID`. They now fail with `INVALID_ARGUMENT` and field violations,
on ConnectRPC and REST alike. `FEES_INVALID` remains for what only the
service can judge: an affiliate `account_id` that is not a public
sub-account ID, and a `referral_code` whose discount and commission for the
route exceed `broker_bps`.

`QuickSwapBrokerService`:

- `CreateKey.label` refuses any leading or trailing Unicode whitespace
  (tab, vertical tab, no-break space, the Unicode space separators and the
  line and paragraph separators), which the service already refused after
  validation.
- `CreateAffiliateVault.label` and `RequestVaultAllowlistChange.label` refuse
  leading or trailing ASCII whitespace, as their documentation already said.
  A padded affiliate vault or allowlist label previously failed later
  without a validation error.
- `RequestVaultAllowlistChange` refuses a `label` with `REMOVE_DESTINATION`.
- `ListQuickSwaps`, `ListPayouts` and `ListVaultLedger` refuse an
  `end_time` at or before `start_time` when both are set.

`QuickSwapReferralService`:

- `ListReferredQuickSwaps` refuses an `end_time` at or before `start_time`
  when both are set.

## How clients use it

Valid requests are unchanged. A refused request now looks like this:

```bash
curl -sS https://<api-host>/swap.quickswap.v1.QuickSwapService/QuoteQuickSwap \
  -H 'Authorization: Bearer <broker-api-key>' \
  -H 'Content-Type: application/json' \
  -d '{"terms": {...}, "brokerBps": 200,
       "affiliates": [{"accountId": "7", "bps": 150}]}'
# HTTP 400, Connect code invalid_argument; the message names the rule:
# broker_bps and the affiliates' bps must total at most 300

curl -sS -X POST https://<api-host>/v1/quickswaps:quote \
  -H 'Authorization: Bearer <broker-api-key>' \
  -H 'Content-Type: application/json' \
  -d '{"terms": {...}, "brokerBps": 200, "affiliates": [{"accountId": "7", "bps": 150}]}'
# 400 application/problem+json with "code": "INVALID_ARGUMENT"
```

## Migration

1. Clients that branch on `FEES_INVALID` for the fee shapes listed above
   should handle `INVALID_ARGUMENT` without that error code for them, or
   check the shapes before sending. `FEES_INVALID` itself still exists.
2. Send labels trimmed, and no label when removing an allowlist entry.
3. Send `end_time` later than `start_time`, or leave either unset.
