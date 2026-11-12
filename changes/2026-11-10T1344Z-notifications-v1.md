---
services:
  - notifications.v1.NotificationService
kind: added
breaking: false
wire_compatible: true
---

## What and why

A new service, `notifications.v1.NotificationService`, serves the account's
notification center: what happened on the account, newest first, with read
state. It starts with one notification type, `PRICE_ALERT` (category
`ALERTS`): a price alert that fired, for alerts whose channels include
`IN_APP`. New types arrive as new `NotificationType` values with a matching
`Notification.detail` field.

Every method takes a session token only. An API key is refused with
permission denied, so notifications are read and marked on the website and in
the app.

**Availability:** devnet only. Not available on testnet yet; the service isn't deployed or routed there.

## How clients use it

```http
POST /notifications.v1.NotificationService/ListNotifications
{"categories": ["ALERTS"], "readState": "UNREAD", "limit": 25}

POST /notifications.v1.NotificationService/GetNotificationCounts
{}

POST /notifications.v1.NotificationService/MarkNotificationsRead
{"notificationIds": ["1234567890123"]}

POST /notifications.v1.NotificationService/MarkAllNotificationsRead
{"categories": ["ALERTS"], "throughNotificationId": "1234567890123"}
```

- `ListNotifications`: newest first; `limit` defaults to 25, max 100. Optional
  `categories`, `read_state` (unspecified returns read and unread) and
  `start_time` (inclusive) / `end_time` (exclusive). `page_token` is opaque,
  keyset-backed and exclusive, bound to the caller, categories, read state and
  time bounds; `next_page_token` is empty at the end.
- `GetNotification` reads one notification, for deep links.
- `GetNotificationCounts` returns exact totals and unread counts per type,
  with each type's category, every type included.
- `MarkNotificationsRead` / `MarkNotificationsUnread` take 1 to 100 distinct
  IDs and are all-or-nothing: an ID that is not the caller's fails the call
  with `NOTIFICATION_NOT_FOUND` and changes nothing.
- `MarkAllNotificationsRead` marks unread notifications in the categories,
  only up to `through_notification_id` when set, so a notification that
  arrives after the client last listed stays unread.

## Migration

None: the service is new.
