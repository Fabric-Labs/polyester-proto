---
services:
  - marketoverview.v1.MarketOverviewService
kind: added
breaking: false
wire_compatible: true
---

## What and why

`MarketOverview` gains `price_band = 18`, a `marketdata.v1.LivePriceBand`:
the pair's live price band, as `GetPriceBand` returns it, on every market
overview row, so a market list shows each pair's band without one call per
pair. It is present only while a band was confirmed current within the last 5
seconds. An absent `price_band` means no fresh band is known, not that the
pair has no band.

## How clients use it

`ListMarketOverview` (ConnectRPC) and `GET /v1/spot/markets/overview` (REST,
`priceBand` on each row) carry it, and so does the market overview stream
`public:spot:market_overview:updates:proto`.

```json
{"markets": [{"symbol": "BTC-USDT", "lastPrice": "61000.25",
  "priceBand": {"symbol": "BTC-USDT", "state": "ACTIVE", "referencePrice": "61000.25",
    "referenceSource": "LAST_TRADE", "maxBuyPrice": "64050.25", "minSellPrice": "57950.25",
    "updatedAt": "2026-11-19T09:58:12Z", "reanchorPending": false,
    "epoch": "3", "updateId": "41", "asOf": "2026-11-19T10:00:00.5Z", "validUntil": "2026-11-19T10:00:05.5Z"}}]}
```

- The copy is a convenience. The overview stream sends a row again when its
  band changes, appears or lapses, not on each confirmation, so a band whose
  `validUntil` has passed is unknown.
- To check an order price, read `GetPriceBand` or the price band stream
  `public:spot:market:price_band:{symbolId}`.

## Migration

None: the field is new and optional.
