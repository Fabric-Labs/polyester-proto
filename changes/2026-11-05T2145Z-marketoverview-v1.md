---
services:
  - marketoverview.v1.MarketOverviewService
kind: changed
breaking: false
wire_compatible: true
---

## What and why

`GET /v1/spot/markets/overview` now honours the `symbolId` query filter that
the reference documents. Before, the REST route ignored `symbolId` and
answered every market, also for a malformed value, so a client reading the
first row could take another market's prices for its own. The `symbols` filter
is unchanged, and the ConnectRPC method `ListMarketOverview` is unchanged.

The documentation of `ListMarketOverviewRequest.symbol_id` is corrected to
match the existing behaviour: with no filter, the list returns every
configured market, including disabled ones, and never a market whose delisting
time has passed. It previously said only enabled markets were returned. No
response changes.

## How clients use it

REST, `GET /v1/spot/markets/overview`: repeat `symbolId` for several pairs; it
combines with `symbols`. ConnectRPC takes the same pair IDs in `symbolId`:

```bash
curl -sS 'https://<api-host>/v1/spot/markets/overview?symbolId=1&symbolId=23'
curl -sS 'https://<api-host>/v1/spot/markets/overview?symbols=BTC-USDT&symbolId=23'
curl -sS 'https://<api-host>/marketoverview.v1.MarketOverviewService/ListMarketOverview' \
  -H 'Content-Type: application/json' -d '{"symbolId": [1, 23], "limit": 50}'
```

- `limit` defaults to 50; the maximum is 2000. Rows are ordered by descending
  24h USD volume unless `orderBy` and `sort` say otherwise.
- Paging: pass the opaque `nextPageToken` back as `pageToken` until it comes
  back empty. The token is bound to the selected pair IDs (on REST, after
  unknown and disabled ones are dropped), `orderBy` and `sort`, so keep the
  filter and the order; `limit` may change between pages.
- A value that is not a positive integer (`abc`, empty, `0`, `-1`, `1.5`, or
  above 4294967295) answers `400` problem+json with `code: BAD_REQUEST`.
- An unknown or disabled pair ID selects no market, exactly as an unknown or
  disabled name in `symbols` does: when no requested pair is selectable the
  answer is `200` with an empty `markets` list; otherwise only the selectable
  pairs are returned.
- A pair named more than once, through `symbols`, `symbolId` or both,
  selects its market once. ConnectRPC still refuses a repeated `symbolId` with
  `InvalidArgument`.

## Migration

None for clients that filter with `symbols` or call ConnectRPC. A REST client
that already sent `symbolId` now receives only the requested markets instead
of every market, and a malformed `symbolId` is now refused instead of ignored.
A REST request that repeats a pair, which answered `400` with
`code: INVALID_ARGUMENT`, now succeeds and returns that market once.
