---
services:
  - swap.quickswap.v1.QuickSwapService
  - swap.quickswap.v1.QuickSwapBrokerService
kind: changed
breaking: false
wire_compatible: true
---

## What and why

`QuickSwapReason` gains two values that explain why a funded QuickSwap was returned or refunded after its trade failed:

- `INSUFFICIENT_LIQUIDITY` (29): available liquidity did not fill an order at its protected price.
- `ORDER_REJECTED` (30): an order was refused.

Until now such a QuickSwap ended `RETURNED` or `REFUNDED` with `reason: REASON_UNSPECIFIED`, and the cause was only on `execution.slices[].reason`. The QuickSwap's own `reason` now names it, so an app can tell the user why the swap came back.

## How clients use it

The calls are unchanged; read `reason` as before:

```http
POST /swap.quickswap.v1.QuickSwapService/GetQuickSwap
Content-Type: application/json

{"swapId":"1234567890"}
```

```http
GET /v1/quickswaps/1234567890
```

```json
{"quickSwap":{"swapId":"1234567890","status":"RETURNED","reason":"INSUFFICIENT_LIQUIDITY","isTerminal":true,
 "execution":{"slices":[{"index":0,"state":"SKIPPED","reason":"INSUFFICIENT_LIQUIDITY"}]}}}
```

- The reason appears on the payout statuses after execution: `RETURNING`, `RETURNED`, `REFUNDING`, `REFUNDED`, `PAYING_OUT` and `COMPLETED` (a partly traded swap whose rest was refunded).
- A more specific outcome of the execution keeps precedence: `EXECUTION_DEADLINE_REACHED`, `PROTECTION_NOT_MET` and `INTERMEDIATE_BELOW_MINIMUM` are unchanged, and so are `REFUND_BELOW_MINIMUM` and `WITHDRAWAL_FAILED`.
- When several slices failed, the reason comes from the first slice in schedule order whose `reason` or `carryReason` is `INSUFFICIENT_LIQUIDITY` or `ORDER_REJECTED`; within a slice, `reason` comes before `carryReason`.
- Reasons for a deposit that was never traded (`WRONG_ASSET`, `DEPOSIT_ADDRESS_ALREADY_USED` and the others) are unchanged.
- `QuickSwapBrokerService.ListQuickSwaps` and `GetQuickSwap` show the same `reason` on each QuickSwap.

## Migration

None: the values are new. Clients that map `reason` to text should add the two values; a client that does not know them yet treats them like any unknown reason.
