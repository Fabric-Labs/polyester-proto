---
services:
  - marketoverview.v1.MarketOverviewService
kind: changed
breaking: false
wire_compatible: true
---

## What and why

`MarketOverview.change_24h_bps` now has a documented reference price, the
24h open, and a market younger than 24 hours no longer reports a 0 change.

- The 24h open has minute granularity, as on established venues: it is the
  price at the start of the minute that contains the time 24 hours ago.
- New: when the market first traded later than that (it was listed less than
  24 hours ago, or trading restarted after an environment reset), the 24h open
  is its first trade. Before, such a market always reported `0`, however far
  its price had moved, so sorting by 24h change ranked those markets
  arbitrarily.
- A market with no trade at all still reports `0`, and its last trade
  timestamp is `0` too. A market that has just started trading reports `0`
  until its first minute of trading has ended.

The field, its type and its encoding are unchanged. The same value appears in
`ListMarketOverview` and in the market overview stream.

## How clients use it

No new calls. Read the field as before:

```bash
curl -sS https://<api-host>/marketoverview.v1.MarketOverviewService/ListMarketOverview \
  -H 'Content-Type: application/json' \
  -d '{"orderBy": "ORDER_BY_CHANGE_24H_BPS", "sort": "SORT_DESC"}'
curl -sS 'https://<api-host>/v1/spot/markets/overview?orderBy=CHANGE_24H_BPS&sort=DESC'
```

`change24hBps` is an integer in basis points: `+123` is `+1.23%`, truncated
toward zero. Sorting by `CHANGE_24H_BPS` uses its absolute value.

## Migration

None: the field keeps its type and encoding. Clients that showed young markets
as unchanged now show their real move. To tell "no trade yet" from "no move",
check the last trade timestamp: it is `0` only when the market has no trade.
