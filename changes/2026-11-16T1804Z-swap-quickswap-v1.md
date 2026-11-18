---
services:
  - swap.quickswap.v1.QuickSwapExplorerService
kind: added
breaking: false
wire_compatible: true
---

## What and why

A new public, read-only service, `QuickSwapExplorerService`, lists completed
QuickSwaps of every broker, Polyester's own QuickSwap app included, for a
public explorer page. It needs no credentials.

- `ListCompletedQuickSwaps` lists completed QuickSwaps only, newest
  completion first, optionally by source or destination asset and network.
  In-progress, refunded, returned and expired QuickSwaps are never listed.
- Each `ExplorerQuickSwap` carries only public facts: `swap_id`,
  `completed_at`, `source_zipped_asset_id`, `destination_zipped_asset_id`,
  `source_amount` (deposited), `destination_amount` (delivered) and `legs`
  (the spot markets traded, by `symbol_id`; none for a bridge). It carries no
  address, memo, transaction hash, broker, fee, idempotency key or quote.
  More fields may be added later.
- New `ErrorCode` value `PAGE_TOKEN_INVALID` (INVALID_ARGUMENT, HTTP 400): a
  `page_token` that is malformed or was returned for other filters.
- QuickSwaps created before the Explorer launched are listed but never match
  a source or destination filter.

## How clients use it

ConnectRPC, no Authorization header:

```bash
curl -sS https://<api-host>/swap.quickswap.v1.QuickSwapExplorerService/ListCompletedQuickSwaps \
  -H 'Content-Type: application/json' \
  -d '{"sourceZippedAssetId": 3, "limit": 20}'
```

REST:

```bash
curl -sS 'https://<api-host>/v1/quickswap/explorer/quickswaps?sourceZippedAssetId=3&limit=20'
```

- `limit`: 0 means 50, at most 100; more is INVALID_ARGUMENT.
- Paging: pass `next_page_token` back as `page_token` with the same filters;
  empty `next_page_token` ends the list. The token is opaque and bound to the
  filters; with other filters it is `PAGE_TOKEN_INVALID`.
- Rate limited per client address: over the limit, RESOURCE_EXHAUSTED (HTTP
  429) with a Retry-After header.

## Migration

None: the service and the error code are new.
