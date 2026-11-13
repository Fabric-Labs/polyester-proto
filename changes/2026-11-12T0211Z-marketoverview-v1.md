---
services:
  - marketoverview.v1.MarketOverviewService
  - orderbook.v1.OrderbookService
kind: changed
breaking: true
wire_compatible: true
---

## What and why

`GET /v1/spot/markets/overview` now filters the way
`MarketOverviewService.ListMarketOverview` does, so both transports answer the
same request the same way:

- `symbols` and `symbolId` select every configured market, disabled ones
  included. Before, they returned nothing for a disabled market, which the
  unfiltered list still showed.
- An unknown symbol or pair ID is refused with `400` and code
  `INVALID_ARGUMENT`. Before, REST answered `200` with no markets. A
  malformed `symbolId` stays `400 BAD_REQUEST`. Markets whose delisting time
  has passed are still never returned.

The order-book depth stream's description now states the rule for
following it (no behaviour change): each update is the difference between two
published views of the tier's top levels; a full `reset` view arrives
periodically, currently about every 30 seconds, and after a gap; a zero
quantity removes the level from the tier's view.

## How clients use it

```http
GET /v1/spot/markets/overview?symbols=BTC-USDT&symbolId=7
```

```json
POST /marketoverview.v1.MarketOverviewService/ListMarketOverview
{ "symbolId": [1, 7] }
```

Depth stream: subscribe, then fetch `GET /v1/orderbook/{symbol}` (or
`OrderbookService.GetOrderBook`) with the channel's depth. Drop updates whose
`bookSeqEnd` is at most the snapshot's `bookSeq`; apply the first update only
when its `bookSeqStart` equals that `bookSeq`, then each next one only when its
`bookSeqStart` equals the previous `bookSeqEnd`. Otherwise wait for the next
`reset`.

## Migration

Breaking for REST overview callers that send unknown symbols or IDs: they now
get `400 INVALID_ARGUMENT` instead of an empty list. Send only symbols and IDs
from the spot configuration, or handle `INVALID_ARGUMENT`. Callers that
filter by a disabled market now receive its row; read the pair's status from
the spot configuration. ConnectRPC callers see no change.
