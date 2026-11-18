---
services:
  - swap.quickswap.v1.QuickSwapService
  - swap.quickswap.v1.QuickSwapBrokerService
  - swap.quickswap.v1.QuickSwapReferralService
kind: added
breaking: false
wire_compatible: true
---

## What and why

The API reference now lists, for every QuickSwap method, the error codes it
can return. Each code shows its title, ConnectRPC code and HTTP status, next
to the generic transport errors. Until now the reference showed only the
generic errors for QuickSwap.

- New method option `polyester.api.error_codes` (common
  `polyester/api/options.proto`): the `ErrorCode` value names an RPC can
  return. It is documentation metadata and never appears in payloads.
- Every `QuickSwapService`, `QuickSwapBrokerService` and
  `QuickSwapReferralService` RPC declares its codes.
- Responses are unchanged: every `ErrorCode` keeps its ConnectRPC code and
  HTTP status. `QuoteQuickSwap` lists `SOURCE_WALLET_INVALID` and
  `SOURCE_WALLET_REJECTED` (PERMISSION_DENIED, HTTP 403); `CreateQuickSwap`
  lists both too.
- Besides the generic errors, every method lists `RESOURCE_EXHAUSTED` (429),
  `DEADLINE_EXCEEDED` and `INTERNAL` (500). Over REST, `DEADLINE_EXCEEDED` is
  503 on every method except `CreateKey` and `RotateKey`, where it is 504
  because a retry could create another key; over ConnectRPC it is
  `deadline_exceeded` everywhere. These are the statuses the service already
  answers with.
- `DepositInstructions.min_native_gas_limit` is documented as what it is: a
  recommended gas limit (250000) for a reliable native-coin deposit on EVM
  networks, which Polyester does not enforce and the sender's wallet or
  exchange sets. A network gas estimate of at least 50000 also works, more
  slowly, and a plain transfer's gas differs by network. Its value is
  unchanged.

## How clients use it

Nothing changes in calls or responses. A client can read an RPC's possible
codes from its descriptor options, for example to map them to messages:

```bash
# CreateQuickSwap declares, among others:
#   option (polyester.api.error_codes) = "IDEMPOTENCY_CONFLICT";  # ALREADY_EXISTS, HTTP 409
#   option (polyester.api.error_codes) = "ROUTE_SUSPENDED";       # FAILED_PRECONDITION, HTTP 400
curl -sS https://<api-host>/swap.quickswap.v1.QuickSwapService/CreateQuickSwap \
  -H 'Authorization: Bearer <broker-api-key>' \
  -H 'Content-Type: application/json' -d '{...}'
```

The error code is in `swap.quickswap.v1.ErrorDetail.code` (ConnectRPC) or in
`code` of the `application/problem+json` body (REST).

## Migration

None: the option is new documentation metadata and no response changes.
