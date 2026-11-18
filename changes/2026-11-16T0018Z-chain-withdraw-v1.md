---
services:
  - chain.withdraw.v1.WithdrawService
  - chain.withdraw.v1.AutoWithdrawService
  - swap.quickswap.v1.QuickSwapBrokerService
kind: changed
breaking: false
wire_compatible: true
---

## What and why

A withdrawal to another network is now refused when the destination route's
zToken supply is not current, not only when it is unknown. Before, a
withdrawal was checked against the last supply ever read, however old: while
the supply had stopped refreshing, a withdrawal the route could no longer
cover was accepted, its funds reserved, and it failed later instead of being
refused up front.

- `ERROR_CODE_SUPPLY_UNAVAILABLE` (Trading withdraw-to-chain) now also covers
  a route whose supply is STALE. Its documented meaning: the zToken supply of
  the destination route is not known, or not current, as
  `AssetChainVariant.supply_status` `MISSING` or `STALE` in the deposit and
  withdraw config shows. Nothing was reserved. It stays a retryable
  `unavailable` error.
- Auto-withdraw run steps and QuickSwap broker vault payout steps gain the
  step reason `supply_unavailable`: the step was skipped because the
  destination route's supply was not known or not current, and the balance
  stays for a later run. A payout whose steps were all skipped this way reads
  as rolled over, as with `route_suspended`.

## How clients use it

Nothing new to call. Before a withdrawal, read the route's supply status from
the deposit and withdraw config:

```text
POST /chain.zipper.v1.ZipperService/GetDepositWithdrawConfig
{}
```

or `GET /v1/chain/deposit-withdraw/config`. A variant whose `supplyStatus` is
`STALE` or `MISSING` refuses new withdrawals with `SUPPLY_UNAVAILABLE` until
it is `AVAILABLE` again. Retry the same withdrawal then.

## Migration

None: the error code and the step reason field already exist. Clients that
show step reasons should add a label for `supply_unavailable`, for example
"Withdrawals to this network are briefly unavailable; the balance stays for
the next run".
