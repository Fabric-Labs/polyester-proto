---
services:
  - chain.analytics.v1.ChainAnalyticsService
kind: changed
breaking: true
wire_compatible: true
---

## What and why

Chain analytics REST errors now answer uppercase codes. Some answered
lowercase names, for example `invalid_request` for a supply chart without
`zippedAssetId`. The statuses, titles, details and `type` URIs are
unchanged.

| Error | Before | Now |
| --- | --- | --- |
| A malformed request, such as a missing or invalid query parameter | `400` `invalid_request` | `400` `INVALID_REQUEST` |
| A failure that is not a service error | `503` `upstream_error` | `503` `UPSTREAM_ERROR` |
| An asset whose scale is not known | `500` `internal_error` | `500` `INTERNAL_ERROR` |
| Any other refusal, such as a temporary failure | its status name in lowercase, such as `503` `unavailable` | the same name in uppercase, such as `UNAVAILABLE` |

## How clients use it

Branch on the uppercase `code`:

```text
GET /v1/chain/analytics/zipped-asset-supply

400 application/problem+json
{"type":"https://polyester.dev/problems/invalid-request","title":"Bad Request",
 "status":400,"detail":"zippedAssetId is required","code":"INVALID_REQUEST"}
```

ConnectRPC is unchanged.

## Migration

A client that matched a lowercase `code` on `/v1/chain/analytics/*` must match
its uppercase form instead, for example `INVALID_REQUEST` for
`invalid_request`.
