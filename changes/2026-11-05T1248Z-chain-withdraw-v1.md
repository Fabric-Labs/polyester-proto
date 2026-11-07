---
services:
  - chain.withdraw.v1.WithdrawService
  - transfer.v1.InternalTransferService
  - auth.v1.PolicyService
kind: changed
breaking: false
wire_compatible: true
---

## What and why

REST withdraw, internal-transfer and subaccount-policy requests now accept
zeros beyond a decimal's scale, as REST order writes already do. Extra
trailing zeros don't change a value: `"1.500000000000000000000"` is the exact
amount `1.5`, so a client that formats amounts with a fixed number of decimals
is no longer refused.

- `amount` on `POST /v1/chain/trading-withdraws`,
  `POST /v1/chain/trading-withdraws:wallet` and
  `POST /v1/trading/internal-transfers` (18 decimals).
- `policy.maxOrderNotional` on `POST /v1/auth/policies/subaccounts`, and
  `maxOrderNotional` on `PATCH /v1/auth/policies/subaccounts/{policy_id}`
  (6 decimals).

A padded decimal carries the same value as its unpadded form: it is checked
and answered exactly as `"1.5"` is, a withdraw's signed payload and a
transfer's idempotency check bind that value rather than its spelling, and
responses keep returning the canonical form without trailing zeros. A request
signed with an API key still signs the body as sent, padding included.

A nonzero digit beyond the scale is still refused, and nothing is rounded or
truncated:

- withdraw and internal transfer: `400 invalid_request`, "invalid amount";
- `POST /v1/auth/policies/subaccounts`: `400 AUTH_INVALID_REQUEST`, "invalid
  maxOrderNotional";
- `PATCH /v1/auth/policies/subaccounts/{policy_id}`:
  `400 AUTH_INVALID_REQUEST`, `patch field "maxOrderNotional" is not a valid
  decimal`.

Before, any digit beyond the scale, even a zero, answered that refusal.

ConnectRPC is unchanged: it carries these values as scaled integers
(`amount_e18`, `max_order_notional`), which REST now produces from either form.

## How clients use it

An internal transfer whose amount carries 21 decimals, the last three zeros:

```bash
curl -sS -X POST https://<api-host>/v1/trading/internal-transfers \
  -H 'Authorization: Bearer <token>' -H 'Content-Type: application/json' \
  -d '{"destinationSubaccountId":"<subaccount-id>","assetId":7,
       "amount":"1.500000000000000000000","idempotencyKey":"t-1"}'
```

It is the transfer `"amount":"1.5"` requests, and its response carries
`"amount":"1.5"`. The ConnectRPC equivalent is
`/transfer.v1.InternalTransferService/CreateInternalTransfer` with
`"amountE18":{"lo":"1500000000000000000"}`.

`"amount":"1.5000000000000000001"` is still refused with `400`.

## Migration

None: every request accepted before is accepted unchanged, and only requests
that were refused now succeed.
