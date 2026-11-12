---
services:
  - chain.withdraw.v1.AutoWithdrawService
kind: added
breaking: false
wire_compatible: true
---

## What and why

Auto-withdraw run history now shows the network fee each payout was charged,
next to the amount sent, so a completed payout shows what the destination
actually received.

- **New field `AutoWithdrawStep.network_fee_e18`** (`ListAutoWithdrawRuns`):
  the network fee charged on the step's withdrawal, in 18-decimal asset
  units, recorded once when the withdrawal completes. It is taken out of
  `amount_e18`: the destination received `amount_e18 - network_fee_e18`.
- It is unset until the step completes, on failed and skipped steps, and when
  the fee was not reported. It is never zero for an unknown fee.
- `PreviewAutoWithdrawRun`'s `network_fee_e18` is the fee a payout is expected
  to be charged; this field is the fee a past payout was charged.

## How clients use it

```http
POST https://<api-host>/chain.withdraw.v1.AutoWithdrawService/ListAutoWithdrawRuns
Content-Type: application/json
Authorization: Bearer <session token>

{"subaccountId": "<public subaccount ID>", "limit": 20}
```

REST: `GET /v1/auto-withdraw/runs`. Each step carries `networkFee`, a decimal
string in whole units of the asset, absent when unset:

```json
{"assetId": 7, "destinationChainId": "8453", "destinationAddress": "0x2222…",
 "available": "512.5", "amount": "500", "state": "COMPLETED",
 "txHash": "0xabc…", "networkFee": "0.4"}
```

## Migration

None: the field is new and optional.
