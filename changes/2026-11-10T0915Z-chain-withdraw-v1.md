---
services:
  - chain.withdraw.v1.AutoWithdrawService
  - chain.withdraw.v1.WithdrawService
kind: removed
breaking: true
wire_compatible: true
---

## What and why

Auto-withdraw rules can now be paused and resumed without losing their
destinations, and every `AutoWithdrawService` RPC has a REST route.

- **New RPC `SetAutoWithdrawRuleEnabled`** pauses (`enabled: false`) or
  resumes (`enabled: true`) the account's rule, keeping its schedule and
  destinations. It needs a session login with withdraw rights on the account.
  Pausing needs no step-up. Resuming needs a fresh step-up. It checks every
  destination again, as setting a rule does, and clears a pause the rule set
  itself. A refused destination leaves the rule paused. A rule already in the
  requested state is returned unchanged.
- **Removed RPC `DisableAutoWithdrawRule`** and its request and response
  messages. Its disabled version had no destinations, so pausing and then
  resuming to the same address was not possible.
- **`AutoWithdrawRule`:**
  - New `created_by_account_id`: the public account ID of the person who
    created the version.
  - `enabled: false` now marks a paused version that keeps its destinations.
- **`AutoWithdrawDestination.min_floor_e18`** (responses only): the least
  amount a payout to the destination can pay now. That is the route's
  withdrawal minimum, and more than its current network fee. A new rule's
  `min_amount_e18` must be at least this floor. It is unset (REST omits
  `minFloor`) when the route or its network fee is not known at the moment;
  show no floor then rather than zero.
- **`AutoWithdrawPause`:**
  - `asset_id` is zero when no payout caused the pause.
  - On a broker vault, the rule pauses with `authority_revoked` as soon as its
    author's broker role changes in any way: removed, demoted, or promoted
    (for example Finance to Owner), and also when the author is removed and
    added back. An Owner or Finance member resumes it with a fresh step-up.
- **New `ErrorCode` values,** each with `ErrorDetail.field_path` naming the
  request field, such as `destinations[0].min_amount_e18`:
  - `DESTINATION_INVALID`: the address is not a valid withdrawal destination
    on its chain.
  - `MIN_AMOUNT_BELOW_FLOOR`: below the floor; `ErrorDetail.min_floor_e18`
    carries it.
  - `ROUTE_UNSUPPORTED`: no enabled withdrawal route pays the asset to that
    chain.
  - `VERSION_CONFLICT`: `expected_version` is stale.

  `EXTERNAL_DESTINATION_NOT_WHITELISTED` now also names the field.
- **`ErrorDetail` gains `field_path` and `min_floor_e18`.**

## How clients use it

Pause, then resume to the same destinations. Resuming sends a fresh step-up
proof in `X-Auth-Step-Up`:

```http
POST https://<api-host>/chain.withdraw.v1.AutoWithdrawService/SetAutoWithdrawRuleEnabled
Content-Type: application/json
Authorization: Bearer <session token>

{"subaccountId": "<public subaccount ID>", "expectedVersion": 3, "enabled": false}
```

The REST routes take the same session login, step-up and amounts in whole
units of the asset (decimal strings with at most 18 decimal places). Omit
`subaccountId` for the root account. REST problems carry `field_path` (the
REST field, such as `destinations[0].minAmount`) and, for
`MIN_AMOUNT_BELOW_FLOOR`, `min_floor` in whole units.

| RPC | REST |
| --- | --- |
| `GetAutoWithdrawRule` | `GET /v1/auto-withdraw/rule?subaccountId=` |
| `SetAutoWithdrawRule` | `PUT /v1/auto-withdraw/rule` |
| `SetAutoWithdrawRuleEnabled` | `PUT /v1/auto-withdraw/rule/enabled` |
| `ListAutoWithdrawRuns` | `GET /v1/auto-withdraw/runs?subaccountId=&limit=&pageToken=` |

```http
PUT https://<api-host>/v1/auto-withdraw/rule/enabled
Content-Type: application/json
Authorization: Bearer <session token>
X-Auth-Step-Up: <fresh step-up proof>

{"expectedVersion": 4, "enabled": true}
```

Each pause or resume creates the next rule version, so send the latest
`version` as `expectedVersion`. On `VERSION_CONFLICT`, read the rule again.
Show `min_floor_e18` beside each destination's minimum.

## Migration

1. Replace `DisableAutoWithdrawRule(subaccount_id, expected_version)` with
   `SetAutoWithdrawRuleEnabled(subaccount_id, expected_version, enabled:
   false)`. A pause no longer needs a step-up. It keeps the destinations, so
   resume with `enabled: true` and a fresh step-up instead of setting the rule
   again.
2. Treat a rule as running only while `enabled` is true and `pause` is unset.
   A paused version now lists its destinations.
3. Rule writes now refuse a stale `expected_version` with `VERSION_CONFLICT`
   (it was `CONFLICT`). A minimum below the floor now gets
   `MIN_AMOUNT_BELOW_FLOOR` (it was `AMOUNT_BELOW_MINIMUM`), checked against
   the floor including the network fee. An invalid address now gets
   `DESTINATION_INVALID` (it was `INVALID_DESTINATION_ADDRESS`). An unsupported
   asset or chain now gets `ROUTE_UNSUPPORTED` (it was `UNSUPPORTED_ASSET` or
   `INVALID_DESTINATION_CHAIN`). Map them to the field named in
   `ErrorDetail.field_path`.
