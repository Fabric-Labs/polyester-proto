---
services:
  - chain.lifecycle.v1.LifecycleReadService
  - chain.withdraw.v1.WithdrawService
  - chain.withdraw.v1.AutoWithdrawService
kind: changed
breaking: false
wire_compatible: true
---

## What and why

A withdrawal Zipper rejects is no longer shown as lost. Zipper returns the
amount less its network fee to the account's smart account wallet as
zTokens, and keeps the fee. The withdrawal's flow now stays open at its
verdict and ends `STATE_REFUNDED` when that refund arrives, instead of
`STATE_FAILED` at the verdict. A withdrawal whose execution failed after its
destination transaction was sent ends `STATE_FAILED` with reason
`ZIPPER_WITHDRAW_EXECUTION_FAILED`; Zipper reviews it and settles it outside
Polyester Chain, and it is never refunded.

- `chain.lifecycle.v1.RequestFeeStatus` gains `REQUEST_FEE_STATUS_KEPT`: the
  withdrawal did not complete and its fee was charged anyway. A rejected or
  execution-failed withdrawal's `request_fee` is now shown on its summary and
  on its `FLOW_STEP_REFUNDED` or `FLOW_STEP_FAILED` step.
- `FlowStepView.refund_address`: on `FLOW_STEP_REFUNDED` of a withdrawal,
  the smart account wallet the refund went to. That step's `amount_e18` is
  the amount refunded, net of the kept fee, and its `milestone_tx_ref` is the
  refund transaction on Polyester Chain.
- A failed withdrawal's `FLOW_STEP_FAILED` names its destination
  transaction, on the destination chain.
- New `chain.withdraw.v1.WithdrawService.GetWalletBalances`: the zTokens the
  account's smart account wallet holds, such as a refund, read live from
  Polyester Chain.
- `AutoWithdrawStep.network_fee_e18` is also set on a failed step: the fee
  Zipper kept. `STEP_FAILED`, `RUN_PARTIALLY_FAILED` and `RUN_FAILED` no
  longer say the funds are back on the account; `STEP_FAILED` says where they
  are.

## How clients use it

Show a rejected withdrawal from its flow detail:

```http
POST /chain.lifecycle.v1.LifecycleReadService/GetFlowById
{"flowId": "flow_..."}
```

When `summary.currentStep` is `FLOW_STEP_REFUNDED` and `summary.flowKind` is
`KIND_WITHDRAW`, the last observed step carries `amountE18` (refunded),
`refundAddress` (the wallet), `milestoneTxRef` (the refund) and `requestFee`
with `status` `REQUEST_FEE_STATUS_KEPT`. For `FLOW_STEP_FAILED` with
`lifecycleReason` `ZIPPER_WITHDRAW_EXECUTION_FAILED`, show it as under review
and link `milestoneTxRef` on `milestoneTxChainId`.

Read the wallet:

```http
POST /chain.withdraw.v1.WithdrawService/GetWalletBalances
{}

GET /v1/chain/wallet/balances?subaccountId=<public subaccount id>
```

The response lists every nonzero balance, ordered by `zippedAssetId`:
`zippedAssetId`, `unifiedAssetId`, `chainId`, `ztokenAddress`, and
`amountE18` (ConnectRPC, a U128 in 18-decimal units) or `amount` (REST, a
decimal string in whole units). An API key needs `READ_BALANCES`. A failed
chain read answers `UNAVAILABLE` (retry); an account with no wallet answers
`FAILED_PRECONDITION` `SOURCE_SMART_ACCOUNT_UNAVAILABLE`.

## Migration

None required: every change is additive on the wire. Clients that recognised
a rejected withdrawal by `STATE_FAILED` with reason
`ZIPPER_VALIDATION_REJECTED` or `ZIPPER_EXECUTION_REJECTED` should also accept
`STATE_REFUNDED` with the same reason, which new rejections end in.
Withdrawals rejected before this release stay `STATE_FAILED`. Treat an
unknown `RequestFeeStatus` as not settled.
