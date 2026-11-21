---
services:
  - swap.quickswap.v1.QuickSwapBrokerService
kind: changed
breaking: true
wire_compatible: false
---

## What and why

`QuickSwapBrokerService.GetBrokerStats` now answers in columns, like the
equity history: one header with implicit bucket timestamps, an asset
dictionary, and series of aligned columns. The answer is several times
smaller, and the same read now serves hourly charts, a breakdown by route,
vault, source asset or destination asset, and a drill-down into one group.

- `StatsPeriod` gains `HOUR`. Windows are at most 7 days for `HOUR`, 366 days
  for `DAY` and 3660 days for `MONTH`, so one request covers a broker's whole
  history in months.
- `GetBrokerStatsRequest` gains `breakdown` (`StatsBreakdown`: `ROUTE`,
  `VAULT`, `SOURCE_ASSET`, `DESTINATION_ASSET`), the `drill_down` oneof
  (`route`, `vault_account_id`, `source_zipped_asset_id`,
  `destination_zipped_asset_id`, matching the breakdown) and `include_cells`.
- `GetBrokerStatsResponse` is replaced: `period`, `start_ts_sec`, `points`,
  `as_of_ts_sec`, `asset_ids` and `repeated BrokerStatsSeries series`.
- Each `BrokerStatsSeries` has a `grouping` oneof (`totals`, `route`,
  `vault_account_id`, `source_zipped_asset_id`, `destination_zipped_asset_id`,
  `others`) and window totals: `created_count`, `status_counts` indexed by
  `QuickSwapStatus` number, exact `deposited`, `delivered` and `fees` amounts
  (`polyester.type.v1.U128`, 18 decimals, parallel to `asset_ids`), and USD
  values as `uint64` scaled by 1e4.
- Every group with activity in the window gets its window totals, with no
  limit. Per-bucket columns (`created`, `completed`, `deposited_usd`,
  `delivered_usd`, `fees_usd`, `unvalued`) go to the totals series, the 10
  groups with the most deposited USD, the drill-down group and `others`.
- The totals series also carries the fee split per bucket in USD
  (`service_fee_usd`, `broker_fee_usd`, `affiliate_fee_usd`,
  `referral_fee_usd`), and, with `include_cells`, the exact amounts per bucket
  and asset as sparse cells.
- Removed: `BrokerStatsPeriod`, `StatusCount` and the `periods` field.
  `GetBrokerStats` no longer returns `STATS_INCOMPLETE`; a temporary failure
  is `UNAVAILABLE`.

## How clients use it

ConnectRPC, 7 days of hourly buckets broken down by route:

```http
POST /swap.quickswap.v1.QuickSwapBrokerService/GetBrokerStats
{"broker": "acme", "groupBy": "HOUR", "startTime": "2026-11-12T00:00:00Z",
 "endTime": "2026-11-19T00:00:00Z", "breakdown": "ROUTE"}
```

REST: `GET /v1/quickswap/brokers/acme/stats?groupBy=HOUR&startTime=...&endTime=...&breakdown=ROUTE`.
A drill-down adds `routeSourceZippedAssetId` with
`routeDestinationZippedAssetId`, or `vaultAccountId`, `sourceZippedAssetId`
or `destinationZippedAssetId`; `includeCells=true` adds the cells.

- Bucket `i` starts `i` hours or days after `start_ts_sec`, or `i` calendar
  months after it for `MONTH`; every column is `points` long.
- `series[0]` is always the totals. Groups follow by deposited USD from
  highest, compared exactly before rounding to `deposited_usd_q4`, and at
  equal value by the identifier in `grouping`, ascending (a vault by its
  public `vaultAccountId`, a route by source, then destination). A drill-down group without activity comes next, at zero, and
  never takes one of the 10 column slots; `others` is last when some groups
  have no columns.
- A QuickSwap counts in the bucket it was created in, by current status. On
  the totals series a fee counts in the bucket it was charged in, as in
  `GetFeeTotals`; on a group series the fees charged to its QuickSwaps count
  in the QuickSwap's creation bucket.
- Groups by vault overlap: a QuickSwap that credited several vaults counts in
  each.

## Migration

1. Replace reads of `periods[]` with the columns: totals from `series[0]`,
   timestamps from `start_ts_sec`, `points` and `period`.
2. Read amounts through `asset_ids`: `deposited[i]` is the amount of
   `asset_ids[i]`, a 128-bit integer of 18 decimals.
3. Read USD values as integers scaled by 1e4 instead of `TokenAmount`s of 18
   decimals.
4. Read status counts by index: `status_counts[n]` counts status number `n`.
5. Per-asset amounts per bucket, which `periods[].deposited` gave, now need
   `include_cells`.
6. Treat `UNAVAILABLE` as retryable; `STATS_INCOMPLETE` no longer comes from
   this RPC.
