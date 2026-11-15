---
services:
  - swap.quickswap.v1.QuickSwapService
  - swap.quickswap.v1.QuickSwapBrokerService
kind: changed
breaking: true
wire_compatible: true
---

## What and why

Rarely, a deposit that was dropped or replaced on its network confirms after
all, after the end user's replacement deposit already took the QuickSwap. Its
funds are then held for review. The QuickSwap now lists that deposit in
`deposits` as its own entry, `ON_HOLD` (reason `REASON_UNSPECIFIED`: the
broker contacts Polyester broker support with the `swap_id`). The replacement
stays the deposit that executes: its entry, the execution, the withdrawals
and the QuickSwap's status are unchanged, also on a terminal QuickSwap such as
`COMPLETED`.

`deposits` stays oldest first, so the held deposit, detected before its
replacement, usually appears before it and shifts its `index` by one.
`Deposit.index` was documented as stable for the QuickSwap's life; it is not:
a deposit detected earlier can appear later and shift the index of every
deposit after it. `Withdrawal.deposit_index` now names the index, in the same
response, of the deposit the withdrawal moves; before, it could read 0 even
when that deposit was not first in `deposits`.

No field, enum value or error code changes.

## How clients use it

Read `GetQuickSwap` as before. Identify each deposit by `tx_hash` with
`tx_occurrence_index`, never by `index`, and resolve a withdrawal's deposit
through `deposit_index` within the same response only.

```http
POST /swap.quickswap.v1.QuickSwapService/GetQuickSwap
{"swapId": "..."}

200 {"quickSwap": {"status": "COMPLETED", "deposits": [
  {"index": 0, "txHash": "0xaaa...", "txOccurrenceIndex": "0", "disposition": "ON_HOLD"},
  {"index": 1, "txHash": "0xbbb...", "txOccurrenceIndex": "0", "disposition": "ELIGIBLE"}],
  "withdrawals": [{"index": 0, "role": "DESTINATION", "depositIndex": 1, ...}], ...}}
```

```http
GET /v1/quickswaps/{swap_id}
```

## Migration

- A client that keys stored deposits by `index`, or matches deposits across
  reads by position, must key them by `tx_hash` and `tx_occurrence_index`
  instead.
- A client that took `deposits[0]` as the deposit that executes must stop
  doing so: once accepted, that deposit is the one whose `disposition` is
  `ELIGIBLE`, and each withdrawal's `deposit_index` names the deposit it moves
  in the same response.
- A client that assumed a terminal QuickSwap's `deposits` never changes must
  accept a new `ON_HOLD` entry on it; the QuickSwap's status does not change.
