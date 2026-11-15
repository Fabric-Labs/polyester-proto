---
services:
  - chain.withdraw.v1.AutoWithdrawService
  - swap.quickswap.v1.QuickSwapBrokerService
  - chain.zipper.v1.ZipperService
kind: changed
breaking: false
wire_compatible: true
---

## What and why

A network suspension now delays withdrawals that were already accepted
instead of failing them, as exchanges usually handle a network suspension.

- A Trading withdrawal accepted before its network's withdrawals were
  suspended, and not yet sent, stays pending with its funds held. It is sent
  once `withdrawal_status` in the deposit and withdraw config is `NORMAL`
  again; it is never failed or refunded because of the suspension. One
  already being sent at the moment the suspension takes effect may still
  fail, and its funds then return to the account. The documentation of
  `chain.zipper.v1.AssetChainVariant.withdrawal_status` now says this. No
  field, value or wire behaviour changes.
- An auto-withdraw run skips a destination whose network's withdrawals are
  suspended, as it skips a balance below its minimum: the step is `skipped`
  with the new reason `route_suspended`, nothing is withdrawn, and the funds
  stay on the account for a later run. A run whose steps are all skipped
  ends `skipped`. On a QuickSwap broker's vault the payout rolls over
  (`PAYOUT_ROLLED_OVER`) with reason `route_suspended`.

`route_suspended` names the status only, never why the network is
suspended.

## How clients use it

Read the reason from the run steps as before:

```http
GET /v1/auto-withdraw/runs

200 {"runs": [{"state": "skipped", "steps": [{"state": "skipped", "reason": "route_suspended", ...}], ...}]}
```

ConnectRPC: `/chain.withdraw.v1.AutoWithdrawService/ListAutoWithdrawRuns`,
`AutoWithdrawStep.reason`; broker payouts:
`/swap.quickswap.v1.QuickSwapBrokerService/ListPayouts`, `Payout.reason` and
`PayoutStep.reason`.

A pending Trading withdrawal needs no action: show it as pending while
`withdrawalStatus` is `SUSPENDED`.

## Migration

None: `reason` is an open string and the new value is additive. A client
that shows unknown reasons verbatim shows `route_suspended`.
