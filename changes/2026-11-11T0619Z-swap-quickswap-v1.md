---
services:
  - swap.quickswap.v1.QuickSwapBrokerService
kind: added
breaking: false
wire_compatible: true
---

## What and why

A broker's payout history now shows how each payout step's withdrawal
closed, so a partner reading a failed payout sees where its funds went
without reading the vault ledger.

- `PayoutStep` gains `close` (`PayoutClose`, field 12) and `refunded_amount`
  (`TokenAmount`, field 13), on `ListPayouts` and `GetPayout`.
- `close` is the same value as the step's `PAYOUT_DEBIT` entry in
  `ListVaultLedger`: `CLOSE_PAID` for a completed step, and for a failed one
  `CLOSE_REFUNDED_TO_SMART_ACCOUNT`, `CLOSE_SENT_TO_RECOVERY`, or
  `CLOSE_UNKNOWN` when where its funds went is not known (a failed payout
  that ended before this was recorded, among others). It is unset
  (`CLOSE_UNSPECIFIED`) while the step is not final, and on a step that never
  left the vault, such as a skipped, refused, held or released one.
- `refunded_amount` is the net refunded to the vault's own smart account, the
  amount of the step's `PAYOUT_REFUND` ledger entry. It is set only with
  `CLOSE_REFUNDED_TO_SMART_ACCOUNT`; otherwise it is absent, never zero.
- `PayoutClose.CLOSE_UNSPECIFIED` is now returned on payout steps (still
  never on a ledger entry), and the comments on `PayoutStatus.PAYOUT_FAILED`,
  `PayoutStep.reason` and `CLOSE_REFUNDED_TO_SMART_ACCOUNT` point to these
  fields.

## How clients use it

```http
POST https://<api-host>/swap.quickswap.v1.QuickSwapBrokerService/GetPayout
Authorization: Bearer <broker API key or member session token>
Content-Type: application/json

{"broker": "acme", "runId": "7468227057549893604"}
```

REST: `GET /v1/quickswap/brokers/{broker}/payouts/{run_id}` and
`GET /v1/quickswap/brokers/{broker}/payouts`. A refunded step reads, in part:

```json
{"state": "PAYOUT_STEP_FAILED", "reason": "withdrawal_failed",
 "amount": {"baseUnits": "500", "decimals": 18},
 "chargedNetworkFee": {"baseUnits": "30", "decimals": 18},
 "close": "CLOSE_REFUNDED_TO_SMART_ACCOUNT",
 "refundedAmount": {"baseUnits": "470", "decimals": 18}}
```

Paging, filters and ordering are unchanged: payouts newest first by
scheduled slot, runs of the same slot in a stable server-defined order, the
same on every page.

## Migration

None: the fields are new. A client that read `ListVaultLedger` to learn a
failed payout's close can read it from the payout step instead; both reads
agree for every step.
