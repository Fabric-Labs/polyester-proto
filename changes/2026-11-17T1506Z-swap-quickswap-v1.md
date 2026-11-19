---
services:
  - swap.quickswap.v1.QuickSwapService
  - swap.quickswap.v1.QuickSwapBrokerService
kind: added
breaking: false
wire_compatible: true
---

## What and why

A deposit that reaches a QuickSwap address after its one eligible deposit,
or after its funding window, is never credited: it shows as `RETURN_ONLY`
and is returned directly to `return_address`, not through withdrawals.
Until now the QuickSwap did not say whether that return happened.

`Deposit` gains `direct_return` (`DepositReturn`), set once the return is
recorded:

- `state` (`DepositReturnState`): `RETURN_PENDING` (queued or being sent),
  `RETURN_COMPLETED` (sent; `tx_hash` is the transaction) or `RETURN_HELD`
  (held for review; the broker contacts broker support with the `swap_id`).
- `tx_hash`: the return transaction on the deposit's network, in its native
  text form, once `RETURN_COMPLETED`.
- `updated_at`: when the return last changed state.

It is unset on every other deposit and until a return is recorded. The
deposit's disposition and the QuickSwap's status, terminal flag and
withdrawals do not change.

## How clients use it

Read it from any response that carries a `QuickSwap`:
`CreateQuickSwap`, `GetQuickSwap` and `QuickSwapBrokerService/GetQuickSwap`.

```json
POST /swap.quickswap.v1.QuickSwapService/GetQuickSwap
{"swapId": "1234"}

{"quickSwap": {"status": "COMPLETED", "deposits": [
  {"index": 0, "disposition": "ELIGIBLE"},
  {"index": 1, "txHash": "D349AF91...", "disposition": "RETURN_ONLY", "reason": "DEPOSIT_ADDRESS_ALREADY_USED",
   "directReturn": {"state": "RETURN_COMPLETED", "txHash": "FAE1EC33...", "updatedAt": "2026-11-16T22:17:50.049Z"}}
]}}
```

REST `GET /v1/quickswaps/{swap_id}` returns the same object as
`deposits[].directReturn` with `state`, `txHash` and `updatedAt`
(RFC 3339).

## Migration

None: the field and enum are new, and an absent `direct_return` means what
`RETURN_ONLY` meant before.
