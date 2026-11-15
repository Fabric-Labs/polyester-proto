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
brings no funds. Before, a QuickSwap whose only deposit was dropped stayed
`PREPARING_DEPOSIT` for good, so it looked as if a deposit was still in
progress. Now it goes back to `AWAITING_DEPOSIT` and no longer lists that
deposit in `deposits`, exactly as before the deposit was seen. This is the
only status that moves back.

- While the deposit address still accepts deposits, the user can simply send
  again: the new deposit progresses as usual (`WAITING_FOR_CAPACITY`,
  `PREPARING_DEPOSIT`, `CONFIRMING`, and so on).
- Otherwise the QuickSwap ends `EXPIRED` with `FUNDING_WINDOW_ENDED` and
  `terminal_at` equal to `funding_expires_at`, as one that never received
  anything. After a dropped deposit, that can happen later than
  `funding_expires_at`: the status can stay `AWAITING_DEPOSIT` past
  `funding_expires_at` until the deposit address is retired. Until then, a
  deposit detected after `funding_expires_at` is listed `RETURN_ONLY`; once
  the address is retired, a deposit sent to it is listed `MANUAL_RECOVERY`
  and is not returned automatically.
- Rarely, a dropped or replaced deposit confirms after all. Its funds are
  then held for review: the QuickSwap shows `HELD` with that deposit
  `ON_HOLD`, also when it was already `EXPIRED`. That is the one change of a
  terminal status; `is_terminal` becomes false.

No field, enum value or error code changes.

## How clients use it

Poll `GetQuickSwap` as before, and handle a move from `PREPARING_DEPOSIT`
back to `AWAITING_DEPOSIT`, and from `EXPIRED` to `HELD`. Broker views of a
QuickSwap show the same status.

```http
POST /swap.quickswap.v1.QuickSwapService/GetQuickSwap
{"swapId": "..."}

200 {"quickSwap": {"status": "AWAITING_DEPOSIT", "deposits": [], "isTerminal": false, ...}}
```

```http
GET /v1/quickswaps/{swap_id}
```

To decide whether to ask the user to send again, compare the current time
with `funding_expires_at`: before it, show the deposit instructions again;
after it, do not ask for a new deposit, and wait for `EXPIRED`. An `EXPIRED`
QuickSwap that becomes `HELD` follows the usual `HELD` handling: the broker
contacts Polyester broker support with the `swap_id`.

## Migration

- A client that assumed statuses only move forward, or that a listed deposit
  is never removed, must accept `PREPARING_DEPOSIT` followed by
  `AWAITING_DEPOSIT` with an empty `deposits`, and an `AWAITING_DEPOSIT`
  QuickSwap past `funding_expires_at`.
- A client that treats `EXPIRED` as final must accept `EXPIRED` followed by
  `HELD`. Polling may still stop at a terminal status; read an `EXPIRED`
  QuickSwap again when the end user reports a deposit to it.
