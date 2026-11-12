---
services:
  - swap.quickswap.v1.QuickSwapService
kind: changed
breaking: false
wire_compatible: true
---

## What and why

A credited deposit that cannot be executed is now returned to
`return_address` on the network it was received on. Previously it was held
for review (`HELD`) until Polyester broker support acted.

- New enum value `QuickSwapReason.MARKET_CLOSED = 27`. A market on the route
  was halted, restricted or delisted when the deposit was credited, and did
  not reopen within a bounded wait after the deposit was credited (the same
  for market and TWAP execution; not the TWAP duration). Nothing was traded. While the
  market can still reopen, the QuickSwap stays `CREDITED` and executes as
  soon as it reopens. Otherwise it shows `RETURNING`, then `RETURNED`, with
  `MARKET_CLOSED`, and its deposit is `RETURN_ONLY` with the same reason.
- `DEPOSIT_BELOW_MINIMUM` now also covers a deposit whose credit after the
  deposit network fee is below the smallest order the route's market
  accepts. `DEPOSIT_ABOVE_MAXIMUM` now also covers a deposit above the
  largest order the route's market accepts. Both are returned as before:
  `RETURNING`, then `RETURNED`, with the reason.
- `PROTECTION_NOT_MET`: when no protected order can be placed for the
  deposit, the deposit is returned (`RETURNING`, then `RETURNED`) instead of
  `HELD`.

## How clients use it

Read the QuickSwap as before, and handle the new reason in the status
display:

```http
POST https://<api-host>/swap.quickswap.v1.QuickSwapService/GetQuickSwap
Content-Type: application/json

{"swapId": "..."}
```

REST: `GET /v1/quickswaps/{swap_id}`. A returned deposit has `status`
`RETURNING` or `RETURNED`, `reason` `MARKET_CLOSED` (or one of the reasons
above), `deposits[].disposition` `RETURN_ONLY`, and one `RETURN` withdrawal.

## Migration

None for wire compatibility: one enum value is added and no field changes.
Clients that switch on `QuickSwapReason` should map `MARKET_CLOSED`, for
example to "Market closed: deposit returned". Clients that expected these
cases to show `HELD` will now see `RETURNING` and `RETURNED`.
