---
services:
  - chain.lifecycle.v1.LifecycleReadService
  - auth.v1.PolicyService
  - auth.v1.SubaccountRoleService
  - auth.v1.ApiKeyService
kind: removed
breaking: true
wire_compatible: false
---

## What and why

A user's public identity on Polyester Chain is their smart account address.
The public flow reads no longer name or select the Polyester account behind a
flow, and they show only the flows a chain transaction started. An account
reads all of its own flows through two new authenticated reads.

- `owner_account_id` is removed from `FlowSummaryView` (field 1),
  `FlowTxMatchView` (field 19) and the `ListFlowsRequest` account selector
  (field 7). `smart_account_address` stays on every flow and is the only
  public account selector.
- `ListFlows`, `GetFlowById` and `ListFlowsByTx` return only flows that carry
  a `source_tx_hash`. A flow settled without a chain transaction, such as a
  transfer between Trading accounts or a Trading withdrawal not yet sent to a
  chain, is no longer listed, and `GetFlowById` answers not found for it. The
  public realtime channels `public:chain:lifecycle:flows` and
  `public:chain:lifecycle:flow:{flowId}` carry the same flows and no account
  identifier; the account channel `private:chain:lifecycle:flows:{accountPublicId}`
  still carries every flow of the account, without `ownerAccountId`.
- `ListAccountFlows` (`GET /v1/chain/account-flows`) lists the authenticated
  account's own flows and the transfers it receives, including those no chain
  transaction started, with the same filters, ordering and pagination as
  `ListFlows`. `GetAccountFlow` (`GET /v1/chain/account-flows/{flow_id}`)
  returns one flow the account owns or receives, with its `participation`;
  any other flow is not found. Both accept a session token or an API key and
  an optional `subaccount_id`.
- New policy action `READ_FUNDING_FLOWS` (40) and subaccount permission
  `SUBACCOUNT_PERMISSION_READ_FUNDING_FLOWS` (31). Every subaccount role holds
  the permission, and the read-only subaccount policy floor includes the
  action. An API key reads the account flows only when its policy grants
  `READ_FUNDING_FLOWS`. The assistant key presets now include it.

## How clients use it

List the signed-in account's flows, or a subaccount's:

```http
GET /v1/chain/account-flows?subaccountId=<subaccount public ID>&scope=ALL&limit=50
Authorization: Bearer <session token>
```

```http
POST /chain.lifecycle.v1.LifecycleReadService/ListAccountFlows
Content-Type: application/json

{"subaccountId": "<subaccount public ID>", "scope": "LIST_ALL", "limit": 50, "participation": "PARTICIPATION_OWNER"}
```

Read one of its flows: `GET /v1/chain/account-flows/{flow_id}`, or
`POST /chain.lifecycle.v1.LifecycleReadService/GetAccountFlow` with
`{"flowId": "flow_…"}`. A page token is bound to the account and the request it
came from; `limit` defaults to 100 and is at most 500.

Public explorer reads select an account by smart account address:

```http
GET /v1/chain/flows?smartAccountAddress=0x5aaeb6053f3e94c9b9a09f33669435e7ef1beaed&scope=ALL
```

## Migration

1. Stop reading `ownerAccountId` from flow summaries, transaction matches and
   realtime payloads; identify the account by `smartAccountAddress`.
2. Replace `ListFlows` with `owner_account_id` by `ListAccountFlows`, signed as
   the account (with `subaccount_id` for a subaccount). A REST request with
   `ownerAccountId` is now refused.
3. Show an account its own history, including transfers between Trading
   accounts and Trading withdrawals not yet on chain, from `ListAccountFlows`
   and `GetAccountFlow`; the public reads no longer return those flows.
4. API keys that read funding history need a policy granting
   `READ_FUNDING_FLOWS`.
