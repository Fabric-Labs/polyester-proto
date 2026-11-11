---
services:
  - swap.quickswap.v1.QuickSwapBrokerService
  - swap.quickswap.v1.QuickSwapReferralService
kind: changed
breaking: false
wire_compatible: true
---

## What and why

Broker and referral list page tokens are now issued in a new format. Like
the payout and vault-balance tokens, they carry only public values. This
covers `ListVaults`, `ListKeys` and `ListQuickSwaps` on
`QuickSwapBrokerService`, and `ListReferredQuickSwaps` on
`QuickSwapReferralService`. Tokens stay opaque strings and their length
limits are unchanged.

The list order documentation now matches what the API returns:

- `ListVaults` lists vaults in a stable server-defined order, the same on
  every page. It does not follow the public `account_id` values.
- `ListQuickSwaps` lists QuickSwaps newest first by creation time. Ties
  follow a stable server-defined order, not `swap_id`.
- `ListPayouts` lists payouts newest first by scheduled slot. Runs of the
  same slot follow a stable server-defined order, not `run_id`.

The order itself has not changed; only its description has.

## How clients use it

Page as before: pass the previous response's `next_page_token` as
`page_token`, with the same filters, until `next_page_token` is empty.

```http
POST /swap.quickswap.v1.QuickSwapBrokerService/ListQuickSwaps
{"broker": "acme", "limit": 100}

POST /swap.quickswap.v1.QuickSwapBrokerService/ListQuickSwaps
{"broker": "acme", "limit": 100, "pageToken": "<nextPageToken>"}
```

Over REST, `GET /v1/quickswap/brokers/{broker}/quickswaps?pageToken=<nextPageToken>`
works the same way. A token is valid only for the broker or referral
partner, the filters and the list that issued it. Any other token is
`INVALID_ARGUMENT`.

## Migration

None in code. A page token issued before this release is rejected with
`INVALID_ARGUMENT`. A client that holds one across the release restarts that
listing from the first page. Do not sort a page client-side by `swap_id`,
`run_id` or `account_id` to reproduce the list order: the order those IDs
give is not the API's order.
