---
services:
  - swap.quickswap.v1.QuickSwapService
kind: changed
breaking: false
wire_compatible: true
---

## What and why

A QuickSwap's guarantee is its minimum output, rounded down to the
smallest ledger amount. When the source a two-leg route executed is so
small that this guarantee rounds to nothing, a second leg that sells has no
positive price to protect. Two outcomes change for that case.

- **Execution.** On a second-leg market with no minimum notional, such a
  QuickSwap kept retrying until its execution deadline. It then ended with
  `EXECUTION_DEADLINE_REACHED`, refunding its intermediate funds and
  returning its untraded source. The intermediate funds are now treated as
  below the second leg's market minimums, as they already were on markets
  with a minimum notional. A market execution resolves at once. A TWAP adds
  them to the next slice's conversion, and refunds what is still
  unconverted when execution ends, as for any intermediate funds below the
  second leg's minimums.
- **Quote.** A quote whose guarantee rounds to nothing could fail without
  an answer. That happened for a TWAP whose every slice guarantees nothing,
  for example a `min_amount_out` of a few base units of an 18-decimal
  destination split into more slices than that. It also happened for a
  `min_amount_out` below one ledger unit of a destination with more than 18
  decimals. A TWAP quote is now refused with `PROTECTION_UNACHIEVABLE`, and a
  market quote with `AMOUNT_OUT_OF_RANGE`.

No message, field, enum value or RPC changes.

## How clients use it

Quote and read QuickSwaps as before:

```http
POST https://<api-host>/swap.quickswap.v1.QuickSwapService/QuoteQuickSwap
POST https://<api-host>/swap.quickswap.v1.QuickSwapService/GetQuickSwap
Content-Type: application/json
```

A refused TWAP quote returns ConnectRPC `failed_precondition` with error
code `PROTECTION_UNACHIEVABLE`. A refused market quote returns
`invalid_argument` with `AMOUNT_OUT_OF_RANGE`. In this case, unlike a
protection the market cannot meet, the protection is too small to
guarantee anything. When the second leg sells, a larger deposit does not
help: quote a `min_amount_out` of at least one ledger unit (10^-18 of the
destination asset) per slice, or use fewer slices. For a TWAP to a
destination with more than 18 decimals, every `min_amount_out` can be
refused: use a `max_slippage_bps` protection instead. When the second leg buys, the
refusal can instead come from intermediate funds too small to buy anything,
which a larger `min_amount_out` does not fix: deposit more, or for a TWAP
use fewer slices.

Take a QuickSwap whose second leg could not convert intermediate funds that
guarantee nothing. If those funds are refunded, `GetQuickSwap` shows
`INTERMEDIATE_BELOW_MINIMUM`. If they are below the refund route's
withdrawal minimum, it shows `REFUND_BELOW_MINIMUM`, with the refund
`NOT_SENT` and the amount in `unsent_amount`. Before this change, either
case could show `EXECUTION_DEADLINE_REACHED`.

## Migration

None. `PROTECTION_UNACHIEVABLE`, `AMOUNT_OUT_OF_RANGE`,
`INTERMEDIATE_BELOW_MINIMUM` and `REFUND_BELOW_MINIMUM` already exist, and
clients already handle them for other quotes and QuickSwaps. Requests that
used to fail without an answer now get one of these refusals.
