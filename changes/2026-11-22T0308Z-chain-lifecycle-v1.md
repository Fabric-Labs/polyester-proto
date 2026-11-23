---
services:
  - chain.lifecycle.v1.LifecycleReadService
kind: changed
breaking: false
wire_compatible: true
---

## What and why

`chain.lifecycle.v1.LifecycleReadService.GetFlowById` now also finds the
flows the ledger settled without a chain transaction: flows that end in their
settlement step (`FLOW_STEP_SETTLEMENT`) with an empty `source_tx_hash`. Each
transfer of an atomic batch transfer between Trading accounts is such a flow,
and so is a single transfer before the Polyester Chain transaction that
records it is observed. Before this change, `GetFlowById` answered "not found"
for them, so a flow id returned for a batch transfer could not be opened in
the explorer. A batch stays a set of individual flows, one per transfer.

These flows carry the same fields as any other transfer flow: the smart
account addresses of both accounts, the amount, the asset, and the ledger
transfer id as `latest_tx_ref` and as the settlement step's
`milestone_tx_ref`. Their `source_tx_hash` is empty, and
`milestone_tx_chain_id` is zero because the ledger transfer id is not a chain
transaction.

The public realtime channel of one flow follows the same rule. Its JSON
channel `public:chain:lifecycle:flow:{flowId}` and its Protobuf channel
`public:chain:lifecycle:flow:{flowId}:proto` now also publish the detail
(`FlowDetailView`) of a flow the ledger settled without a chain transaction.
That update arrives when the flow ends in its settlement step.

Nothing else changes. `ListFlows` and `ListFlowsByTx` still list only the
flows that carry a chain transaction, and so does the public summary channel
`public:chain:lifecycle:flows` (and its `:proto` variant). A flow without a
chain transaction that the ledger has not settled, such as a Trading
withdrawal not yet sent to a chain, is still not found and is not published
publicly; its account reads it with `GetAccountFlow`.

## How clients use it

Read the flow by the `flow_id` the transfer response returned, as for any
other flow:

```bash
curl -sS https://<api-host>/chain.lifecycle.v1.LifecycleReadService/GetFlowById \
  -H 'Content-Type: application/json' -d '{"flowId": "flow_7dXq2Mv9KpR"}'
curl -sS 'https://<api-host>/v1/chain/flows/flow_7dXq2Mv9KpR'
```

Show the flow's transaction link only when `source_tx_hash` is not empty.

To follow a flow live, subscribe to its own channel with the same id, for
example `public:chain:lifecycle:flow:flow_7dXq2Mv9KpR:proto` for Protobuf
`FlowDetailView` updates. A flow the ledger settled is already final when it
first appears there, so read it with `GetFlowById` too; the channel may have
published it before you subscribed.

## Migration

None: the method returns flows it used to answer as not found. A client that
treated "not found" as "this flow is private" for a transfer flow can now link
it.
