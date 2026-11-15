---
services:
  - chain.withdraw.v1.WithdrawService
  - chain.withdraw.v1.AutoWithdrawService
  - chain.deposit.v1.DepositAddressService
kind: changed
breaking: false
wire_compatible: true
---

## What and why

The withdraw and deposit REST endpoints now answer one more refusal as a
problem body. A request signed with an API key whose body cannot be read in
full, for example because the client disconnected mid-upload, is answered
`502 application/problem+json` with `code` `upstream_error` and `detail`
"request body could not be read". Before, it was `502 text/plain` with no
`code`. A body over the size limit was already answered
`413 application/problem+json` `REQUEST_TOO_LARGE`.

## How clients use it

Nothing changes for a request the server reads in full. The refusal looks like:

```http
POST /v1/chain/trading-withdraws

HTTP/1.1 502 Bad Gateway
Content-Type: application/problem+json

{"type":"https://polyester.dev/problems/upstream-error","title":"Bad Gateway","status":502,"detail":"request body could not be read","code":"upstream_error"}
```

## Migration

None: the status stays 502. A client that read this refusal as plain text can
now read `code` and `detail` from the problem body, like every other refusal
of these endpoints.
