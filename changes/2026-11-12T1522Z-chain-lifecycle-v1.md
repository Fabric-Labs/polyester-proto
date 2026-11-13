---
services:
  - chain.lifecycle.v1.LifecycleReadService
  - chain.withdraw.v1.AutoWithdrawService
kind: changed
breaking: false
wire_compatible: true
---

## What and why

Sometimes the bridge commits a withdrawal's destination transaction and then
fails the withdrawal's execution. The flow ends `FAILED` with the lifecycle
reason `ZIPPER_WITHDRAW_EXECUTION_FAILED`. This is not a refund: nothing goes
back to the account, the request fee is kept, and the bridge operator reviews
the withdrawal. The user may or may not have received the funds on the
destination chain.

Until now, `zipperReason` on such a flow showed the generic text of the code
the bridge gave, for example "The request could not be processed." That text
said nothing about the review or the funds.

`zipperReason` on these flows now reads:

- `code`: the bridge's code, unchanged. A code this API does not define still
  reads as its unmapped code.
- `reasonId`: `withdrawal.execution_failed.under_review`, whatever the code.
- `message`: "Failed, under review by the bridge operator. The funds were not
  returned to your account."

The flow's summary has these values, and so does its `FLOW_STEP_FAILED` step.
They are set even when the bridge gave no code, in which case `code` is
unset. The `FLOW_STEP_FAILED` step's `milestoneTxRef` is
the destination transaction, on the chain named by `milestoneTxChainId`, as
before.

The documentation of `AutoWithdrawStep.tx_hash` is corrected. It said the hash
was set once the withdrawal completed. In fact it is set once the destination
transaction is committed. A step whose withdrawal's execution failed after that
commit fails with `withdrawal_failed` and keeps the hash.

## How clients use it

Read a flow as before:

```http
GET https://<api-host>/v1/chain/flows/{flow_id}
```

```http
POST https://<api-host>/chain.lifecycle.v1.LifecycleReadService/GetFlowById
Content-Type: application/json

{ "flowId": "flow_..." }
```

On a flow whose `summary.lifecycleReason` is
`ZIPPER_WITHDRAW_EXECUTION_FAILED`, show `summary.zipperReason.message`, or
localize it with `reasonId`. Next to it, link the `FLOW_STEP_FAILED` step's
`milestoneTxRef` using the explorer of the chain whose `chain_id` is that
step's `milestoneTxChainId`. Do not show the withdrawal as refunded, and do not
promise when or how the review ends. The same applies to `ListFlows`,
`ListFlowsByTx` and the lifecycle realtime stream. On REST the step is
`"step": "FAILED"`. In ConnectRPC JSON and the realtime stream it is
`"step": "FLOW_STEP_FAILED"`.

For auto-withdraw, `ListAutoWithdrawRuns`
(`POST /chain.withdraw.v1.AutoWithdrawService/ListAutoWithdrawRuns`) is
unchanged. A failed step's `txHash`, when set, is the destination transaction
under review.

## Migration

None. Every field and value already existed. A client that maps `code` to its
own text should use `reasonId` or `message` for these flows instead, so it shows
the review rather than the code's generic text.
