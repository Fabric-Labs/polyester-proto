---
services:
  - marketoverview.v1.MarketOverviewService
kind: added
breaking: false
wire_compatible: true
---

## What and why

Market overview rows gain rolling window statistics that ticker clients and
market-analysis tools would otherwise rebuild from candles:

- `stats_1h`, `stats_4h` and `stats_7d` (`RollingWindowStats`): the open,
  high, low and change in basis points of each rolling window;
- `open_24h_ticks`: the 24h open that `change_24h_bps` is measured from;
- `trade_count_24h`: trades executed in the rolling 24h window;
- `vwap_24h_ticks`: the 24h volume-weighted average price;
- `open_utc_day_ticks`: the price at the start of the current UTC day.

Every window ends now and starts at the start of the minute that contains the
time one window length ago, exactly like the existing rolling 24h stats. All
prices come from executed trades, never from the index price.

Behaviour change: `high_24h_ticks` and `low_24h_ticks` now cover exactly the
same minutes as the 24h open, the 24h volumes and the new trade count, so the
window's first minute always counts toward the 24h high and low. Values can
differ slightly from earlier responses at the window's edge.

## How clients use it

The fields appear on every `MarketOverview` row of `ListMarketOverview` and of
the market overview stream; no request parameter changes.

```text
POST /marketoverview.v1.MarketOverviewService/ListMarketOverview
{"symbolId": [1]}

GET /v1/spot/markets/overview?symbolId=1
```

ConnectRPC row (prices in quote units scaled by 1e9):

```json
{
  "symbolId": 1,
  "lastPriceTicks": "61500000000000",
  "change24hBps": 250,
  "open24hTicks": "60000000000000",
  "tradeCount24h": "1834",
  "vwap24hTicks": "60812345678901",
  "openUtcDayTicks": "60750000000000",
  "stats1h": {"openTicks": "61200000000000", "highTicks": "61700000000000", "lowTicks": "61100000000000", "changeBps": 49},
  "stats4h": {"openTicks": "60500000000000", "highTicks": "61700000000000", "lowTicks": "60300000000000", "changeBps": 165},
  "stats7d": {"openTicks": "58000000000000", "highTicks": "62000000000000", "lowTicks": "57000000000000", "changeBps": 603}
}
```

REST renders the prices as decimal strings in quote units: `open24h`,
`vwap24h`, `openUtcDay`, and `stats1h`, `stats4h` and `stats7d` with `open`,
`high`, `low` and `changeBps`. `tradeCount24h` is an unsigned 64-bit decimal
string.

- A `stats_*` window is absent while it has no open: the market has no trade
  yet, its first trade is a few seconds old, or the window's trade history is
  temporarily unavailable. `open_24h_ticks` is 0 whenever the 24h window has
  no open. A new market's windows open at its first trade.
- `open_utc_day_ticks` is 0 while the market has no trade; just after
  00:00 UTC it can be 0 for a few seconds while the new day's first price is
  recorded.
- `vwap_24h_ticks` is the exact 24h quote notional divided by the 24h base
  volume, rounded down, and is omitted when the window has no base volume.
  The notional is not rounded to the quote scale, so dividing the displayed
  `volume_24h_quote_scaled` by `volume_24h_base_scaled` can differ slightly;
  the VWAP cannot be reproduced from the displayed volumes. The base volume is
  exact when the base asset's `market_data_volume_scale` equals its
  `quantity_scale` in `GetSpotConfig`; otherwise each second with trades
  rounds its base volume by at most half a volume unit (0.05 DOGE today), so
  the price error shrinks as the 24h base volume grows.

## Migration

None: every field is new and optional to read. Clients that compared
`high_24h_ticks` or `low_24h_ticks` against their own candles should expect the
24h window's first minute to be included.
