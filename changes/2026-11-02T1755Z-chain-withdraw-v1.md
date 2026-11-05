---
services:
  - chain.withdraw.v1.WithdrawService
  - chain.withdraw.v1.AutoWithdrawService
  - ledger.read.v1.LedgerReadService
  - chain.guard.v1.GuardSignerService
  - vip.v1.VIPService
  - checkpoints.v1.CheckpointService
  - auth.v1.SubaccountRoleService
kind: changed
breaking: true
wire_compatible: true
---

## What and why

Every method that accepts only a session token and an account's credentials
now refuses an account API key the same way: HTTP 403, Connect `permission_denied` with an `auth.v1.AuthErrorDetail`
whose code is `AUTH_SESSION_KIND_NOT_ALLOWED`, and REST
`application/problem+json` with `"code": "AUTH_SESSION_KIND_NOT_ALLOWED"`.
The detail reads `interactive session required`. A session token that
carries an API key is refused the same way. The reference marks these methods
"Session token". The QuickSwap broker methods keep their own credentials
(broker API keys and member sessions): an account API key is no credential
there and is answered as unauthenticated.

Before, most session-only methods already answered this way, but a few used
their service's generic code:

- `WithdrawService.CreateWalletTradingWithdraw` (REST
  `POST /v1/chain/trading-withdraws:wallet`),
  `AutoWithdrawService.SetAutoWithdrawRule` and
  `AutoWithdrawService.DisableAutoWithdrawRule` answered `PERMISSION_DENIED`
  with a withdraw `ErrorDetail`.
- `LedgerReadService.GetPortfolioEquitySnapshot` and
  `LedgerReadService.GetPortfolioEquityHistorySeries` (REST
  `GET /v1/equity/portfolio/snapshot` and
  `GET /v1/equity/portfolio/history/series`) answered `PERMISSION_DENIED` with
  a ledger read `ErrorDetail`.

`GuardSignerService`'s session-only methods, `VIPService.GetVIPStatus`,
`CheckpointService.ListMyCheckpoints` and
`SubaccountRoleService.GetEffectiveSubaccountPermissions` already answered
`AUTH_SESSION_KIND_NOT_ALLOWED`; only their detail text changes, to
`interactive session required`.

HTTP statuses do not change. 401 stays reserved for a missing or invalid
credential; an authenticated API key on a session-only method is a 403.

## How clients use it

Call session-only methods with a session token. A request signed with an API
key gets, on REST:

```json
{
  "type": "https://polyester.dev/problems/auth-session-kind-not-allowed",
  "title": "Interactive Session Required",
  "status": 403,
  "detail": "interactive session required",
  "code": "AUTH_SESSION_KIND_NOT_ALLOWED"
}
```

and on ConnectRPC (for example
`POST https://<api-host>/chain.withdraw.v1.WithdrawService/CreateWalletTradingWithdraw`):

```json
{
  "code": "permission_denied",
  "message": "interactive session required",
  "details": [{"type": "auth.v1.AuthErrorDetail",
               "debug": {"code": "AUTH_SESSION_KIND_NOT_ALLOWED", "message": "interactive session required"}}]
}
```

## Migration

1. If you match the refusal of an API key on the withdraw, auto-withdraw rule
   or portfolio equity methods listed above, match
   `AUTH_SESSION_KIND_NOT_ALLOWED` instead of `PERMISSION_DENIED`: in the
   REST `code`, or in the Connect `auth.v1.AuthErrorDetail` instead of the
   withdraw or ledger read `ErrorDetail`.
2. If you match refusal detail text from `GetVIPStatus`, `ListMyCheckpoints`,
   `GetEffectiveSubaccountPermissions` or guard signer methods, match the code
   instead.

Clients that call these methods with a session token see no change.
