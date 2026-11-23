---
services:
  - swap.quickswap.v1.QuickSwapService
kind: changed
breaking: false
wire_compatible: true
---

## What and why

A bridge deposit that meets `min_deposit_amount` but is credited less than
one base unit of the destination token, after its actual deposit network
fee, is now not executed (`DEPOSIT_BELOW_MINIMUM`), like a deposit below
the minimum: it is returned to `return_address`, or, when that return is
below the withdrawal minimum of its asset and network, the QuickSwap ends
`RETURNED` with `REFUND_BELOW_MINIMUM` and the return `NOT_SENT`. The deposit
network fee is charged on the actual deposit and can exceed the quoted one.
Such a deposit used to be bridged into an amount no withdrawal can carry:
nothing was paid out, and the QuickSwap stayed on `CREDITED` without ever
reaching a terminal status.

The documentation of `QuickSwapQuote.min_deposit_amount` and
`QuickSwapReason.DEPOSIT_BELOW_MINIMUM` now names this case. No status,
reason, field or message was added or changed.

## How clients use it

No new call. Read the QuickSwap as before:

```http
POST /swap.quickswap.v1.QuickSwapService/GetQuickSwap
{"swapId": "1234567890123456789"}

GET /v1/quickswaps/1234567890123456789
```

Such a deposit shows `disposition: RETURN_ONLY` with reason
`DEPOSIT_BELOW_MINIMUM`, and the QuickSwap moves through `RETURNING` to
`RETURNED` with the same reason. When the return itself is below the
withdrawal minimum of its asset and network, the QuickSwap ends `RETURNED`
with `REFUND_BELOW_MINIMUM`, and the return is a `NOT_SENT` withdrawal.

## Migration

None: clients already handle `DEPOSIT_BELOW_MINIMUM` returns. A client that
assumed every deposit of at least `min_deposit_amount` executes should
expect such a return for a bridge when the deposit network fee rises after
the quote.
