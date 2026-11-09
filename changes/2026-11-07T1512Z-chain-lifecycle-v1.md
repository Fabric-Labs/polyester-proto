---
services:
  - chain.lifecycle.v1.LifecycleReadService
kind: changed
breaking: true
wire_compatible: true
---

## What and why

`ListFlows` with an account selector (`owner_account_id` or
`smart_account_address`) now returns the transfers the account received,
not only the flows it owns. Before, a transfer appeared only in the
sender's list, so transfers from other users, transfers between two
subaccounts of one main account, and reward payouts were missing from the
recipient's list.

- New enum `FlowParticipation`: `PARTICIPATION_OWNER` (the account owns the
  flow: its deposit, withdrawal, or a transfer it sent, including a move
  between its own Funding and Trading balances) and
  `PARTICIPATION_RECIPIENT` (a transfer another account sent to it).
- New `FlowSummaryView.participation`: set on every `ListFlows` row when the
  request selects an account. Each flow appears once, with one
  participation; ownership wins, so a move within one account is
  `PARTICIPATION_OWNER`. Unspecified on unscoped lists, flow detail and
  realtime updates.
- New `ListFlowsRequest.participation` filter: unspecified returns both
  sides; `PARTICIPATION_OWNER` returns only owned flows (the previous
  result set); `PARTICIPATION_RECIPIENT` returns only received transfers.
  It requires an account selector; without one the request fails with
  `INVALID_ARGUMENT`.

Scope, sort, ordering timestamp, limits and keyset paging are unchanged and
apply to both sides together. A `page_token` is now bound to the request it
was returned for: sent with a different account selector, participation,
filter, scope, sort or ordering timestamp it fails with `INVALID_ARGUMENT`
instead of resuming another list at the wrong position. Only `limit` may
change between pages, and tokens issued before this change are refused. Only transfers have a recipient: deposits and
withdrawals stay owner-only.

## How clients use it

List an account's recent activity, sent and received, newest first:

```http
POST https://<api-host>/chain.lifecycle.v1.LifecycleReadService/ListFlows
Content-Type: application/json

{"ownerAccountId":"6789012345678","scope":"LIST_ALL","limit":20}
```

```http
GET https://<api-host>/v1/chain/flows?ownerAccountId=45LUmqjT&scope=ALL&limit=20
```

Each row carries `participation`. For a transfer, show
`PARTICIPATION_OWNER` as sent and `PARTICIPATION_RECIPIENT` as received;
the counterparty is `destinationAddress` for a sent transfer and
`sourceAddress` (with `ownerAccountId`) for a received one.

Only received transfers:

```http
GET https://<api-host>/v1/chain/flows?smartAccountAddress=0x...&participation=RECIPIENT
```

Continue with `pageToken` from `nextPageToken`, keeping the same filters and
`participation`.

## Migration

For the previous owner-only list, pass `participation=OWNER` (`PARTICIPATION_OWNER`).
