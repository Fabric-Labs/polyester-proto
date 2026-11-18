---
services:
  - swap.quickswap.v1.QuickSwapService
  - swap.quickswap.v1.QuickSwapBrokerService
  - swap.quickswap.v1.QuickSwapExplorerService
kind: changed
breaking: false
wire_compatible: true
---

## What and why

QuickSwap volumes and fees now carry their USD value as of when they
happened, so charts and totals of past periods no longer move with today's
market prices. Each amount is valued once, a few minutes at most after it
happens, at the asset's USD price then, and never repriced:

- a QuickSwap's deposit when it is taken into execution;
- what its destination is paid when that payout is requested;
- each fee when it is charged.

Every USD value is a `TokenAmount` in base units of 18 decimals, rounded
down. A value that is not known is unset, never zero: an amount from the
last few minutes, one no current price valued in time, or one from before
USD values existed. Sums cover the amounts that have a value and say how many
are left out.

New fields:

- `QuickSwap` and `QuickSwapSummary`: `deposited_value_usd`,
  `delivered_value_usd` (set once the DESTINATION withdrawal is FINALIZED).
  Setting a value on a `QuickSwap` increments its `version`.
- `ChargedFee.value_usd`: unset while any fee summed in `amount` has none.
- `AssetVolume.value_usd` and `AssetVolume.unvalued_count`, in
  `BrokerStatsPeriod.deposited` and `delivered` (`GetBrokerStats`) and in
  `QuickSwapStatsTotals.fees` (`GetQuickSwapStats`).
- `FeeSplit.service_fee_usd`, `broker_fee_usd`, `affiliate_fee_usd`,
  `referral_fee_usd` and `unvalued_count`. A fee with a zero amount is worth
  exactly zero.
- `FeeTotal.value_usd` and `FeeTotal.unvalued_count` (`GetFeeTotals`).
- `QuickSwapStatsTotals.source_volume_usd`, `destination_volume_usd`,
  `fees_usd` and `unvalued_count`; `QuickSwapRouteStats.source_volume_usd`,
  `destination_volume_usd` and `unvalued_count` (`GetQuickSwapStats`).

## How clients use it

The fields arrive on the existing calls, over ConnectRPC and REST alike:

```bash
curl -sS https://<api-host>/swap.quickswap.v1.QuickSwapBrokerService/GetBrokerStats \
  -H 'Authorization: Bearer <token>' -H 'Content-Type: application/json' \
  -d '{"broker": "acme", "groupBy": "DAY", "startTime": "2026-11-01T00:00:00Z", "endTime": "2026-11-17T00:00:00Z"}'
curl -sS 'https://<api-host>/v1/quickswap/explorer/stats?window=LAST_7_DAYS'
```

```json
{"assetId": 1, "amount": {"baseUnits": "2000000000000000000", "decimals": 18},
 "valueUsd": {"baseUnits": "120000246913560000000000", "decimals": 18}, "unvaluedCount": "1"}
```

- Chart a period's USD from its stored values (`valueUsd`, `feesUsd`,
  `sourceVolumeUsd`, ...) instead of pricing per-asset amounts at the current
  price. A completed period returns the same USD figure on every read.
- When `unvaluedCount` is above zero, the USD value leaves that many amounts
  out: show it as partial. When `valueUsd` is absent, nothing in the group
  has a value; do not show zero.
- QuickSwaps from before USD values existed have none and stay counted in
  `unvaluedCount`.

## Migration

None: the fields are new and optional. Clients that priced amounts
themselves can switch to the stored values.
