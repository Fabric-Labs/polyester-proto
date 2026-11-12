---
services:
  - notifications.v1.NotificationService
kind: changed
breaking: false
wire_compatible: true
---

## What and why

The notification center gains its first security notice: `TELEGRAM_APPROVAL`
(category `SECURITY`), an action approved from the Telegram account linked to
the caller's account. Every approval given from Telegram, cancels and pauses
included, produces one, so the account's owner sees it outside Telegram. If the
caller did not approve it, the linked Telegram account may be in someone
else's hands.

Added: the `NotificationType` value `TELEGRAM_APPROVAL`, the
`NotificationCategory` value `SECURITY`, the `Notification.detail` field
`telegram_approval` (`TelegramApprovalNotice`), the enum
`TelegramApprovalAction` and the message `NoticeField`.

**Availability:** devnet only. Not available on testnet yet; the service isn't deployed or routed there.

## How clients use it

```http
POST /notifications.v1.NotificationService/ListNotifications
{"categories": ["SECURITY"], "readState": "UNREAD"}
```

A `TELEGRAM_APPROVAL` notification carries `telegramApproval`:
`approvalId` (the Assistant API's approval ID), `action` (what was approved),
`accountId` (the trading account it acted on), `summary` and `parameters` (the
approved action in plain words, in display order) and `approvedAt`.
`GetNotificationCounts` now lists both types, each with its category.

## Migration

None: the additions are new enum values and a new oneof member. A client that
does not know `TELEGRAM_APPROVAL` shows the notification's `title` and `text`.
