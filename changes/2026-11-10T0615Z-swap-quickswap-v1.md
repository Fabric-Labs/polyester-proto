---
services:
  - swap.quickswap.v1.QuickSwapBrokerService
kind: changed
breaking: false
wire_compatible: true
---

## What and why

Documentation only: no message, field, enum value or RPC changes. The
`BrokerRole` comments now state who may move a broker's funds out and change
its payout settings: only the Owner and Finance. Admin keeps everything else
except changing the Owner and transferring ownership.

- `BrokerRole.OWNER`: everything, including withdrawals, payout settings and
  transferring ownership.
- `BrokerRole.ADMIN`: everything except withdrawals, payout settings,
  changing the Owner and transferring ownership.

The broker roles now also reach the broker's vaults (its default vault and
its affiliate vaults) for auto-withdraw rules: an Owner or Finance member
creates, changes and disables a vault's rule, with a fresh step-up, and every
broker role reads it. Payouts of a rule stop when its author leaves the
broker or no longer holds Owner or Finance, until a member with that role
writes a new rule version.

## How clients use it

Read the caller's role as before:

```http
POST https://<api-host>/swap.quickswap.v1.QuickSwapBrokerService/GetBroker
Content-Type: application/json

{"broker": "<broker name>"}
```

REST: `GET https://<api-host>/v1/quickswap/brokers/{broker}`.

Offer withdrawal and payout-setting actions only to `OWNER` and `FINANCE`;
hide or disable them for `ADMIN`, `AFFILIATE_MANAGER` and `VIEWER`. Every
role may view payout settings.

## Migration

None for code: no field or behavior of `QuickSwapBrokerService` changes, and
no broker RPC withdraws or changes payout settings yet. Clients that offered
those actions to `ADMIN` should stop offering them.
