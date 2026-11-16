---
services:
  - swap.quickswap.v1.QuickSwapBrokerService
  - swap.quickswap.v1.QuickSwapReferralService
  - swap.quickswap.v1.QuickSwapService
kind: changed
breaking: false
wire_compatible: true
---

## What and why

Documentation only: no message, field, enum value, validation rule or
behaviour changes.

- The product is spelled QuickSwap everywhere in the QuickSwap contract. The
  referral service and the broker limits previously wrote "Quick Swap".
- Every long broker and referral method comment now starts with a one-line
  summary, followed by its behaviour and its errors as a list:
  `TransferOwnership`, `ListQuickSwaps`, `GetQuickSwap` (broker),
  `GetFeeTotals`, `GetBrokerStats`, `ListPayouts`, `GetPayout`,
  `ListUpcomingPayouts`, `ListVaultPayoutRules`, `ListVaultLedger`,
  `ListVaultAllowlist`, `RequestVaultAllowlistChange`,
  `ListReferredQuickSwaps` and `GetReferredQuickSwap`.
- Errors that were returned but not documented are now listed:
  - `ListVaultBalances`: `INVALID_ARGUMENT` for a `page_token` from another
    broker or vault filter, or malformed (its errors are now a list).
  - `ListMyBrokers`: `PERMISSION_DENIED` / `ACTION_DENIED` for a broker API
    key.
  - `ListQuickSwaps` and `GetFeeTotals`: `INVALID_ARGUMENT` when
    `vault_account_id` is not a public sub-account ID.
- `EnrollReferralPartnerRequest.display_name`: leading and trailing spaces
  are removed, and at least one other character must remain. The previous
  comment said such spaces were not allowed; requests were always trimmed.
- REST: the summary of `GET /v1/quickswaps:lookup` is now "Get QuickSwap by
  idempotency key".

## How clients use it

Nothing changes in how clients call these services. For example, the call
whose documented errors grew still reads:

```bash
curl -sS https://<api-host>/swap.quickswap.v1.QuickSwapBrokerService/ListQuickSwaps \
  -H 'Authorization: Bearer <broker-api-key>' \
  -H 'Content-Type: application/json' \
  -d '{"broker": "acme", "vaultAccountId": "0", "limit": 50}'
curl -sS 'https://<api-host>/v1/quickswap/brokers/acme/quickswaps?limit=50' \
  -H 'Authorization: Bearer <broker-api-key>'
```

## Migration

None: the wire contract and the service behaviour are unchanged; only the
documentation is corrected and restructured.
