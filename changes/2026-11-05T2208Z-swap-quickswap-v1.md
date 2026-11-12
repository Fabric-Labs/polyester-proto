---
services:
  - swap.quickswap.v1.QuickSwapService
kind: changed
breaking: false
wire_compatible: true
---

## What and why

A QuickSwap could end with nothing to withdraw: every amount left was
smaller than one unit of its token on its network, for example when all the
source traded but the intermediate funds were too small for the second leg
and also less than one unit of the refund token. Such a QuickSwap stayed on
its last status (for example `EXECUTING`) and never became terminal.

It now ends like any refund or return below the withdrawal minimum:
`REFUNDED` (or `RETURNED` when only the untraded input is left) with reason
`REFUND_BELOW_MINIMUM`, `is_terminal` true, and that withdrawal `NOT_SENT`.
Its `unsent_amount` is rounded down to the token's smallest unit, as
documented, so it is `0`. No message, field, enum value or RPC changes.

A swap that also pays a withdrawal is unchanged: a remainder below one token
unit beside a payout is rounding and is not listed.

Not covered yet: a QuickSwap whose only leftover is destination output below
one unit of the destination token (possible for a deposit smaller than the
quoted one, to a destination with no withdrawal minimum) can still stay on
its last status, for example `EXECUTING`, until a later change handles it.

## How clients use it

Read the status, reason and withdrawals from `GetQuickSwap`, as before:

```http
POST https://<api-host>/swap.quickswap.v1.QuickSwapService/GetQuickSwap
Content-Type: application/json

{"swapId": "<swap_id>"}
```

Such a QuickSwap returns `status` `REFUNDED`, `reason`
`REFUND_BELOW_MINIMUM`, and one `REFUND` withdrawal in state `NOT_SENT` with
`unsent_amount` `{"baseUnits": "0", "decimals": <token decimals>}`.

## Migration

None. The status, reason and withdrawal state already exist and are already
returned for refunds and returns below the withdrawal minimum. Clients that
stop polling at a terminal status now stop for these QuickSwaps too.
