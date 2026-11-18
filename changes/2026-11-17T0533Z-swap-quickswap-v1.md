---
services:
  - swap.quickswap.v1.QuickSwapBrokerService
kind: changed
breaking: false
wire_compatible: true
---

## What and why

The REST reference now marks `startTime`, `endTime` and `groupBy` as required
query parameters of `GET /v1/quickswap/brokers/{broker}/fee-totals` and
`GET /v1/quickswap/brokers/{broker}/stats`. The server always required them:
a request without them answered `400` `INVALID_ARGUMENT`, while the reference
listed them as optional and its example sent no window. The reference's example now sends a valid window, with `endTime`
strictly after `startTime`.

The server's behavior and the ConnectRPC contract are unchanged.

## How clients use it

Send a window of 1 nanosecond to 366 days, `endTime` exclusive, as RFC 3339
times, with the grouping:

```text
GET /v1/quickswap/brokers/{broker}/fee-totals?groupBy=BY_VAULT&startTime=2025-01-01T00:00:00Z&endTime=2025-01-02T00:00:00Z
GET /v1/quickswap/brokers/{broker}/stats?groupBy=DAY&startTime=2025-01-01T00:00:00Z&endTime=2025-01-02T00:00:00Z
```

Over ConnectRPC, `/swap.quickswap.v1.QuickSwapBrokerService/GetFeeTotals` and
`/swap.quickswap.v1.QuickSwapBrokerService/GetBrokerStats` take the same
fields as `startTime`, `endTime` and `groupBy` in the JSON body.

## Migration

None: a request without these parameters was already refused, so every
working client sends them.
