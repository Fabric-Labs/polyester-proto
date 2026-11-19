---
services:
  - notifications.v1.NotificationService
kind: added
breaking: false
wire_compatible: true
---

## What and why

The assistant can now convert dust, balances too small to trade, into one
asset within a trading account, once the user approves it. A conversion
approved from the linked Telegram account is recorded in the notification
center like every other action approved there, so `TelegramApprovalAction`
gains a value for it:

- `CONVERT_DUST = 10`: dust converted into one asset within the trading
  account.

## How clients use it

A `TELEGRAM_APPROVAL` notification's `telegram_approval.action` may now be
`CONVERT_DUST`. Show it like the other actions: its `summary` and
`parameters` already say what was converted, into what, and for how much.

```json
POST /notifications.v1.NotificationService/ListNotifications
{"categories": ["SECURITY"]}

{
  "notifications": [
    {
      "telegramApproval": {
        "approvalId": "123",
        "action": "CONVERT_DUST",
        "accountId": "456",
        "summary": "Convert the dust of subaccount … into USDT: 0.004 DOGE, 0.5 PEPE …",
        "parameters": [{"label": "Receive", "value": "0.000689 USDT"}],
        "approvedAt": "2026-11-17T10:55:00Z"
      }
    }
  ]
}
```

## Migration

None. The value is new; a client that does not know it yet should show the
notice's `summary` and `parameters`, as for any action it does not recognize.
