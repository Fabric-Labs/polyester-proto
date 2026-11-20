---
services:
  - swap.quickswap.v1.QuickSwapBrokerService
kind: changed
breaking: false
wire_compatible: true
---

## What and why

`ListPayouts` filtered by `statuses` now does a bounded amount of work per
page, however many of a vault's payouts have another status. As a result, a
page may hold fewer than `limit` payouts, or none at all, while
`next_page_token` is set. Only an empty `next_page_token` means the list has
ended. Messages, fields, the order and the page token format are unchanged.
`ListVaultLedger` already pages this way.

`next_page_token` marks a position in the run history, in the list order.
Without a status filter it is the last payout returned. On any page filtered
by status, a full page included, the position can lie past the last payout
returned: when the page uses up its scan before finding another match, it is
the last payout run the page examined.

## How clients use it

Keep calling with the returned `next_page_token` until it comes back empty,
whatever the size of each page:

```http
POST /swap.quickswap.v1.QuickSwapBrokerService/ListPayouts
{"broker": "acme", "statuses": ["PAYOUT_PAID"], "limit": 50}
```

```http
GET /v1/quickswap/brokers/acme/payouts?status=PAYOUT_PAID&limit=50
```

Each page is newest first by scheduled slot. A page with fewer than `limit`
payouts, or none, and a non-empty `next_page_token` is normal, even when
matching payouts remain farther back; request the next page.

A complete traversal, followed until `next_page_token` is empty, lists every
payout that matched throughout it exactly once, in order. A
payout started meanwhile past the token's position, for an older scheduled
slot, is listed by a later page. A payout started meanwhile at or before the
position, including one between the last payout returned and the position,
appears only in a listing started again from the first page. So does a payout
there whose status changes meanwhile into the filter, for example from
`PAYOUT_IN_PROGRESS` to `PAYOUT_PAID`. To pick up changes at or ahead of the
token, start again without a `page_token`.

## Migration

None for a client that pages until `next_page_token` is empty, as the
contract already said. A client that stops paging when a page holds fewer
than `limit` payouts, or none, must page on the token instead. A client that
expects a continuation to pick up a payout started meanwhile, or one whose
status changed into the filter, just after the last payout it received must
start a new listing from the first page instead, because a filtered token can
mark a later position, even after a full page.
