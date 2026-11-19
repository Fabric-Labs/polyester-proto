---
services:
  - chain.lifecycle.v1.LifecycleReadService
kind: changed
breaking: false
wire_compatible: true
---

## What and why

Every money movement except trading is public in the flows. A transfer
between Trading accounts is recorded on Polyester Chain once it settles. Its
flow now carries that Polyester Chain transaction as `source_tx_hash`, so the
public flow reads find it, as they find deposits and withdrawals:
`GetFlowById`, `ListFlows` and `ListFlowsByTx`, and the public realtime flow
channels. The parties are shown by smart-account address only:
`smart_account_address`, `source_address` and `destination_address`.

Until the recording transaction is observed, the transfer's `source_tx_hash`
is empty and the flow stays private: only the accounts that take part read it,
with `ListAccountFlows` and `GetAccountFlow`. No message, field or RPC is
added or removed. The comments of `source_tx_hash` now describe transfers,
and the descriptions of `GetFlowById`, `ListFlows`, `ListFlowsByTx`,
`ListAccountFlows` and `GetAccountFlow` say "flows that carry a chain
transaction" instead of "flows a chain transaction started".

## How clients use it

Read a transfer flow publicly by its flow id once it has a `source_tx_hash`:

```text
POST /chain.lifecycle.v1.LifecycleReadService/GetFlowById
{"flowId": "flow_Lx2qhJnzTFF"}

GET /v1/chain/flows/flow_Lx2qhJnzTFF
```

List an account's public transfer flows by its smart-account address, or find
the flows of one Polyester Chain transaction:

```text
POST /chain.lifecycle.v1.LifecycleReadService/ListFlows
{"smartAccountAddress": "0xaa7421ea12c0230a1cca966d43a570efc2a482d4", "flowKind": "KIND_TRANSFER"}

GET /v1/chain/flows/by-tx/{tx_hash}/matches
```

One Polyester Chain transaction can record several transfers, so
`ListFlowsByTx` can return several transfer flows for one hash. Paging,
ordering and limits are unchanged.

## Migration

None. Existing fields keep their types. Clients that treat every public flow
as a deposit or withdrawal should expect `KIND_TRANSFER` flows in public
results too, and a client that listed transfers only through
`ListAccountFlows` now also finds them with `ListFlows`.
