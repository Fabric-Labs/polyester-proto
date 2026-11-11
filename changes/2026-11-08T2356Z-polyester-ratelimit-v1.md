---
services:
  - ai.v1.AssistantService
kind: changed
breaking: false
wire_compatible: true
---

## What and why

`polyester.ratelimit.v1.PolicyClass` gains three values for the trading
assistant's own limits, which `ai.v1.AssistantService` now enforces per
account:

- `ASSISTANT_MESSAGE` (14): `SendMessage`, each message starting an
  assistant run.
- `ASSISTANT_CONTROL` (15): `CancelRun`, `DecideApproval` and `DeleteMemory`.
- `ASSISTANT_READ` (16): the lists and `WatchRun` connections.

Messages have a budget of their own, so a burst of messages never uses up
the budget for cancelling a run, deciding an approval or reading. An account
also holds a bounded number of `WatchRun` streams at once.

## How clients use it

A refused call answers `resource_exhausted` (HTTP 429) with the typed
`polyester.ratelimit.v1.RateLimitDetail` error detail and the headers
`Retry-After`, `RateLimit-Limit`, `RateLimit-Remaining`, `RateLimit-Reset`
and `X-Polyester-RateLimit-Policy-Version`, as other rate-limited calls do:

```json
{
  "code": "resource_exhausted",
  "message": "rate limit exceeded",
  "details": [{ "type": "polyester.ratelimit.v1.RateLimitDetail", "debug": {
    "reason": "QUOTA_EXCEEDED", "limit": "120", "remaining": "0",
    "retryAfterMs": "29481", "policyVersion": "1",
    "operationId": "ai.v1.AssistantService.SendMessage",
    "policyClass": "ASSISTANT_MESSAGE", "scope": "ACCOUNT",
    "refillModel": "CONTINUOUS" } }]
}
```

Wait `retryAfterMs` (or `Retry-After` seconds) before sending again. A
`WatchRun` refused because the account holds too many open streams answers
the same way with `policyClass` `ASSISTANT_READ`; close a stream the client
no longer reads.

## Migration

None: the values are new, and an older client decodes them as unknown enum
values.
