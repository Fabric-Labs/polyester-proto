---
services:
  - notifications.v1.NotificationService
kind: changed
breaking: false
wire_compatible: true
---

## What and why

Every Owner and Finance member of a QuickSwap broker is notified of each
payout address change of the broker's vaults. They hear when a member requests
the change, and again when it is applied or not. A change nobody on the team
asked for is a security signal. An added address becomes payable only 24 hours
after it is listed, so members have time to act.

Added:
- the `NotificationType` value `BROKER_ALLOWLIST_CHANGE`, in category
  `SECURITY`;
- the `Notification.detail` field `broker_allowlist` (`BrokerAllowlistNotice`);
- the enums `BrokerAllowlistEvent` (`CHANGE_REQUESTED`, `CHANGE_APPLIED`,
  `CHANGE_NOT_APPLIED`) and `BrokerAllowlistAction` (`ADD_DESTINATION`,
  `REMOVE_DESTINATION`).

**Availability:** devnet only. Not available on testnet yet; the service isn't deployed or routed there.

## How clients use it

```http
POST /notifications.v1.NotificationService/ListNotifications
{"categories": ["SECURITY"], "readState": "UNREAD"}
```

A `BROKER_ALLOWLIST_CHANGE` notification carries `brokerAllowlist`:
- `broker`: the broker name;
- `vaultAccountId` and `changeId`: the same IDs as
  `swap.quickswap.v1.QuickSwapBrokerService.ListVaultAllowlistChanges`
  returns;
- `event` and `action`;
- `destinationChainId` and `destinationAddress`: the address in canonical
  form;
- `label`;
- `requestedByAccountId`.

For a `CHANGE_NOT_APPLIED` event, read the change with
`ListVaultAllowlistChanges` to see why: its `failure`.
`GetNotificationCounts` lists the new type with its category.

## Migration

None: the additions are new enum values and a new oneof member. A client that
does not know `BROKER_ALLOWLIST_CHANGE` shows the notification's `title` and
`text`.
