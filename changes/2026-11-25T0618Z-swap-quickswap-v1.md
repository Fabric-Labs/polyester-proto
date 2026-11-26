---
services:
  - swap.quickswap.v1.QuickSwapBrokerService
kind: added
breaking: false
wire_compatible: true
---

## What and why

A broker vault's payout rule can convert every other asset into one payout
asset before it pays. The broker payout reads now describe that, so pages
that list payouts across a broker's vaults show it without one call per
vault:

- `ListVaultPayoutRules`: `VaultPayoutRule.payout_asset_id` and
  `VaultPayoutRule.conversion_twap_duration_ms` (both zero for a rule that
  pays every asset in kind).
- `ListUpcomingPayouts`: `UpcomingPayoutAsset.conversion`, a new
  `UpcomingPayoutConversion` with the current quote for converting the
  asset: `payout_asset_id`, `expected_amount_out`, `min_amount_out`,
  `price_impact_bps`, `execution` (market or TWAP) and `skip_reason` (why
  it would not convert). An asset that converts has `hold_reason`
  `converts`, pays nothing in kind and holds nothing back; the payout
  asset's entry includes every expected output.
- `ListPayouts` and `GetPayout`: `Payout.conversions`, a new
  `PayoutConversion` per converted asset: `source_asset_id`,
  `payout_asset_id`, `amount`, `swap_id`, `execution`, `status`, `reason`,
  `is_terminal`, `executed_amount`, `unspent_amount`, `received_amount` and
  `skip_reason` (for example `policy_denied`).
- `PayoutStage.STAGE_CONVERTING`: a run that is converting is
  `PAYOUT_IN_PROGRESS` at this stage. It comes after `STAGE_PLANNING` and
  before `STAGE_REQUESTING`, although its number is 5.

The field names match `chain.withdraw.v1.AutoWithdrawPreviewConversion` and
`chain.withdraw.v1.AutoWithdrawConversion`; amounts use the broker reads'
`TokenAmount` (base units of 18 decimals) instead of `_e18` integers, and
per-leg detail is not included.

## How clients use it

```http
GET /v1/quickswap/brokers/acme/upcoming-payouts?limit=20
```

```json
{
  "upcomingPayouts": [{
    "vaultAccountId": "1234",
    "status": "WILL_PAY",
    "assets": [
      {"assetId": 2, "payable": {"baseUnits": "4500", "decimals": 18}, "heldBack": {"baseUnits": "0", "decimals": 18}},
      {"assetId": 21, "holdReason": "converts", "payable": {"baseUnits": "0", "decimals": 18}, "heldBack": {"baseUnits": "0", "decimals": 18},
       "conversion": {"payoutAssetId": 2, "expectedAmountOut": {"baseUnits": "2000", "decimals": 18},
                      "minAmountOut": {"baseUnits": "1980", "decimals": 18}, "priceImpactBps": 10, "execution": {"market": {}}}}
    ]
  }]
}
```

ConnectRPC: `/swap.quickswap.v1.QuickSwapBrokerService/ListUpcomingPayouts`,
`/swap.quickswap.v1.QuickSwapBrokerService/ListPayouts`,
`/swap.quickswap.v1.QuickSwapBrokerService/GetPayout` and
`/swap.quickswap.v1.QuickSwapBrokerService/ListVaultPayoutRules`, unchanged
requests.

Every amount is in its own asset, so read them per asset and never add
amounts of different assets together:

- The payout asset's entry is the run's one payout. Its `available` is the
  vault's balance of the payout asset plus every conversion's
  `expected_amount_out`. Its `payable` is what the run is expected to pay:
  `available` after the rule's per-run ceiling, minimums and rounding to
  whole units of the destination network. Its `held_back` is what of it
  would stay in the vault.
- An asset that converts (`hold_reason` `converts`) has `payable` and
  `held_back` zero: its whole balance leaves the vault through the
  conversion, and its `conversion.expected_amount_out` is in the payout
  asset.
- An asset that would not convert keeps its whole balance in the vault as
  its own `held_back`, with `conversion.skip_reason` saying why.

## Migration

None: every field and the enum value are additive. A client that switches
on `PayoutStage` should handle `STAGE_CONVERTING`; before this change such a
run read as `STAGE_PLANNING`.
