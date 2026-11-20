---
services:
  - swap.quickswap.v1.QuickSwapExplorerService
kind: changed
breaking: true
wire_compatible: false
---

## What and why

`GetQuickSwapStats` answers in columns, so a chart reads it directly and the answer is several times
smaller: on 250 routes, the 24-hour answer shrinks from about 190 KB to 30 KB of JSON (18 KB to 6 KB
gzipped), and the 90-day one from about 540 KB to 38 KB.

- Assets are listed once: `zipped_asset_ids` with `zipped_asset_decimals`, and `fee_asset_ids` (fees
  are in base units of 18 decimals).
- `totals` and `previous_totals` carry per-asset amounts as lists parallel to those:
  `source_volume`, `destination_volume` (parallel to `zipped_asset_ids`), `fees`,
  `fees_usd_by_asset` and `fees_unvalued_count` (parallel to `fee_asset_ids`), with `"0"` or `0` for an
  asset with nothing in that period.
- `buckets` is one `QuickSwapStatsBuckets` message of parallel lists, entry `i` being bucket `i`,
  oldest first: `swap_count`, `unvalued_count`, `source_volume_usd`, `destination_volume_usd`,
  `fees_usd`, `fees_unvalued_count`. Bucket `i` starts at `start_time` plus `i` bucket lengths. Buckets
  carry no per-asset amounts.
- `routes` is one `QuickSwapStatsRoutes` message of parallel lists, in ascending source then
  destination order: `source_zipped_asset_ids`, `destination_zipped_asset_ids`, `swap_count`,
  `unvalued_count`, `source_volume`, `destination_volume`, `source_volume_usd`,
  `destination_volume_usd`.
- Counts are `uint32`: JSON numbers, not strings. Amounts are exact decimal strings of base units.
  USD values are decimal strings of base units of 18 decimals, `"0"` when nothing was valued; the
  unvalued counts tell an unvalued `"0"` from a real one.
- `ZippedAssetVolume`, `QuickSwapStatsBucket` and `QuickSwapRouteStats` are removed. Field numbers of
  `GetQuickSwapStatsResponse` and `QuickSwapStatsTotals` are reassigned.

## How clients use it

```text
GET https://<api-host>/v1/quickswap/explorer/stats?window=LAST_7_DAYS
POST /swap.quickswap.v1.QuickSwapExplorerService/GetQuickSwapStats
{"window": "LAST_7_DAYS"}
-> {"window": "LAST_7_DAYS", "startTime": "2026-11-12T00:00:00Z", "endTime": "2026-11-19T00:00:00Z",
    "bucketSize": "DAILY", "zippedAssetIds": [3, 58], "zippedAssetDecimals": [18, 6], "feeAssetIds": [4],
    "totals": {"swapCount": 3, "unvaluedCount": 1, "sourceVolumeUsd": "...", "destinationVolumeUsd": "...",
               "feesUsd": "...", "sourceVolume": ["...", "0"], "destinationVolume": ["0", "..."],
               "fees": ["..."], "feesUsdByAsset": ["..."], "feesUnvaluedCount": [0]},
    "previousTotals": {...},
    "buckets": {"swapCount": [1, 0, 0, 0, 0, 0, 2], "unvaluedCount": [...], "sourceVolumeUsd": [...],
                "destinationVolumeUsd": [...], "feesUsd": [...], "feesUnvaluedCount": [...]},
    "routes": {"sourceZippedAssetIds": [3], "destinationZippedAssetIds": [58], "swapCount": [3], ...},
    "asOf": "2026-11-18T16:40:00Z"}
```

The amount of asset `zippedAssetIds[i]` is `totals.sourceVolume[i]` in base units of
`zippedAssetDecimals[i]` decimals. A route's amounts use its own source and destination asset's
decimals from the same lists. As before, ProtoJSON omits a zero scalar, such as a period's
`swapCount` of 0.

`ListCompletedQuickSwaps` is unchanged; its page size is `limit` (REST query `limit`), 50 by default
and at most 100. There is no `pageSize` parameter.

## Migration

1. Read per-asset totals by index into `zippedAssetIds` / `feeAssetIds` instead of from
   `{zippedAssetId, amount}` and `{assetId, amount, valueUsd}` objects.
2. Read the time series from `buckets.<column>[i]`, with bucket starts computed from `startTime` and
   `bucketSize`, instead of from a list of bucket objects.
3. Read routes from `routes.<column>[i]` instead of a list of route objects; take each amount's
   decimals from `zippedAssetDecimals`.
4. Read counts as numbers, and USD values as decimal strings of 18-decimal base units where `"0"` with
   a non-zero unvalued count means no value, instead of an absent `TokenAmount`.
