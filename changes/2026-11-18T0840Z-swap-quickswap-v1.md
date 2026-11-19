---
services:
  - swap.quickswap.v1.QuickSwapService
kind: changed
breaking: false
wire_compatible: true
---

## What and why

Documentation of an existing guarantee is corrected; no message, field or enum changes.
`QuickSwapQuote.min_deposit_amount`, the `RETURN_ONLY` deposit disposition and the
`DEPOSIT_BELOW_MINIMUM` reason now state that a deposit below `min_deposit_amount` is returned only if
it reaches the source network's minimum deposit (`AssetChainVariant.deposit_min_amount` in the
deposit and withdraw config, for `DepositInstructions.zipped_asset_id`). A deposit below that network
minimum is not detected: it never appears on the QuickSwap, is not returned and is lost.
`min_deposit_amount` is never below the network minimum. For a bridge it often equals it; when they
are equal, any smaller deposit is lost.

A route whose minimum deposit or withdrawal minimum is not configured is no longer quoted: quote and
create answer the existing `QUOTE_UNAVAILABLE` error code.

## How clients use it

Read `min_deposit_amount` from the quote and tell the end user to send at least that amount:

```text
POST /swap.quickswap.v1.QuickSwapService/QuoteQuickSwap
{"terms": {...}}
-> {"quote": {"minDepositAmount": {"baseUnits": "2200000", "decimals": 6}, ...}}
```

The network minimum is `depositMinAmount` of the matching variant in
`GET /v1/chain/deposit-withdraw/config`, matched on the deposit instructions' `zippedAssetId`.

## Migration

None: nothing on the wire changes. Clients that tell end users a deposit below the minimum is
always returned should instead say to send at least `min_deposit_amount`, because a deposit below the
network's minimum deposit is lost.
