---
services:
  - swap.quickswap.v1.QuickSwapExplorerService
kind: changed
breaking: true
wire_compatible: false
---

## What and why

`ListCompletedQuickSwaps` filters on the server what the Explorer offers, so every page holds up to
`limit` matching QuickSwaps and paging stays complete:

- several assets and networks per side;
- an asset on all its networks;
- a range of source USD value.

`ListCompletedQuickSwapsRequest` changes:

- `source_zipped_asset_id` (1) and `destination_zipped_asset_id` (2) become the repeated
  `source_zipped_asset_ids` and `destination_zipped_asset_ids`, at most 20 each, unique, non-zero.
- New `source_asset_ids` (5) and `destination_asset_ids` (6): `AssetConfig.ledger_id` values from
  `GET /v1/chain/deposit-withdraw/config`, each meaning that asset on every network. At most 20 each.
- New `min_source_value_usd` (7, inclusive) and `max_source_value_usd` (8, exclusive): strings, bounds
  on `ExplorerQuickSwap.source_value_usd` in its base units of 18 decimals (as
  `source_value_usd.base_units`); unset (empty) for no bound. Each bound is 0, or 1, 2 or 5 times a power of ten
  dollars from $1 to $1,000,000,000: $1, $2, $5, $10, $20, $50, $100 and so on. In base units that is
  `0`, `1`, `2` or `5` followed by 18 to 26 zeros, or `1` followed by 27 zeros; `100000000000000000000`
  is $100.

A side matches a QuickSwap whose asset and network is any of its zipped asset IDs, or on any network
of any of its asset IDs. The two sides and the USD range must all match. A QuickSwap without a
`source_value_usd` never matches a USD bound.

With a USD bound, the list holds only QuickSwaps that completed at least 5 minutes before it is
read, once their `source_value_usd` is final. So USD-filtered results lag completion by up to 5
minutes, and their pages never gain or lose a QuickSwap while you read them. Lists without a USD
bound show completions at once.

The filters together may select at most 400 routes (source and destination asset and network
pairs), counting each asset ID as all its networks. With a USD bound, they may select at most 400
routes times the value steps the bounds cover. The steps run $0 to $1, $1 to $2, $2 to $5, $5 to $10
and so on; the last holds every value from $1,000,000,000 up. A range without a maximum covers the
steps from its minimum to the last: from $1 that is 28 steps (at most 14 routes), from $500 20 steps
(20 routes). The count depends on the request alone, so every page of an accepted request is
accepted.

## How clients use it

ConnectRPC, USDT on any network to BTC or ETH on Ethereum, worth $1,000 to $10,000:

```http
POST /swap.quickswap.v1.QuickSwapExplorerService/ListCompletedQuickSwaps
Content-Type: application/json

{"sourceAssetIds": [2], "destinationZippedAssetIds": [7, 3],
 "minSourceValueUsd": "1000000000000000000000",
 "maxSourceValueUsd": "10000000000000000000000",
 "limit": 50}
```

REST repeats each list parameter, and takes the USD bounds as the same base-unit strings:

```http
GET /v1/quickswap/explorer/quickswaps?sourceAssetId=2&destinationZippedAssetId=7&destinationZippedAssetId=3&minSourceValueUsd=1000000000000000000000&maxSourceValueUsd=10000000000000000000000
```

Paging is unchanged: newest completion first, `next_page_token` bound to the exact filters. Reuse it
only with the same filters, in any order of the IDs; anything else is `INVALID_ARGUMENT` with
`PAGE_TOKEN_INVALID`.

The token is also bound to the networks the asset IDs covered on the first page. If a network is
added to or removed from one of those assets before a later page, that page is `PAGE_TOKEN_INVALID`
with the message "the asset catalog changed since the first page; request the first page again".
Such changes are rare.

The following are `INVALID_ARGUMENT`:

- a USD bound off the grid, for example $150 or $1,000.01;
- filters selecting more than 400 routes, or more than 400 routes times value steps;
- a `max_source_value_usd` not above `min_source_value_usd`, so a lone maximum of `0` too.

For no USD bound, omit the REST parameter. Over ConnectRPC JSON, leave the field out or send it empty:
the empty string is the field's default and means unset.

An unknown asset ID matches nothing.

## Migration

1. Over ConnectRPC JSON, send `sourceZippedAssetIds: [id]` and `destinationZippedAssetIds: [id]` (arrays)
   instead of `sourceZippedAssetId` and `destinationZippedAssetId`. Protobuf binary clients keep
   working: field numbers 1 and 2 are unchanged.
2. REST keeps the parameter names `sourceZippedAssetId` and `destinationZippedAssetId`, now
   repeatable; a single value works as before. `0` no longer means "every route": it is
   `INVALID_ARGUMENT`. Omit the parameter (or, over ConnectRPC, send an empty list) for every route.
3. Page tokens issued before this change are `PAGE_TOKEN_INVALID`: start again from the first page.
4. Replace client-side filtering by asset across networks, several assets, or USD value with the new
   fields.
