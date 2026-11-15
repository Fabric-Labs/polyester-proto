---
services:
  - swap.quickswap.v1.QuickSwapService
  - swap.quickswap.v1.QuickSwapBrokerService
kind: changed
breaking: false
wire_compatible: true
---

## What and why

A QuickSwap whose only deposit was dropped or replaced on its network goes
back to `AWAITING_DEPOSIT`, as before. It now becomes `EXPIRED` with
`FUNDING_WINDOW_ENDED` shortly after `funding_expires_at`, with
`terminal_at` equal to `funding_expires_at`, exactly as a QuickSwap that never
received anything. Before, it could stay `AWAITING_DEPOSIT` past
`funding_expires_at` until its deposit address was retired, which could be
days later; that no longer happens.

Nothing else changes: `address_retires_at` and the deposit address are
unchanged, a deposit detected after `funding_expires_at` is still listed
`RETURN_ONLY` until the address is retired, and a dropped or replaced deposit
that confirms after all still turns an `EXPIRED` QuickSwap `HELD`, with that
deposit `ON_HOLD`.

No field, enum value or error code changes.

## How clients use it

Poll `GetQuickSwap` as before. An `AWAITING_DEPOSIT` QuickSwap past
`funding_expires_at` turns `EXPIRED` within minutes.

```http
POST /swap.quickswap.v1.QuickSwapService/GetQuickSwap
{"swapId": "..."}

200 {"quickSwap": {"status": "EXPIRED", "reason": "FUNDING_WINDOW_ENDED", "isTerminal": true, ...}}
```

```http
GET /v1/quickswaps/{swap_id}
```

## Migration

None: a client that waited past `funding_expires_at` for `EXPIRED` after a
dropped or replaced deposit keeps working, and now gets it soon after the
deadline. Clients may drop any special case for a long `AWAITING_DEPOSIT`
after `funding_expires_at`.
