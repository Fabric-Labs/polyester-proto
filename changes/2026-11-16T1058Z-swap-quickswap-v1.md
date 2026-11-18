---
services:
  - swap.quickswap.v1.QuickSwapService
  - swap.quickswap.v1.QuickSwapBrokerService
kind: changed
breaking: false
wire_compatible: true
---

## What and why

Comment-only corrections to the QuickSwap reference text, so it matches what
the API already does. No field, message, enum value or behavior changes.

- `QuoteQuickSwap` and `CreateQuickSwap` list every code they return:
  `REFERRAL_CODE_INVALID` and `REFERRAL_NOT_SUPPORTED` (INVALID_ARGUMENT), and
  `ROUTE_SUSPENDED`, `REFERRAL_CODE_UNKNOWN` and `REFERRAL_PARTNER_INACTIVE`
  (FAILED_PRECONDITION).
- The REST status list says 409 for `ALREADY_EXISTS` (`IDEMPOTENCY_CONFLICT`)
  and `ABORTED` (`ADDRESS_CONFLICT`), and 503 for both `UNAVAILABLE` and
  `DEADLINE_EXCEEDED`. `QuickSwapBrokerService` notes its one exception:
  `CreateKey` and `RotateKey` answer `DEADLINE_EXCEEDED` with 504.
- `VAULT_NOT_FOUND` covers any vault of the broker, the default vault
  included; `GetVault` and `SetVaultStatus` take affiliate vaults only.
- `charged_fees` (on `QuickSwap` and `QuickSwapSummary`) and the quote's fee
  note include referral fees.
- `ExecutionProgress.slices_unavailable_reason`: with `PRECISION_UNAVAILABLE`,
  an aggregate amount in a token whose precision is unavailable is unset;
  every other aggregate stays available.
- The broker role table lists `ListVaultLedger` as a read every role and API
  key may call.
- `BrokerRole.FINANCE` describes what Finance does: reads, the payout address
  allowlist, payout rules and withdrawals of the broker's vaults.
- `Broker.default_vault_account_id`: the default vault receives every broker
  fee.

## How clients use it

Nothing changes in calls or responses. For example, a create whose referral
code belongs to an inactive partner still returns FAILED_PRECONDITION with
`REFERRAL_PARTNER_INACTIVE`:

```bash
curl -sS https://<api-host>/swap.quickswap.v1.QuickSwapService/CreateQuickSwap \
  -H 'Authorization: Bearer <broker-api-key>' \
  -H 'Content-Type: application/json' -d '{...,"referralCode":"<code>"}'
# REST: POST /v1/quickswaps with the same body answers HTTP 400,
# problem code REFERRAL_PARTNER_INACTIVE
```

## Migration

None: only documentation text changes.
