---
services:
  - notifications.v1.NotificationService
kind: changed
breaking: false
wire_compatible: true
---

## What and why

Linking a Telegram account to a Polyester account now records a security
notification of its own: the new notification type `TELEGRAM_LINKED`
(category `SECURITY`) with the detail `telegram_link`
(`TelegramLinkNotice`, its `linked_at`). The linked Telegram account receives
the same text in its chat with the Polyester bot, with the next steps, so the
user sees that the link worked. If the user did not link it, they unlink it
and review their sessions.

## How clients use it

`ListNotifications`, `CountNotifications` and the mark calls return and count
it like any other notification, under the `SECURITY` category:

```json
POST /notifications.v1.NotificationService/ListNotifications
{"categories": ["SECURITY"]}
```

A `TELEGRAM_LINKED` notification carries `telegram_link.linked_at`, the time
the user confirmed the link; its `title` is "Telegram linked" and its `text`
the plain-text message.

## Migration

None: a new enum value and a new `oneof detail` member. A client built on the
previous contract shows the notification's title and text, as for any type it
does not know.
