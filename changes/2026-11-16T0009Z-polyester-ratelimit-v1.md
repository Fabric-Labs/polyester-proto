---
services:
  - orders.v1.OrdersService
  - triggers.v1.TriggersService
  - auth.v1.AuthService
  - claims.v1.ClaimsService
  - chain.withdraw.v1.WithdrawService
  - changelog.v1.ChangelogService
kind: changed
breaking: true
wire_compatible: false
---

## What and why

REST rate-limit problem bodies now spell the rate-limit detail one way, the
way REST batch item errors and the REST reference already spell it:

- The member is `rateLimit`, not `rate_limit`.
- Its fields are camelCase: `reason`, `operationId`, `policyClass`, `scope`,
  `refillModel`, `limit`, `remaining`, `retryAfterMs` and `policyVersion`
  (previously `operation_id`, `policy_class`, `refill_model`,
  `retry_after_ms` and `policy_version`). Enum values are unchanged.
- `limit`, `remaining`, `retryAfterMs` and `policyVersion` stay decimal
  strings in problem bodies, as before.

This applies to every REST response with code `RATE_LIMIT_EXCEEDED` (HTTP 429)
or `RATE_LIMIT_AUTHORITY_UNAVAILABLE` (HTTP 503), including the gateway's own
HTTP 429 on public REST routes, which carries
`"rateLimit":{"reason":"QUOTA_EXCEEDED","scope":"CLIENT_IP"}`. The REST
reference now documents `rateLimit` on the orders and triggers problem bodies.
ConnectRPC errors are unchanged: they still carry
`polyester.ratelimit.v1.RateLimitDetail` as a typed error detail. The
`Retry-After`, `RateLimit-*` and `X-Polyester-RateLimit-Policy-Version`
headers are unchanged.

## How clients use it

A REST request refused for quota:

```http
POST /v1/orders
Content-Type: application/json
```

```http
HTTP/1.1 429 Too Many Requests
Content-Type: application/problem+json
Retry-After: 2

{
  "type": "https://polyester.dev/problems/rate-limit-exceeded",
  "title": "Rate limit exceeded",
  "status": 429,
  "detail": "Rate limit exceeded",
  "code": "RATE_LIMIT_EXCEEDED",
  "rateLimit": {
    "reason": "QUOTA_EXCEEDED",
    "operationId": "orders.create",
    "policyClass": "TRADING_PLACE",
    "scope": "ACCOUNT",
    "refillModel": "CONTINUOUS",
    "limit": "250",
    "remaining": "0",
    "retryAfterMs": "1500",
    "policyVersion": "7"
  }
}
```

Only `reason` is always present. `operationId`, `policyClass`, `scope` and
`refillModel` are present when the service refused the request; `limit`,
`remaining` and `retryAfterMs` on the service's `RATE_LIMIT_EXCEEDED`;
`policyVersion` when known. The API gateway's own 429 carries only `reason`
and `scope`. Branch on `code`, and wait at least `rateLimit.retryAfterMs` (or
the `Retry-After` header when the body has none) before retrying.

## Migration

1. Read the rate-limit detail of a REST problem body from `rateLimit` instead
   of `rate_limit`.
2. Read its fields by their camelCase names: `operationId`, `policyClass`,
   `refillModel`, `retryAfterMs` and `policyVersion`.
3. Keep parsing `limit`, `remaining`, `retryAfterMs` and `policyVersion` as
   decimal strings.
