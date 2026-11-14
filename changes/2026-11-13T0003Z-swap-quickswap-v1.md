---
services:
  - swap.quickswap.v1.QuickSwapBrokerService
kind: changed
breaking: false
wire_compatible: true
---

## What and why

Broker administration calls made with a member's session token are now rate
limited, as calls made with a broker API key already are. Limits apply per
member account, across every broker the member belongs to: changes (vaults,
keys, members, invites, ownership and the payout address allowlist) count
against the account's change quota, and reads against the account's own read
rate. A call over a limit fails exactly as an API key over its rate does:
`RESOURCE_EXHAUSTED` with the `ErrorDetail` code `BROKER_LIMIT` and a
`Retry-After` header in seconds, REST 429. While a member's change quota
cannot be checked, a change fails with `UNAVAILABLE` carrying a
`polyester.ratelimit.v1.RateLimitDetail` (REST: the message
`rate limit authority unavailable`). Both refusals happen before the call runs,
so they change nothing.

Any other `UNAVAILABLE` on a change can leave its outcome unknown, as before:
the change may have been made.

The `BROKER_LIMIT` and `QuickSwapBrokerService` comments say so. No message,
field or enum value changes.

## How clients use it

Nothing new to call. A member's session call such as

```http
POST /swap.quickswap.v1.QuickSwapBrokerService/SetVaultStatus
Authorization: Bearer <session token>
{"broker": "acme", "accountId": "7433029815562201289", "status": "VAULT_DISABLED"}
```

or `POST /v1/quickswap/brokers/acme/vaults/7433029815562201289/status` over
REST can now answer 429 with `Retry-After`:

```json
{"code": "resource_exhausted", "message": "the account exceeded its request rate; retry after Retry-After seconds",
 "details": [{"type": "swap.quickswap.v1.ErrorDetail", "debug": {"code": "BROKER_LIMIT"}}]}
```

Wait the number of seconds in `Retry-After`, then send the same request again.

## Migration

None for clients that already handle `BROKER_LIMIT` with `Retry-After` on API
key calls: handle member session calls the same way. A client that sends many
changes in a burst with a member's session should space them out. Keep reading
the state again before retrying a change that failed with any other
`UNAVAILABLE`.
