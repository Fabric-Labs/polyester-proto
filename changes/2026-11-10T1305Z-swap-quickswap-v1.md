---
services:
  - swap.quickswap.v1.QuickSwapService
  - swap.quickswap.v1.QuickSwapBrokerService
kind: changed
breaking: false
wire_compatible: true
---

## What and why

A QuickSwap whose deposit is returned because its risk screen failed
(`SOURCE_SCREENING_FAILED`) now ends. Once the deposit has been sent back to
`return_address`, the QuickSwap moves from `RETURNING` to `RETURNED` with the
same reason, and `terminal_at` is when the return was made. Before, it stayed
`RETURNING`.

That QuickSwap lists one `Withdrawal`: role `RETURN`, state `FINALIZED`,
`tx_hash` the return's transaction on the deposit's network and
`finalized_at` its time. The return is sent from the deposit address and
reports no delivered amount, so its `amount` and `network_fee` are unset. Its
`flow_id` is the deposit's flow (`Deposit.flow_id`), which ends `REFUNDED`
with the return as its transaction. Only comments changed in the contract.

## How clients use it

Read the QuickSwap as before and stop polling once `is_terminal` is true:

```http
POST https://<api-host>/swap.quickswap.v1.QuickSwapService/GetQuickSwap
Authorization: Bearer <broker API key>
Content-Type: application/json

{"swapId": "<swap_id>"}
```

A returned deposit reads `"status": "RETURNED"`, `"reason":
"SOURCE_SCREENING_FAILED"`, with `withdrawals[0]` a `RETURN` whose `txHash`
links the return on the deposit's network. `GET /v1/chain/flows/{flow_id}`
with that withdrawal's `flowId` shows the deposit's flow ending `REFUNDED`.

## Migration

None: no field or value changed. Clients that show a delivered amount for
every `FINALIZED` withdrawal should handle an unset `amount` on this `RETURN`
and show the deposit's amount instead. Clients that read a withdrawal's
`flow_id` as a withdraw flow should accept a deposit flow here.
