---
services:
  - chain.deposit.v1.DepositAddressService
  - chain.withdraw.v1.WithdrawService
  - chain.guard.v1.GuardSignerService
  - swap.v1.SwapService
  - swap.quickswap.v1.QuickSwapService
  - transfer.v1.InternalTransferService
  - auth.v1.AuthService
  - auth.v1.ApiKeyService
  - auth.v1.SubaccountService
  - auth.v1.SubaccountViewService
  - auth.v1.PolicyService
  - auth.v1.AddressBookService
  - fees.v1.FeeService
  - checkpoints.v1.CheckpointService
kind: changed
breaking: true
wire_compatible: true
---

## What and why

Oversized API-key requests for deposit addresses, withdrawals, Guard Signer,
Swap, account reads and internal transfers now return HTTP 413 with
`Content-Type: application/problem+json` instead of plain text. Oversized
Whiteboards REST requests return the same problem. The problem code is
`REQUEST_TOO_LARGE` and its type is
`https://polyester.dev/problems/request-too-large`.

REST body decoding for deposit addresses, withdrawals, internal transfers and
QuickSwap also uses `REQUEST_TOO_LARGE`, replacing the lowercase
`request_too_large` or `INVALID_ARGUMENT` codes for HTTP 413. Request limits and
when they are enforced do not change. No protobuf message or RPC changes.

## How clients use it

For example, an API-key request to `POST https://<api-host>/v1/chain/deposit-addresses`
with more than 1 KiB of body data returns:

```http
HTTP/1.1 413 Request Entity Too Large
Content-Type: application/problem+json

{"type":"https://polyester.dev/problems/request-too-large","title":"Request Entity Too Large","status":413,"detail":"request too large","code":"REQUEST_TOO_LARGE"}
```

The size refusal still precedes API-key verification, including for unknown
keys and chunked bodies. The deposit-address and Guard Signer API-key limits
remain 1 KiB; withdrawals, Swap, account reads and internal transfers remain
64 KiB; Whiteboards REST remains 256 KiB. QuickSwap REST retains its 64 KiB
limit. A smaller body must still satisfy the endpoint's normal validation and
authentication requirements.

Guard Signer API-key callers use the existing ConnectRPC endpoint:

```http
POST /chain.guard.v1.GuardSignerService/GetGuardSignerStatus
Content-Type: application/json

{"subaccount_id":"1101"}
```

Its authentication-time oversized-body refusal uses the problem described
above; decoded ConnectRPC message-limit errors retain their existing protocol
encoding.

## Migration

Clients that parse plain-text HTTP 413 responses must parse the JSON problem
and branch on `code: REQUEST_TOO_LARGE`. Replace checks for the lowercase
`request_too_large` or the QuickSwap HTTP 413 `INVALID_ARGUMENT` code with
`REQUEST_TOO_LARGE`. Keep status-based HTTP 413 handling and reduce the body to
fit the existing limit before sending another request. Protobuf encoding and
generated request and response types need no changes.
