---
services:
  - swap.quickswap.v1.QuickSwapExplorerService
kind: added
breaking: false
wire_compatible: true
---

## What and why

`QuickSwapExplorerService` gains `GetQuickSwapStats`: public, read-only
statistics of completed QuickSwaps of every broker, for an analytics page.
It needs no credentials.

- The request names one preset window (`QuickSwapStatsWindow`):
  `LAST_24_HOURS` (hourly buckets), `LAST_7_DAYS`, `LAST_30_DAYS`,
  `LAST_90_DAYS` (daily) or `LAST_26_WEEKS` (weekly, 7-day buckets). Each
  ends at the end of the current bucket, so the last bucket is still
  filling.
- The answer has the window's `totals` and the `previous_totals` of the
  same-length window before it, every bucket oldest first (empty ones
  included), and every route's totals (`routes`, source then destination
  order), plus `as_of`. Each window's outcome, answer or error, is shared
  for about a minute.
- Totals are a swap count, deposited amounts per source zipped asset and
  delivered amounts per destination zipped asset (`ZippedAssetVolume`, exact
  token units), and fees per ledger asset (`AssetVolume`, 18 decimals): the
  service, broker, affiliate and referral fees charged, network fees not
  included. No USD values: the client prices the assets.
- New messages and enums: `QuickSwapStatsWindow`, `QuickSwapStatsBucketSize`,
  `ZippedAssetVolume`, `QuickSwapStatsTotals`, `QuickSwapStatsBucket`,
  `QuickSwapRouteStats`, `GetQuickSwapStatsRequest`,
  `GetQuickSwapStatsResponse`. `STATS_INCOMPLETE` now also covers this RPC.

## How clients use it

ConnectRPC, no Authorization header:

```bash
curl -sS https://<api-host>/swap.quickswap.v1.QuickSwapExplorerService/GetQuickSwapStats \
  -H 'Content-Type: application/json' -d '{"window": "LAST_7_DAYS"}'
```

REST:

```bash
curl -sS 'https://<api-host>/v1/quickswap/explorer/stats?window=LAST_7_DAYS'
```

- A missing or unknown window is INVALID_ARGUMENT (HTTP 400).
- `STATS_INCOMPLETE` (INTERNAL, HTTP 500) names a completed QuickSwap whose
  record is incomplete. Errors are shared like answers: a retry within about
  a minute gets the same error.
- Rate limited per client address: RESOURCE_EXHAUSTED (HTTP 429) with a
  Retry-After header.

## Migration

None: the RPC, its messages and enums are new.
