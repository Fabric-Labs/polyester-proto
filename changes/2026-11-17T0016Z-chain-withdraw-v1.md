---
services:
  - chain.withdraw.v1.WithdrawService
kind: changed
breaking: true
wire_compatible: true
---

## What and why

`GetWalletBalances` now answers the balances it can read even when one
zipped asset's zToken balance cannot be read, instead of failing the whole
request as unavailable. Each asset it could not read is listed in the new
`GetWalletBalancesResponse.unavailable_assets`, ordered by
`zipped_asset_id`, with its zipped asset, unified asset, network, zToken
address and a `reason`:

| `reason` | Meaning |
| --- | --- |
| `NO_CONTRACT` | No zToken contract exists at the listed address on Polyester Chain. |
| `READ_FAILED` | Polyester Chain answered the balance read with an error. |
| `INVALID_BALANCE` | The balance read returned a value that is not a valid balance. |

An asset in `unavailable_assets` has an unknown balance, not a zero one: the
wallet may hold some of it. When no asset's balance can be read at all, the
request still fails as unavailable, as before.

## How clients use it

```text
GET /v1/chain/wallet/balances

200 application/json
{"smartAccountAddress":"0x5a5a…5a5a",
 "balances":[{"zippedAssetId":7,"unifiedAssetId":3,"chainId":2,"ztokenAddress":"0x1111…1111","amount":"1.5"}],
 "unavailableAssets":[{"zippedAssetId":700007,"unifiedAssetId":0,"chainId":8453,"ztokenAddress":"0x0000…0007","reason":"NO_CONTRACT"}]}
```

ConnectRPC: `/chain.withdraw.v1.WithdrawService/GetWalletBalances` returns
`unavailable_assets` with `reason` as `NO_CONTRACT`, `READ_FAILED` or
`INVALID_BALANCE`.

## Migration

A successful answer can now be incomplete. Before, an asset missing from
`balances` always meant the wallet held none of it. Now it may instead be in
`unavailableAssets`, and then its balance is unknown.

- Read `unavailableAssets` (ConnectRPC `unavailable_assets`) on every
  answer.
- Show each asset there as "balance unavailable", never as zero.
- Do not add an unavailable asset into a total, and do not decide anything
  that depends on its balance, such as offering to move it, until a later
  read returns its balance.
