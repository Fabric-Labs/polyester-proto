---
services:
  - swap.quickswap.v1.QuickSwapService
kind: changed
breaking: false
wire_compatible: true
---

## What and why

On a two-leg route, a first leg can fill only part of its order and leave
intermediate funds too small for a second leg that buys to purchase one whole
quantity step, even at the market's lowest price (one tick). That includes
funds below one quantity increment of the asset the second leg spends. No
second-leg order can carry them, so they are now classified as below the
second leg's market minimums. Until now they were classified as a
protection failure, although the protection was never the obstacle. In the
rarest form of the case, the QuickSwap could also stall until its execution
window ended before resolving.

What happens to the unconverted funds is unchanged: they follow the refund
rules of `refund_address`'s route. Funds below that route's withdrawal
minimum are not sent. The QuickSwap then shows `REFUND_BELOW_MINIMUM`, with
the refund withdrawal `NOT_SENT` and the amount in `unsent_amount`. Funds
less than one quantity increment are normally below that minimum, so this is
the usual outcome for the smallest of these funds. Only when no withdrawal
minimum is known for that route is a refund requested. The QuickSwap then
shows `INTERMEDIATE_BELOW_MINIMUM` where it previously could show
`PROTECTION_NOT_MET`. The protection does not change, and neither does any
amount. No message, field, enum value or RPC changes.

## How clients use it

Read `QuickSwap.reason` and its withdrawals from `GetQuickSwap`, as before:

```http
POST https://<api-host>/swap.quickswap.v1.QuickSwapService/GetQuickSwap
Content-Type: application/json

{"swapId": "<swap_id>"}
```

On a two-leg route whose first leg traded, take intermediate funds that are
too small for any second-leg order: below what one quantity step costs at one
tick, below one quantity increment of the second leg's spent asset, or below
its minimum quantity or minimum notional. If such funds are refunded, the
reason is `INTERMEDIATE_BELOW_MINIMUM`. If they are below the refund route's
withdrawal minimum, the reason is `REFUND_BELOW_MINIMUM` with the refund
`NOT_SENT`.

`PROTECTION_NOT_MET` remains the reason when the intermediate funds can buy
at least one whole step at one tick, but no second-leg price meets the
QuickSwap's protection.

## Migration

None. `INTERMEDIATE_BELOW_MINIMUM` and `REFUND_BELOW_MINIMUM` already exist,
and are already returned for intermediate funds below the second leg's
market minimums. Clients that handle the refund outcome from the withdrawals
and their states need no change. Clients that show a reason-specific message
now show the intermediate-minimum message for this case.
