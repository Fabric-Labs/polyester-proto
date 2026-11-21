---
services:
  - swap.quickswap.v1.QuickSwapService
  - swap.quickswap.v1.QuickSwapBrokerService
kind: added
breaking: false
wire_compatible: true
---

## What and why

A QuickSwap quote now discloses the network fees that are withheld only when
funds are not delivered. Until now a quote priced the deposit and destination
network fees, but a refund or a return could be charged a network fee the end
user never saw before swapping.

- `QuickSwapQuote.contingent_fees` is a new list of `QuoteFee`, separate from
  `fees`. It holds `RETURN_NETWORK`, the estimated network fee of returning
  untraded source funds on the source route, on every route, and
  `REFUND_NETWORK`, the estimated network fee of refunding unconverted
  intermediate asset on the refund route of a two-leg route, when that route
  is known. An entry is never omitted: a known fee of zero is listed with
  amount `"0"`.
- `FeeKind` gains `RETURN_NETWORK` and `REFUND_NETWORK`. They appear only in
  `contingent_fees`.
- `QuoteQuickSwapRequest.refund_zipped_asset_id` is a new optional field. It
  takes the same refund route as `CreateQuickSwapRequest.refund_zipped_asset_id`
  and is validated the same way (`REFUND_ADDRESS_INVALID`). On a two-leg route
  it fixes the intermediate asset, as create does, and adds `REFUND_NETWORK`
  to the quote. It is unused on a direct route or a bridge.

Both lines are estimates: the fee withheld is the network's fee at payout
time. They are not part of `fees`, `expected_amount_out`, `min_amount_out` or
`min_rate`, which are unchanged. A partly traded two-leg QuickSwap can be
charged both. `RETURN_NETWORK` is always present, and so is `REFUND_NETWORK`
once the refund route is known, even when the sampled fee is zero (amount
`"0"`). Only a quote whose return or refund route has no known network fee
fails with `QUOTE_UNAVAILABLE`. The quote of a created two-leg QuickSwap
always carries `REFUND_NETWORK`, because create requires the refund route.

## How clients use it

Quote with the refund route the QuickSwap will use, then show the worst case
next to the expected output:

```http
POST /swap.quickswap.v1.QuickSwapService/QuoteQuickSwap
{"terms": {"sourceZippedAssetId": 7, "destinationZippedAssetId": 42, "basis": "SOURCE_AMOUNT",
           "amount": {"baseUnits": "5000000", "decimals": 8},
           "protection": {"maxSlippageBps": 75}, "execution": {"market": {}}},
 "refundZippedAssetId": 12}
```

REST takes the same body:

```http
POST /v1/quickswaps:quote
```

The response quote carries, for example:

```json
"contingentFees": [
  {"kind": "RETURN_NETWORK", "zippedAssetId": 7, "amount": {"baseUnits": "1500", "decimals": 8}},
  {"kind": "REFUND_NETWORK", "zippedAssetId": 12, "amount": {"baseUnits": "720000", "decimals": 6}}
]
```

Each amount is in the token units of its `zipped_asset_id`'s route, and `bps`
is zero. `contingentFees` is also on the quote of every QuickSwap that
`CreateQuickSwap`, `GetQuickSwap` and `QuickSwapBrokerService.GetQuickSwap`
return, as fixed at creation.

## Migration

None. The new field and enum values are additive. Clients that add up `fees`
for the cost of a delivered QuickSwap keep the same totals. A client that maps
every `FeeKind` value should map `RETURN_NETWORK` and `REFUND_NETWORK`, and
show `contingentFees` as charged only if funds are returned or refunded.
