---
services:
  - swap.quickswap.v1.QuickSwapService
  - chain.withdraw.v1.WithdrawService
  - chain.zipper.v1.ZipperService
kind: changed
breaking: false
wire_compatible: true
---

## What and why

A route whose network is suspended is now refused up front instead of failing
later. The deposit and withdraw config already shows each network of an asset
with `deposit_status` and `withdrawal_status` (`NORMAL`, `SUSPENDED`, or
`STATUS_UNSPECIFIED` while not known). Two new error codes act on it:

- `swap.quickswap.v1.ErrorCode.ROUTE_SUSPENDED` (54): `QuoteQuickSwap` and
  `CreateQuickSwap` refuse a route while deposits on its source network or
  withdrawals on its destination network are suspended. Nothing is bound to
  the idempotency key. QuickSwaps created before the suspension continue;
  their payouts wait and are paid once the network resumes.
- `chain.withdraw.v1.ErrorCode.ERROR_CODE_ROUTE_SUSPENDED` (50):
  `CreateTradingWithdraw` and `CreateWalletTradingWithdraw` refuse a new
  withdrawal to an external network whose withdrawals of the asset are
  suspended, before anything is reserved. Repeating an accepted request with
  the same idempotency key still returns the accepted withdrawal.

Only `SUSPENDED` refuses. A status that is not known yet refuses nothing.

The documentation of `chain.zipper.v1.AssetChainVariant.deposit_status` and
of `RouteStatus.SUSPENDED` no longer describes what happens to funds sent on a
suspended route or to requests created before the suspension. It now tells
the client what to do instead: while a route is `SUSPENDED`, do not send
funds on it; wait until it returns to `NORMAL`. No field, value or wire
behaviour changes.

## How clients use it

Both are `FAILED_PRECONDITION` over ConnectRPC and `400` over REST, with the
code in the error detail and in the problem's `code`.

```http
POST /v1/quickswaps:quote
{"terms": {"sourceZippedAssetId": 3, "destinationZippedAssetId": 58, ...}}

400 {"code": "ROUTE_SUSPENDED", "status": 400, ...}
```

```http
POST /v1/chain/trading-withdraws
{"payload": {"action": "TO_EXTERNAL_CHAIN", "assetId": 4, "destinationChainId": 3, ...}, ...}

400 {"code": "ROUTE_SUSPENDED", "status": 400, ...}
```

Read `depositStatus` and `withdrawalStatus` from the deposit and withdraw
config to grey out a suspended network before the user submits, and submit
again once it is `NORMAL`.

## Migration

None: the codes are additive. A client that does not know them sees an
unrecognized error code with `FAILED_PRECONDITION` and `400`, and a request
that was refused bound or reserved nothing.
