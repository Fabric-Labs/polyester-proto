---
services:
  - marketoverview.v1.MarketOverviewService
kind: changed
breaking: false
wire_compatible: true
---

## What and why

Documentation only: the market overview row now states precisely when its
opens are unknown. `open_24h_ticks` and `change_24h_bps` are 0,
`open_utc_day_ticks` is 0, and the `stats_1h`, `stats_4h` and `stats_7d`
messages are absent while the open is unavailable. That includes when the
market's trade history is temporarily unavailable, not only before a
market's first trade. Behaviour and wire format are unchanged.

## How clients use it

No call changes. Read the same fields of `ListMarketOverview` and the market
overview stream:

```text
POST /marketoverview.v1.MarketOverviewService/ListMarketOverview
{"symbolId": [1]}

GET /v1/spot/markets/overview?symbolId=1
```

Treat `open_24h_ticks == 0` or `open_utc_day_ticks == 0` as "open unknown",
and `change_24h_bps` as meaningful only while `open_24h_ticks` is non-zero.
An absent `stats_*` message means that window's open is unknown.

## Migration

None: the behaviour is unchanged; only its description is more precise.
Clients that treated `change_24h_bps == 0` as "no change" should check
`open_24h_ticks` first.
