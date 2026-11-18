---
services:
  - chain.deposit.v1.DepositAddressService
kind: changed
breaking: true
wire_compatible: true
---

## What and why

Deposit address REST errors now answer uppercase codes. Some answered
lowercase names, for example `permission_denied` when an API key without the
right to read deposit addresses lists them. The statuses, titles, details and
`type` URIs are unchanged.

| Error | Before | Now |
| --- | --- | --- |
| A malformed REST request, such as a body that is not JSON or cannot be read, or an unsupported query parameter | `400` `invalid_request` | `400` `INVALID_REQUEST` |
| A signed API-key request whose body the authentication layer cannot read, over REST or ConnectRPC (any `DepositAddressService` method, such as `ListDepositAddresses`) | `503` `upstream_error` | `503` `UPSTREAM_ERROR` |
| An unexpected server error writing the response | `500` `internal_error` | `500` `INTERNAL_ERROR` |
| Any other refusal, such as a missing permission, an unknown subaccount or a temporary failure | its status name in lowercase, such as `403` `permission_denied`, `404` `not_found` or `503` `unavailable` | the same name in uppercase: `PERMISSION_DENIED`, `NOT_FOUND`, `UNAVAILABLE` |

## How clients use it

Branch on the uppercase `code`:

```text
GET /v1/chain/deposit-addresses

403 application/problem+json
{"type":"https://polyester.dev/problems/permission-denied","title":"Forbidden",
 "status":403,"detail":"API key policy does not grant READ_DEPOSIT_ADDRESSES",
 "code":"PERMISSION_DENIED"}
```

Over ConnectRPC only the authentication refusal above changes: an API-key call
whose body cannot be read answers `503` with the problem `code`
`UPSTREAM_ERROR`. Every other ConnectRPC error is unchanged.

## Migration

A client that matched a lowercase `code` on `/v1/chain/deposit-addresses`
must match its uppercase form instead, for example `PERMISSION_DENIED` for
`permission_denied`, and one that matched `upstream_error` on an API-key
ConnectRPC call must match `UPSTREAM_ERROR`.
