---
services:
  - chain.withdraw.v1.AutoWithdrawService
  - chain.withdraw.v1.WithdrawService
kind: changed
breaking: false
wire_compatible: true
---

## What and why

A broker vault now pays an external destination only once it has been on the
vault's external withdraw whitelist for 24 hours. The vault's whitelist
changes are signed by the platform for the broker, so the delay gives the
broker's Owner and Finance members time to react to an address they did not
expect. Ordinary accounts and sub-accounts are unchanged: they pay a listed
destination at once.

- **New `ErrorCode` `EXTERNAL_DESTINATION_COOLING_DOWN`** (`FAILED_PRECONDITION`,
  REST `400`): a withdrawal from a broker vault to a destination added to its
  whitelist less than 24 hours ago. Nothing is reserved.
- **`ErrorDetail.payable_from`** (new): with that code, when the destination
  can be paid; unset when that time is not known yet.
- **`AutoWithdrawPreviewAsset.payable_from`** (new): on a broker vault, while
  the destination is cooling down at the next run, when it can be paid. The
  asset then holds with `hold_reason` `destination_cooling_down`, pays `0`, and
  the preview `status` is `ROLLS_OVER` unless another asset pays.
- **Run steps:** a payout to a destination still cooling down is a skipped step
  with `reason` `destination_cooling_down`. Its funds stay on the vault, the
  rule keeps running, and the first run after the cool-down pays them. It
  never pauses the rule.

## How clients use it

Read the preview to show when a new address becomes usable:

```http
GET /v1/auto-withdraw/preview?subaccountId=<vault public ID>
```

```json
{
  "status": "ROLLS_OVER",
  "assets": [
    {
      "assetId": 7,
      "payable": "0",
      "heldBack": "125.5",
      "holdReason": "destination_cooling_down",
      "payableFrom": "2026-11-11T14:00:00Z"
    }
  ]
}
```

A withdrawal from the vault to such an address answers:

```json
{
  "code": "EXTERNAL_DESTINATION_COOLING_DOWN",
  "status": 400,
  "payable_from": "2026-11-11T14:00:00Z"
}
```

Over ConnectRPC the same refusal is `failed_precondition` with
`chain.withdraw.v1.ErrorDetail{code: ERROR_CODE_EXTERNAL_DESTINATION_COOLING_DOWN, payable_from}`.

## Migration

None: every change is a new enum value or a new optional field. A client that
does not know `EXTERNAL_DESTINATION_COOLING_DOWN` sees a `400` /
`failed_precondition` refusal, as before for any withdrawal it cannot make.
