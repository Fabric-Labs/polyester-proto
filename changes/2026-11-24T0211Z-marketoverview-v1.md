---
services:
  - marketoverview.v1.MarketOverviewService
kind: added
breaking: false
wire_compatible: true
---

## What and why

`ListMarketOverview` gains server-side range filters, so a screener can ask
for the markets it shows instead of fetching every row:

- `last_price` (`PriceTicksRange`): `last_price_ticks` between `min_ticks` and
  `max_ticks`, in quote units scaled by 1e9;
- `change_24h` (`BpsRange`): the signed `change_24h_bps` between `min_bps` and
  `max_bps`;
- `volume_24h` (`UsdVolumeRange`): `volume_24h_usd_scaled` between
  `min_usd_scaled` and `max_usd_scaled`, scaled by 1e6.

Every bound is optional and inclusive; an omitted bound leaves that side
unbounded. The filters combine with each other and with `symbol_id`: a market
is returned only when it matches every filter given.

## How clients use it

ConnectRPC: markets between $1M and $50M of 24h USD volume that moved between
-5% and +10%:

```text
POST /marketoverview.v1.MarketOverviewService/ListMarketOverview
{
  "volume24h": {"minUsdScaled": "1000000000000", "maxUsdScaled": "50000000000000"},
  "change24h": {"minBps": -500, "maxBps": 1000},
  "orderBy": "ORDER_BY_VOLUME_24H_USD",
  "limit": 50
}
```

REST takes the bounds as query parameters in the units the rows use:

```text
GET /v1/spot/markets/overview?minVolume24hUsd=1000000&maxVolume24hUsd=50000000&minChange24hBps=-500&maxChange24hBps=1000
```

- REST: `minLastPrice` and `maxLastPrice` are decimal strings in quote units
  with at most 9 decimal places, like `lastPrice`; `minVolume24hUsd` and
  `maxVolume24hUsd` are decimal strings in USD with at most 6 decimal places,
  like `volume24hUsd`; `minChange24hBps` and `maxChange24hBps` are integers.
  A decimal has digits on both sides of any dot and no sign (`0.5`, not
  `.5`, `1.` or `-1`). Leave a parameter out for no bound, and give each at
  most once: a present value REST cannot read, an empty one included, and a
  repeated parameter are refused `400` `BAD_REQUEST`, never taken as zero.
- Once a range sets a bound, a market without that value never matches: one
  with no trade yet for a price bound, one whose 24h change has no open
  (`open_24h_ticks` is 0) for a change bound, and one whose
  `volume_24h_usd_scaled` is omitted for a volume bound.
- The change filter is signed; sorting by `ORDER_BY_CHANGE_24H_BPS` still
  uses the absolute change.
- Errors, all `INVALID_ARGUMENT` (REST `400` `INVALID_ARGUMENT`), with a
  field violation whose path is relative to the request:
  - a range whose minimum exceeds its maximum: the path names the range,
    `last_price`, `change_24h` or `volume_24h`, with rule ID
    `price_ticks_range.ordered`, `bps_range.ordered` or
    `usd_volume_range.ordered`;
  - a negative price or volume bound (ConnectRPC only, since REST decimals
    carry no sign): the path names the bound itself, such as
    `last_price.max_ticks` or `volume_24h.min_usd_scaled`, with rule ID
    `int64.gte`.
- The filters apply before sorting and paging. `page_token` is bound to the
  symbol IDs, the range filters and the sort: reuse it with exactly the same
  ones. A token reused with anything else is refused `INVALID_ARGUMENT`;
  restart paging from the first page without a token.
- Pages are cut from current values. A market whose sort value or range
  membership changes between two pages can appear on both or on neither, as
  it already could without filters. For a consistent screen, request one
  page with the maximum `limit` (2000).

## Migration

None for requests: the filters are new and optional, and a request without
them returns what it returned before. A `page_token` issued before this
change is refused `INVALID_ARGUMENT` on its next use: a client paging across
the upgrade restarts from the first page without a token.
