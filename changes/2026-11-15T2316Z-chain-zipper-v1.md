---
services:
  - chain.zipper.v1.ZipperService
kind: changed
breaking: true
wire_compatible: true
---

## What and why

A route's zToken supply now says whether it is known and current, and when it
was read from Polyester chain. Before, a route with no known supply reported a
supply of zero, so a client could not tell "supply unavailable" from "supply
really is 0" when sizing a withdrawal.

- New enum `SupplyStatus`: `AVAILABLE` (current), `STALE` (not refreshed for
  longer than expected; may no longer be accurate), `MISSING` (no supply is
  known yet). `SUPPLY_UNSPECIFIED` is never sent.
- `AssetChainVariant` in `GetDepositWithdrawConfig` gains `supply_status`
  (field 15) and `supply_observed_at` (field 16, timestamp). `supply_q` and
  `supply_observed_at` are set only when the status is `AVAILABLE` or
  `STALE`; a `supply_q` of 0 then is a real supply of zero.
- `ZippedAssetSupplyUpdate` on the supply WebSocket stream gains `status`
  (field 3) and `observed_at` (field 4). Updates carry `AVAILABLE` or `STALE`.
  A route whose supply stops being refreshed gets one `STALE` update repeating
  its last value and observation time; its next reading turns it `AVAILABLE`
  again.
- REST `GET /v1/chain/deposit-withdraw/config`: each variant gains
  `supplyStatus` and `supplyObservedAt` (RFC 3339 UTC), and omits `supply`
  and `supplyObservedAt` while the status is `MISSING`. It no longer sends
  `"supply": "0"` for an unknown supply.
- JSON WebSocket channel `public:chain:zipped-asset:supply`: each update gains
  `status` and `observedAt` (RFC 3339 UTC).

## How clients use it

Read the initial state of every route from the config, then apply stream
updates by `zippedAssetId`.

ConnectRPC:

```http
POST /chain.zipper.v1.ZipperService/GetDepositWithdrawConfig
Content-Type: application/json

{}
```

```json
{
  "assets": [{
    "asset": "USDT",
    "quantityScale": 6,
    "variants": [
      {"zippedAssetId": 3, "supplyQ": "1250000500000", "supplyStatus": "AVAILABLE", "supplyObservedAt": "2026-11-15T12:00:00.500Z"},
      {"zippedAssetId": 5, "supplyStatus": "MISSING"}
    ]
  }]
}
```

REST `GET /v1/chain/deposit-withdraw/config` returns the same variants with
`supply` as a decimal string:

```json
{"zippedAssetId": 3, "supply": "1250000.5", "supplyStatus": "AVAILABLE", "supplyObservedAt": "2026-11-15T12:00:00.5Z"}
{"zippedAssetId": 5, "supplyStatus": "MISSING"}
```

WebSocket `public:chain:zipped-asset:supply`:

```json
{"updates": [{"zippedAssetId": 3, "supply": "1250000.5", "status": "STALE", "observedAt": "2026-11-15T12:00:00.5Z"}]}
```

Allow a withdrawal sized against a route's supply only while its status is
`AVAILABLE`. Treat `STALE` and `MISSING` as "liquidity cannot be confirmed
right now", not as zero liquidity.

## Migration

1. Read `supplyStatus` (stream: `status`) before the supply value.
2. REST clients: `supply` may now be absent. Do not default it to `"0"`; an
   absent `supply` means `MISSING`.
3. Generated SDK clients: the `supply_q` getter still returns 0 when nothing
   is known; check the status first.
