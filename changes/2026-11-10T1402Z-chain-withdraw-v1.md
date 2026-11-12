---
services:
  - chain.withdraw.v1.AutoWithdrawService
kind: added
breaking: false
wire_compatible: true
---

## What and why

Accounts can now see what their next auto-withdraw run would do before it
runs, computed by the same planning the run uses.

- **New RPC `PreviewAutoWithdrawRun(subaccount_id)`** returns:
  - `status` (new enum `AutoWithdrawPreviewStatus`): `WILL_PAY`,
    `ROLLS_OVER` (nothing reaches its minimums, or a USD minimum has no
    current price; the balance stays for a later run), `NO_DESTINATION`
    (every positive balance is in an asset the rule pays nowhere), `PAUSED`
    or `NO_RULE`;
  - `next_run_at`: the next UTC slot (for an hourly rule, the next hourly
    check), or the slot that is due now and waiting to start; unset without
    a rule or while it is paused;
  - `rule_version`, the rule's self-`pause` (unset when a person paused it),
    `min_usd_value_e18`, and `computed_at`;
  - `assets` (new message `AutoWithdrawPreviewAsset`), one per destination
    and per other asset with a positive balance, ordered by `asset_id`:
    `available_e18`, `payable_e18`, `held_back_e18` (always `available_e18 -
    payable_e18`), `hold_reason` (`below_minimum`, `price_unavailable`,
    `no_destination`, `above_max`, `dust`, or empty), `minimum_e18`,
    `value_usd_e18` with `priced_at`, and `network_fee_e18` with
    `network_fee_sampled_at`.
- `network_fee_e18` is the network fee a payout to the destination is
  expected to be charged now, taken out of the payout: the same latest fee
  `min_floor_e18` stays above.
- Session logins and API keys with `READ_AUTO_WITHDRAW` can read it, with the
  same account rules as `GetAutoWithdrawRule`.

## How clients use it

```http
POST https://<api-host>/chain.withdraw.v1.AutoWithdrawService/PreviewAutoWithdrawRun
Content-Type: application/json
Authorization: Bearer <session token>

{"subaccountId": "<public subaccount ID>"}
```

REST: `GET /v1/auto-withdraw/preview?subaccountId=` (omit `subaccountId` for
the root account). Amounts are decimal strings in whole units of the asset or
of USD, times are RFC 3339 UTC:

```json
{
  "status": "WILL_PAY",
  "nextRunAt": "2026-11-11T09:00:00Z",
  "ruleVersion": 3,
  "minUsdValue": "25",
  "computedAt": "2026-11-10T14:02:11Z",
  "assets": [
    {"assetId": 7, "destinationChainId": "8453", "destinationAddress": "0x2222…",
     "available": "512.5", "payable": "500", "heldBack": "12.5", "holdReason": "above_max",
     "minimum": "50", "valueUsd": "499.95", "pricedAt": "2026-11-10T14:02:10Z",
     "networkFee": "0.4", "networkFeeSampledAt": "2026-11-10T14:00:00Z"}
  ]
}
```

The preview is the account's state now: balances, prices and fees can change
before `nextRunAt`. A price that is not current, or a price read that fails,
leaves `valueUsd`/`pricedAt` absent rather than failing the call, unless the
rule has a USD minimum: prices then decide what pays, so a failed price read
answers `UNAVAILABLE` and can be retried.

## Migration

None: the RPC, its messages and the enum are new.
