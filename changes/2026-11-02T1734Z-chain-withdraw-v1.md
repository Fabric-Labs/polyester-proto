---
services:
  - chain.withdraw.v1.AutoWithdrawService
  - chain.withdraw.v1.WithdrawService
kind: changed
breaking: false
wire_compatible: true
---

## What and why

The auto-withdraw contract now states what an API key needs to read rules
and runs. `GetAutoWithdrawRule` and `ListAutoWithdrawRuns` answer an API key
only when its policy grants `READ_AUTO_WITHDRAW` (or `MANAGE_AUTO_WITHDRAW`,
which includes it), and only for the key's own account: a root-account key
reads the root account, and a subaccount key reads its subaccount. This is
how every other API-key read works, for example `READ_BALANCES` for balances.
The contract used to say only that API keys can read rules and runs, so a
key without the action looked refused for no reason.

A refusal now says which policy action is missing. When an API key's policy
lacks the action a withdraw-service call checks, the answer stays
`PERMISSION_DENIED` and its message names the action, for example
"api key policy does not allow READ_AUTO_WITHDRAW". Before, the message was
"forbidden". This covers the auto-withdraw reads and the API-key withdrawal
calls, which check `EXTERNAL_WITHDRAW` or `INTERNAL_TRANSFER`.

Writes are unchanged: `SetAutoWithdrawRule` and `DisableAutoWithdrawRule`
still need a signed-in person with withdraw rights and a fresh step-up, and
are refused to every API key whatever its policy.

## How clients use it

Grant `READ_AUTO_WITHDRAW` in the key's API-key policy, then call with the
key's signed headers. Omit `subaccountId` to read the key's own account. A
subaccount key may also name its own subaccount.

```bash
curl -sS https://<api-host>/chain.withdraw.v1.AutoWithdrawService/GetAutoWithdrawRule \
  -H 'Content-Type: application/json' \
  -H 'X-API-KEY-ID: <key id>' -H 'X-API-TIMESTAMP: <ms>' \
  -H 'X-API-NONCE: <nonce>' -H 'X-API-SIGNATURE: <signature>' \
  -d '{}'
# {"rule": {...}}, or {} when the account has no rule.

curl -sS https://<api-host>/chain.withdraw.v1.AutoWithdrawService/ListAutoWithdrawRuns \
  -H 'Content-Type: application/json' \
  -H 'X-API-KEY-ID: <key id>' -H 'X-API-TIMESTAMP: <ms>' \
  -H 'X-API-NONCE: <nonce>' -H 'X-API-SIGNATURE: <signature>' \
  -d '{"limit": 20}'
```

A key whose policy lacks the action gets HTTP 403:

```json
{"code": "permission_denied",
 "message": "api key policy does not allow READ_AUTO_WITHDRAW",
 "details": [{"type": "chain.withdraw.v1.ErrorDetail", "debug": {"code": "ERROR_CODE_PERMISSION_DENIED"}}]}
```

The REST withdrawal routes (`POST /v1/chain/withdraws:validate-destination`,
`POST /v1/chain/trading-withdraws`) answer the same refusal as
`application/problem+json` with `"code": "PERMISSION_DENIED"` and the same
text in `detail`. The auto-withdraw methods have no REST routes.

A key that names an account other than its own, such as a root-account key
naming a subaccount, is refused with `PERMISSION_DENIED` and the message
"api key target scope mismatch", whatever its policy.

## Migration

None for code. A key that must read auto-withdraw needs `READ_AUTO_WITHDRAW`
in its policy; add it to the key's policy where it is missing. Clients that
matched the refusal message "forbidden" should match the
`PERMISSION_DENIED` code instead.
