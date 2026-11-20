---
services:
  - swap.quickswap.v1.QuickSwapExplorerService
kind: added
breaking: false
wire_compatible: true
---

## What and why

The public QuickSwap Explorer gains two things:

- `ExplorerQuickSwap` carries `source_value_usd` and `destination_value_usd`: the USD values of the deposit
  and of what the destination was paid, fixed when each happened, the same values `QuickSwap` reports as
  `deposited_value_usd` and `delivered_value_usd` and that `GetQuickSwapStats` sums. A swap's value no
  longer has to be estimated at today's prices. Each is unset when the QuickSwap has none (never zero for
  unknown); `destination_value_usd` can appear a few minutes after `completed_at`.
- New RPC `GetCompletedQuickSwap` reads one completed QuickSwap of any broker by `swap_id`, as
  `ListCompletedQuickSwaps` lists it, with no credentials. A QuickSwap that has not completed, whatever its
  status, returns the same `NOT_FOUND` / `SWAP_NOT_FOUND` as an unknown `swap_id`.

## How clients use it

ConnectRPC:

```http
POST /swap.quickswap.v1.QuickSwapExplorerService/GetCompletedQuickSwap
Content-Type: application/json

{"swapId": "7433029815562201289"}
```

REST:

```http
GET /v1/quickswap/explorer/quickswaps/7433029815562201289
```

Both answer `{"quickSwap": {...}}` with the `ExplorerQuickSwap` fields, including the new
`sourceValueUsd` and `destinationValueUsd` (`TokenAmount`, 18 decimals).

When no completed QuickSwap has the swap ID, both answer one and the same error: ConnectRPC `NOT_FOUND`,
REST HTTP 404, each with code `SWAP_NOT_FOUND`. That covers an ID no QuickSwap has, a QuickSwap that has
not completed (whatever its status), `0` or an absent `swapId`, and a REST path segment that is not a
decimal unsigned 64-bit integer.

The call is rate limited per client address, like the list. Over the limit, ConnectRPC answers
`RESOURCE_EXHAUSTED` and REST answers HTTP 429 with code `RATE_LIMIT_EXCEEDED`, both with a
`Retry-After` header in seconds.

## Migration

None: the RPC and fields are new. `QuickSwapService.GetQuickSwap` is unchanged and still reads only the
calling broker's QuickSwaps.
