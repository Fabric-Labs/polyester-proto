---
services:
  - chain.withdraw.v1.AutoWithdrawService
kind: added
breaking: false
wire_compatible: true
---

## What and why

An auto-withdraw rule can now pay out in one chosen asset. Each run first converts every other asset on the account into it through a Swap placed from the account, then pays that asset to the rule's destination. A conversion runs as TWAP when its USD value exceeds 10,000 or its quoted price impact exceeds 0.5%, otherwise at market. Whatever a conversion does not spend stays on the account for the next run.

- `AutoWithdrawRule` gains `payout_asset_id` (10) and `conversion_twap_duration_ms` (11).
- `SetAutoWithdrawRuleRequest` gains `payout_asset_id` (6) and `conversion_twap_duration_ms` (7).
- `AutoWithdrawRunState` gains `RUN_CONVERTING` (7): the run's conversions are executing, and payouts start once every conversion is final and settled.
- `AutoWithdrawPreviewAsset` gains `conversion` (14, `AutoWithdrawPreviewConversion`), and `hold_reason` gains `converts`: a payout-asset rule's preview shows each asset it would convert with its current quote (expected and least output, price impact, market or TWAP) or why it would not convert (`skip_reason`), and sizes the payout asset's entry from its balance plus the expected outputs.
- `AutoWithdrawRun` gains `conversions` (9), one `AutoWithdrawConversion` per converted asset: the source asset and amount, the Swap's `swap_id` (readable with `swap.v1.SwapService/GetSwap`), its `execution` (market or TWAP), `status`, `reason`, `is_terminal`, the executed, unspent and received amounts, its `legs` (`swap.v1.SwapLeg`), and a `skip_reason` when the asset was not converted.

## How clients use it

Set a rule that pays everything in one asset. It has exactly one destination, for that asset:

```http
POST /chain.withdraw.v1.AutoWithdrawService/SetAutoWithdrawRule
{
  "expectedVersion": 0,
  "schedule": {"cadence": "AUTO_WITHDRAW_CADENCE_DAILY", "minuteOfDay": 540},
  "destinations": [{"assetId": 2, "destinationChainId": "1", "destinationAddress": "0x…",
                    "minAmountE18": {"lo": "1000000000000000000"}, "maxAmountE18": {"lo": "…"}}],
  "payoutAssetId": 2,
  "conversionTwapDurationMs": "600000"
}
```

REST: `PUT /v1/auto-withdraw/rule` with `payoutAssetId` and `conversionTwapDurationMs` (a decimal string).

- `payout_asset_id` 0 pays every asset in kind, as before.
- `conversion_twap_duration_ms` runs from 60000 to 3600000; 0 means 300000.
- A payout asset without exactly one destination, for that asset, is `INVALID_ARGUMENT`.
- Creating or resuming a rule with a payout asset also needs the permission to swap on the account (`PERMISSION_DENIED` without it); on a broker vault that is the permission to manage its payouts, held by the Owner and Finance roles. Pausing does not. A run whose author has lost it leaves the assets in kind with `policy_denied`.
- `GetAutoWithdrawRule` and the responses return both fields. `conversion_twap_duration_ms` is 0 on an in-kind rule.
- A payout step can now fail with reason `withdrawals_disabled`: the environment does not allow withdrawals, and the funds stay on the account.
- `ListAutoWithdrawRuns` can return `RUN_CONVERTING`, and lists each run's `conversions` ordered by `source_asset_id`. A conversion still executing shows `EXECUTING`. A skipped one (`route_unsupported`, `below_minimum`, `above_maximum`, `conversion_unavailable`, `price_unavailable`, `conversion_refused`, `rule_superseded` or `policy_denied`) is terminal, spends nothing and has no `swap_id`, `status` or `reason`; a completed Swap that ended normally has no `reason` either.

Preview a payout-asset rule with `GET /v1/auto-withdraw/preview` (or `/chain.withdraw.v1.AutoWithdrawService/PreviewAutoWithdrawRun`): each converted asset has `payable` `0`, `holdReason` `converts` and a `conversion`; its `heldBack` is its whole balance with the skip reason instead when it would not convert. The status is never `NO_DESTINATION` for such a rule. The preview decides each conversion as the run would: `execution` is TWAP when the run would use TWAP, in fewer, longer slices (never fewer than two) when the route's minimum requires it, and `below_minimum` otherwise. Its expected and least outputs are estimated from the whole amount at the current book depth.

## Migration

None: the fields are new and optional, and existing rules keep paying in kind. Clients that switch on the run state should show `RUN_CONVERTING` as in progress.
