---
services:
  - swap.quickswap.v1.QuickSwapService
kind: changed
breaking: false
wire_compatible: true
---

## What and why

`QuoteQuickSwapRequest.source_wallet` and `CreateQuickSwapRequest.source_wallet`
now state which networks screen a declared source wallet in advance: Bitcoin,
Solana and most EVM networks. Until now only EVM wallets were screened in
advance. Bitcoin and Solana wallets, for example from wallet apps on Solana,
are now screened too, so a flagged Bitcoin or Solana wallet gets
`SOURCE_WALLET_REJECTED` before any quote or deposit address, as an EVM
wallet already did. Wallets on other networks, including the Bitcoin and
Solana test networks, are still quoted and created without advance
screening, and their deposits are screened on arrival. Only comments change;
no message, field or enum changes.

## How clients use it

Unchanged: pass the end user's wallet as `source_wallet` on
`/swap.quickswap.v1.QuickSwapService/QuoteQuickSwap` and
`/swap.quickswap.v1.QuickSwapService/CreateQuickSwap` (REST
`POST /v1/quickswaps:quote` and `POST /v1/quickswaps`).

```json
{ "terms": { "sourceZippedAssetId": 7, "destinationZippedAssetId": 12, "basis": "SOURCE_AMOUNT", "amount": { "baseUnits": "5000000", "decimals": 8 } }, "sourceWallet": "bc1qar0srrr7xfkvy5l643lydnw9re59gtzzwf5mdq" }
```

A Bitcoin or Solana wallet that fails screening now returns
`PERMISSION_DENIED` with `SOURCE_WALLET_REJECTED`, which clients already
handle for EVM wallets.

## Migration

None: the contract shape is unchanged. Clients that handle
`SOURCE_WALLET_REJECTED` already cover Bitcoin and Solana wallets.
