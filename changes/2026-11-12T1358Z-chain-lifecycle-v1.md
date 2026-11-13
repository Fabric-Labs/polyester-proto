---
services:
  - chain.lifecycle.v1.LifecycleReadService
kind: changed
breaking: false
wire_compatible: true
---

## What and why

A deposit that Zipper's compliance screening rejected can be paid back on its
source chain in one of two ways:

- a return to the depositor's return address: the flow ends `REFUNDED`, with a
  `FLOW_STEP_REFUNDED` step;
- a transfer by Zipper's compliance team to an address of its choosing: the
  flow ends `FAILED` with `ZIPPER_COMPLIANCE_TRANSFERRED`, with a
  `FLOW_STEP_FAILED` step.

Zipper pays out the deposit less its fees, and the bridge reports neither the
amount it paid nor the fees. Until now, `FlowStepView.amount_e18` on those two
steps, and `FlowStepActivityView.amount_e18` on their activities, showed the
full deposited amount, which reads as if the whole deposit was paid out.

Those steps and their activities now leave `amount_e18` unset. Polyester never
estimates the paid amount. The flow's summary `amount_e18` and every other
step of the deposit keep the deposited amount. Withdrawals are unchanged: a
rejected withdrawal's `FLOW_STEP_REFUNDED` still shows the refunded amount net
of the kept request fee.

## How clients use it

Read a flow as before and treat an unset step `amountE18` as "not reported":

```http
GET https://<api-host>/v1/chain/flows/{flow_id}
```

```http
POST https://<api-host>/chain.lifecycle.v1.LifecycleReadService/GetFlowById
Content-Type: application/json

{ "flowId": "flow_..." }
```

On a deposit, two `observedSteps` entries have no `amountE18`: the return
step, and the failed step whose `lifecycleReason` is
`ZIPPER_COMPLIANCE_TRANSFERRED`. The step names are spelled differently per
surface:

| Surface | Return step | Compliance transfer step |
| --- | --- | --- |
| REST (`GET /v1/chain/flows/...`) | `"step": "REFUNDED"` | `"step": "FAILED"` |
| ConnectRPC JSON and the lifecycle realtime stream | `"step": "FLOW_STEP_REFUNDED"` | `"step": "FLOW_STEP_FAILED"` |

`lifecycleReason` is `ZIPPER_COMPLIANCE_TRANSFERRED` on every surface. Show
the step's `milestoneTxRef` on the chain named by `milestoneTxChainId`, and
show the deposited amount from `summary.amountE18`, labelled as deposited and
never as paid. The same applies to `ListFlows` and `ListFlowsByTx` results.

## Migration

Existing clients keep decoding every response. A client that showed the step
amount of a returned or compliance-transferred deposit as the amount paid must
handle the field being absent on those steps: show no paid amount, or say the
bridge does not report it, rather than 0 or the deposited amount.
