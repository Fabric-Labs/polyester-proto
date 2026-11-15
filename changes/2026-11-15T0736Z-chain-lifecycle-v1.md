---
services:
  - chain.lifecycle.v1.LifecycleReadService
kind: changed
breaking: false
wire_compatible: true
---

## What and why

A deposit can now end delivered to the account's own Polyester Chain wallet
instead of its Funding or Trading balance. This happens when the destination
does not accept the credit, for example because deposits into it are paused:
the deposit's zTokens are delivered to the account's wallet, and no Funding or
Trading balance is credited. Such a deposit used to read as in progress
forever; it now ends.

The new `LifecycleReason` value `ZIPPER_DEPOSIT_DELIVERED_TO_WALLET` (106)
marks it. A wallet-delivered deposit reads:

- `is_terminal: true`, `is_open: false`, with `terminal_at_unix_ms` set;
- `current_step: FLOW_STEP_TRANSFER`, the deposit's transfer step, not
  `FLOW_STEP_SETTLEMENT`: no ledger settlement follows;
- `destination_domain: DOMAIN_ZIPPER`, the wallet, instead of
  `DOMAIN_FUNDING` or `DOMAIN_TRADING`;
- `lifecycle_reason: ZIPPER_DEPOSIT_DELIVERED_TO_WALLET` on the summary and on
  the final transfer step. This is the one completed flow that carries a
  reason;
- the final transfer step's `milestone_tx_ref` and `milestone_tx_chain_id`
  name the delivery transaction on Polyester Chain.

The funds are at the flow's `smart_account_address`. `destination_address`
keeps the deposit address, as for every deposit.

## How clients use it

Read flows as before, with `GetFlowById`, `ListFlows` or the realtime flow
channels; nothing new is requested.

```http
POST /chain.lifecycle.v1.LifecycleReadService/GetFlowById
{"flowId": "flow_..."}

GET /v1/chain/flows/{flow_id}
```

Show a terminal deposit whose `lifecycleReason` is
`ZIPPER_DEPOSIT_DELIVERED_TO_WALLET` (or whose `destinationDomain` is
`DOMAIN_ZIPPER`) as completed to the wallet, for example "Delivered to wallet;
Funding was not credited", with `smartAccountAddress` and the final step's
transaction. Do not show it as credited to Funding or Trading.

## Migration

None required: the change adds an enum value, and no field changes meaning.
A client that labels every completed deposit as credited to Funding or
Trading should read `lifecycle_reason` or `destination_domain` first, or it
mislabels these deposits.
