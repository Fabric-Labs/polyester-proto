---
services:
  - swap.quickswap.v1.QuickSwapService
kind: changed
breaking: false
wire_compatible: true
---

## What and why

Two quote outcomes change. No message, field, enum value or RPC changes.

- **A guarantee that rounds to nothing on a route whose second leg buys.**
  A QuickSwap's guarantee is its minimum output, rounded down to the
  smallest ledger amount. When that guarantee rounds to nothing and the
  intermediate funds are also below the smallest amount the second leg's
  market spends, the quote could fail without an answer. That happened for a
  TWAP whose every slice guarantees nothing, and for a market quote with a
  `min_amount_out` below one ledger unit of a destination with more than 18
  decimals. A TWAP quote is now refused with `PROTECTION_UNACHIEVABLE`, and a
  market quote with `AMOUNT_OUT_OF_RANGE`: the same error codes as for a
  second leg that sells, with a different remedy.
- **Minimum deposit on a destination with no withdrawal minimum.**
  `min_deposit_amount` already made sure the guaranteed output reaches the
  destination's withdrawal minimum. When the destination has none, it now
  makes sure the guaranteed output is at least one base unit of the
  destination token. Before, a smaller deposit could be accepted and
  executed for an amount below one base unit, which no withdrawal can carry,
  and the QuickSwap could stay `EXECUTING`. Such a deposit is now below
  `min_deposit_amount`: it is returned to `return_address` with
  `DEPOSIT_BELOW_MINIMUM`. Only routes without a destination withdrawal
  minimum can quote a higher `min_deposit_amount`.

## How clients use it

Quote QuickSwaps as before:

```http
POST https://<api-host>/swap.quickswap.v1.QuickSwapService/QuoteQuickSwap
Content-Type: application/json
```

A refused TWAP quote returns ConnectRPC `failed_precondition` with error
code `PROTECTION_UNACHIEVABLE`; a refused market quote returns
`invalid_argument` with `AMOUNT_OUT_OF_RANGE`. Here each slice's
intermediate funds are too small to buy anything on the second leg, so
raising `min_amount_out` does not help: deposit more or use fewer slices.
For a market quote, deposit more.

Read `min_deposit_amount` from each quote rather than deriving it from the
source network's minimum: on a route whose destination has no withdrawal
minimum it can now be above that minimum.

## Migration

None. `PROTECTION_UNACHIEVABLE`, `AMOUNT_OUT_OF_RANGE` and
`DEPOSIT_BELOW_MINIMUM` already exist, and clients already handle them.
Requests that used to fail without an answer now get one of these refusals,
and a client that shows the quoted `min_deposit_amount` shows the new one.
