---
services:
  - auth.v1.AddressBookService
  - auth.v1.ApiKeyService
  - auth.v1.AuthService
  - auth.v1.MFAService
  - auth.v1.PolicyService
  - auth.v1.ProfileService
  - auth.v1.ResolveService
  - auth.v1.SocialVerificationService
  - auth.v1.SubaccountRoleService
  - auth.v1.SubaccountService
  - auth.v1.SubaccountViewService
  - chain.analytics.v1.ChainAnalyticsService
  - chain.deposit.v1.DepositAddressService
  - chain.lifecycle.v1.LifecycleReadService
  - chain.withdraw.v1.WithdrawService
  - chain.zipper.v1.ZipperService
  - changelog.v1.ChangelogService
  - checkpoints.v1.CheckpointService
  - claims.v1.ClaimsService
  - fees.v1.FeeService
  - marketoverview.v1.MarketOverviewService
  - ratelimit.v1.RateLimitService
  - rewards.v1.RewardCampaignService
  - system.v1.SystemService
  - transfer.v1.InternalTransferService
  - triggers.v1.TriggersService
  - vip.v1.VIPService
kind: changed
breaking: true
wire_compatible: true
---

## What and why

The published OpenAPI document `openapi/polyester.openapi.json` is now the
ConnectRPC reference only: every public unary RPC under its ConnectRPC path,
`POST /<package>.<Service>/<Method>`, with ConnectRPC's JSON (prefixed enum
values such as `PARTICIPATION_OWNER`, ProtoJSON integers).

Until now, an RPC with a REST route appeared in that document only under its
REST path, for example `GET /v1/chain/flows`, but with ConnectRPC's schemas.
It showed REST callers enum values REST refuses, and it had no ConnectRPC
path for those RPCs. 118 ConnectRPC operations replace those 94 REST paths;
the other 91 operations and every component schema, required and read-only fields included, are unchanged.

The REST reference is the per-service documents under `docs/openapi/`, which
use REST's values (`OWNER`, `ALL`, `NEWEST`) and do not change. Nothing
changes on the wire.

## How clients use it

Generate or read ConnectRPC clients from `openapi/polyester.openapi.json`:

```http
POST /chain.lifecycle.v1.LifecycleReadService/ListFlows
{"participation": "PARTICIPATION_OWNER", "limit": 50}
```

Read REST from `docs/openapi/<Service>.json`:

```http
GET /v1/chain/flows?participation=OWNER&limit=50
```

## Migration

Breaking for artifact consumers only. API clients are unaffected, but any tool
reading REST paths from `openapi/polyester.openapi.json` must switch to the
per-service REST specs under `docs/openapi/<Service>.json`.
