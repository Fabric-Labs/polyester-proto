---
services:
  - swap.quickswap.v1.QuickSwapService
  - swap.quickswap.v1.QuickSwapBrokerService
  - chain.lifecycle.v1.LifecycleReadService
kind: changed
breaking: false
wire_compatible: true
---

## What and why

A QuickSwap whose deposit was blocked by its risk screen
(`SOURCE_SCREENING_BLOCKED`) was `HELD` for good. Zipper's compliance team
can transfer such funds out, on the network the deposit was received on, to
an address of its choosing, which is not necessarily the user's. The
QuickSwap now ends at that transfer, with a new terminal status and a new
withdrawal role. Neither one says the funds were returned or refunded,
because they were not.

- `QuickSwapStatus.CLOSED` (17), terminal: the blocked deposit's funds were
  transferred out by Zipper's compliance team. `reason` stays
  `SOURCE_SCREENING_BLOCKED`, `is_terminal` is true, and `terminal_at` is
  when the transfer was made. The deposit keeps disposition `ON_HOLD`.
- `WithdrawalRole.COMPLIANCE_TRANSFER` (4): the QuickSwap's one withdrawal,
  `FINALIZED`. `tx_hash` is the transfer's transaction on the deposit's
  network and `finalized_at` its time. `flow_id` is the deposit's flow
  (`Deposit.flow_id`). `address`, `memo`, `amount` and `network_fee` are
  unset: Polyester records neither the recipient nor the amount.
- `chain.lifecycle.v1.LifecycleReason.ZIPPER_COMPLIANCE_TRANSFERRED` (105):
  a blocked deposit's flow now stays open after the block, and ends `FAILED`
  with this reason once the transfer is made, its `FAILED` step naming the
  transfer. Before, it ended `FAILED` at the block.

## How clients use it

Read the QuickSwap as before and stop polling once `is_terminal` is true:

```http
POST https://<api-host>/swap.quickswap.v1.QuickSwapService/GetQuickSwap
Authorization: Bearer <broker API key>
Content-Type: application/json

{"swapId": "<swap_id>"}
```

A transferred deposit reads `"status": "CLOSED"`, `"reason":
"SOURCE_SCREENING_BLOCKED"`, `"isTerminal": true`. `withdrawals[0]` is a
`COMPLIANCE_TRANSFER` whose `txHash` links the transfer on the deposit's
network. Show it as "closed after compliance review: the funds were
transferred by the compliance team to an address of its choosing". Never
show it as returned or refunded, and do not show `returnAddress` as its
recipient. `GET /v1/chain/flows/{flow_id}` with that withdrawal's `flowId`
shows the deposit's flow ending `FAILED` with reason
`ZIPPER_COMPLIANCE_TRANSFERRED`.

`ListQuickSwaps` and the broker statistics accept and report `CLOSED` like
the other statuses.

## Migration

Clients that switch on `QuickSwapStatus` or `WithdrawalRole` must handle
`CLOSED` and `COMPLIANCE_TRANSFER`. Treat `CLOSED` as terminal (or rely on
`is_terminal`). A `COMPLIANCE_TRANSFER` has no recipient and no amount.
Clients that treat a blocked deposit's flow as ended at the block should
expect it to stay open until the transfer, then end `FAILED` with
`ZIPPER_COMPLIANCE_TRANSFERRED`.
