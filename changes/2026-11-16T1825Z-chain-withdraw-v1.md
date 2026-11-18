---
services:
  - chain.withdraw.v1.WithdrawService
  - chain.withdraw.v1.AutoWithdrawService
kind: changed
breaking: true
wire_compatible: true
---

## What and why

Withdraw and auto-withdraw REST refusals now answer the uppercase codes of
the withdraw error catalog, the codes ConnectRPC already carries in
`chain.withdraw.v1.ErrorDetail`. Some answered lowercase names that are not
catalog codes. The title and `type` follow the code.

| Error | Before | Now |
| --- | --- | --- |
| A malformed request, such as a body that is not JSON or a refused field | `400` `invalid_request`, title `Bad Request` | `400` `INVALID_REQUEST`, title `Invalid Request`; `type` unchanged |
| A signed API-key request whose body the authentication layer cannot read, over REST or ConnectRPC (any `WithdrawService` or `AutoWithdrawService` method) | `503` `upstream_error`, `type` `.../upstream-error` | `503` `SERVICE_UNAVAILABLE`, title `Service Unavailable`, `type` `.../service-unavailable` |
| An unexpected server error writing the response | `500` `internal_error`, title `Internal Server Error` | `500` `INTERNAL_ERROR`, title `Internal Error`; `type` unchanged |

## How clients use it

Branch on the uppercase `code`:

```text
POST /v1/chain/trading-withdraws

400 application/problem+json
{"type":"https://polyester.dev/problems/invalid-request","title":"Invalid Request",
 "status":400,"detail":"invalid request body","code":"INVALID_REQUEST"}
```

Over ConnectRPC only the authentication refusal above changes: an API-key call
whose body cannot be read answers `503` with the problem `code`
`SERVICE_UNAVAILABLE` and `type`
`https://polyester.dev/problems/service-unavailable`. Every other ConnectRPC
error is unchanged.

## Migration

A client that matched `invalid_request`, `upstream_error` or
`internal_error` on the withdraw or auto-withdraw REST routes, or the
authentication refusal over ConnectRPC, must match
`INVALID_REQUEST`, `SERVICE_UNAVAILABLE` or `INTERNAL_ERROR` instead. A client
that matched the `type` `https://polyester.dev/problems/upstream-error` must
match `https://polyester.dev/problems/service-unavailable`.
