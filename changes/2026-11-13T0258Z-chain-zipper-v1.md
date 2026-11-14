---
services:
  - chain.zipper.v1.ZipperService
kind: added
breaking: false
wire_compatible: true
---

## What and why

Each chain variant of an asset in `GetDepositWithdrawConfig` now says whether
its route currently accepts new deposits and new withdrawals, so a client can
show a network as suspended and stop a user before they send funds or start a
withdrawal that would not be accepted.

`AssetChainVariant` gains two fields of the new enum `RouteStatus`:

- `deposit_status` (13): whether the route accepts new deposits.
- `withdrawal_status` (14): whether the route accepts new withdrawals.

`RouteStatus` values:

- `NORMAL`: the route accepts new requests.
- `SUSPENDED`: the route does not accept new requests. Requests created
  before the suspension continue to process.
- `STATUS_UNSPECIFIED`: availability is not known yet. Do not treat the route
  as available.

A suspension can stop one direction and not the other. Funds sent to a
deposit address while deposits are suspended are delayed, not lost: they can
be processed only after deposits resume, so show such a deposit as delayed.

## How clients use it

Read the two fields from each variant of the existing config call:

```http
POST /chain.zipper.v1.ZipperService/GetDepositWithdrawConfig
{}
```

```json
{
  "assets": [{
    "asset": "USDT",
    "variants": [{
      "zippedAssetId": 3,
      "chainId": 2,
      "depositStatus": "NORMAL",
      "withdrawalStatus": "SUSPENDED"
    }]
  }]
}
```

REST `GET /v1/chain/deposit-withdraw/config` returns the same two fields per
variant as `depositStatus` and `withdrawalStatus` with the values `NORMAL` and
`SUSPENDED`, and omits a field while its availability is not known.

The response stays cacheable for 10 seconds; poll it to see a suspension
start or end.

## Migration

None: the fields are new. A client that does not read them keeps working;
one that does should treat an absent or `STATUS_UNSPECIFIED` value as not
available, not as normal.
