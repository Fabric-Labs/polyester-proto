---
services:
  - swap.quickswap.v1.QuickSwapService
  - swap.quickswap.v1.QuickSwapBrokerService
kind: changed
breaking: true
wire_compatible: true
---

## What and why

A deposit that is dropped or replaced on its network before it confirms
brings no funds, also when it was still waiting for processing capacity.
Before, a QuickSwap whose only deposit was dropped while it waited stayed
`WAITING_FOR_CAPACITY`, and could never expire. Now it goes back to
`AWAITING_DEPOSIT` and no longer lists that deposit in `deposits`, exactly as
a QuickSwap whose dropped deposit was already `PREPARING_DEPOSIT` does. After
`funding_expires_at` it becomes `EXPIRED` with `FUNDING_WINDOW_ENDED` and
`terminal_at` equal to `funding_expires_at`, as one that never received
anything.

Moving back to `AWAITING_DEPOSIT` is still the only status that moves back;
it can now come from `WAITING_FOR_CAPACITY` as well as from
`PREPARING_DEPOSIT`. Rarely, such a deposit confirms after all before
`funding_expires_at`: it is then listed again and the status moves forward
as usual (`WAITING_FOR_CAPACITY`, `PREPARING_DEPOSIT`, and so on).

No field, enum value or error code changes.

## How clients use it

Poll `GetQuickSwap` as before, and handle a move from `WAITING_FOR_CAPACITY`
back to `AWAITING_DEPOSIT` exactly as the move from `PREPARING_DEPOSIT`.

```http
POST /swap.quickswap.v1.QuickSwapService/GetQuickSwap
{"swapId": "..."}

200 {"quickSwap": {"status": "AWAITING_DEPOSIT", "deposits": [], "isTerminal": false, ...}}
```

```http
GET /v1/quickswaps/{swap_id}
```

## Migration

A client that accepts the move back to `AWAITING_DEPOSIT` only from
`PREPARING_DEPOSIT` must accept it from `WAITING_FOR_CAPACITY` too. A client
that treats any move back to `AWAITING_DEPOSIT` as a dropped deposit needs no
change.
