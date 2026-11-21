---
services:
  - swap.quickswap.v1.QuickSwapService
  - swap.quickswap.v1.QuickSwapBrokerService
kind: removed
breaking: true
wire_compatible: false
---

## What and why

A deposit sent to a QuickSwap address after the address retired
(`DEPOSIT_ADDRESS_RETIRED`) is now returned automatically to
`return_address`, the same way as a deposit that reaches the address after
its one eligible deposit. When Polyester records the deposit on the
QuickSwap, it used to show as `MANUAL_RECOVERY`
(support-assisted, potentially unrecoverable); it now shows as
`RETURN_ONLY` with reason `DEPOSIT_ADDRESS_RETIRED`, and `direct_return`
reports the return.

The return states are now documented as final:

- `RETURN_PENDING`: queued or being sent. No `tx_hash` is reported, because
  the transaction may still be replaced.
- `RETURN_COMPLETED`: the return is final on the deposit's network;
  `tx_hash` is its transaction. Terminal: it never changes again.
- `RETURN_HELD`: not sent automatically, for example with no return address
  or with the funds no longer at the deposit address, awaiting review, or
  settled with support's help. Review can release it back to
  `RETURN_PENDING`. The broker contacts broker support with the `swap_id`.

`DepositDisposition.MANUAL_RECOVERY` (5) is removed, with no reservation:
no deposit is given it any more. No field or message changed. Buf's wire
rules count the unreserved deletion as a break: regenerated clients no
longer know the number 5 or the name `MANUAL_RECOVERY`.

## How clients use it

A deposit the network refuses before Polyester records it is not listed in
`deposits`, and no QuickSwap shows it or its return; the broker reports its
transaction to broker support with the `swap_id`.

Read `deposits[]` from any response that carries a `QuickSwap`:
`CreateQuickSwap`, `GetQuickSwap` and `QuickSwapBrokerService/GetQuickSwap`.

```json
POST /swap.quickswap.v1.QuickSwapService/GetQuickSwap
{"swapId": "1234"}

{"quickSwap": {"status": "EXPIRED", "isTerminal": true, "deposits": [
  {"index": 0, "txHash": "0x5c1e...", "disposition": "RETURN_ONLY", "reason": "DEPOSIT_ADDRESS_RETIRED",
   "directReturn": {"state": "RETURN_COMPLETED", "txHash": "0x9a40...", "updatedAt": "2026-11-20T06:10:00Z"}}
]}}
```

REST `GET /v1/quickswaps/{swap_id}` returns the same object.

## Migration

- Treat a `RETURN_ONLY` deposit with reason `DEPOSIT_ADDRESS_RETIRED` like
  any other `RETURN_ONLY` deposit: show it as returned only once
  `direct_return.state` is `RETURN_COMPLETED`, as a pending return while it
  is `RETURN_PENDING` or unset, and as held, with a pointer to support,
  while it is `RETURN_HELD`.
- Stop sending users to support for a retired-address deposit unless its
  return is `RETURN_HELD`.
- Clients that matched `MANUAL_RECOVERY` should treat a retired-address
  deposit as `RETURN_ONLY` with reason `DEPOSIT_ADDRESS_RETIRED`, and drop
  the `MANUAL_RECOVERY` case: the generated enum no longer has it.
