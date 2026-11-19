---
services:
  - alerts.v1.PriceAlertService
  - auth.v1.AccountActivityService
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
  - chain.guard.v1.GuardSignerService
  - chain.lifecycle.v1.LifecycleReadService
  - chain.withdraw.v1.AutoWithdrawService
  - chain.withdraw.v1.WithdrawService
  - chain.zipper.v1.ZipperService
  - changelog.v1.ChangelogService
  - checkpoints.v1.CheckpointService
  - claims.v1.ClaimsService
  - fees.v1.FeeService
  - ledger.read.v1.LedgerReadService
  - ledger.read.v1.StatementExportService
  - marketdata.v1.HeatmapService
  - marketdata.v1.MarketDataService
  - marketoverview.v1.MarketOverviewService
  - notifications.v1.NotificationService
  - orderbook.v1.OrderbookService
  - orders.v1.OrdersReadService
  - orders.v1.OrdersService
  - ratelimit.v1.RateLimitService
  - rewards.v1.RewardCampaignService
  - swap.quickswap.v1.QuickSwapBrokerService
  - swap.quickswap.v1.QuickSwapExplorerService
  - swap.quickswap.v1.QuickSwapReferralService
  - swap.quickswap.v1.QuickSwapService
  - swap.v1.SwapService
  - system.v1.SystemService
  - transfer.v1.DustConversionService
  - transfer.v1.InternalTransferService
  - triggers.v1.TriggersService
  - vip.v1.VIPService
kind: changed
breaking: false
wire_compatible: true
---

## What and why

A ConnectRPC, gRPC or gRPC-web call whose request is over the API's size
limit now carries a typed error detail, `polyester.api.v1.RequestErrorDetail`,
with `code` `REQUEST_TOO_LARGE` and `limit_bytes`, the limit it is over. The
ConnectRPC code stays `resource_exhausted`.

Until now these refusals carried only the message, which differed by how the
request was sent, so a client that does not read message text could not tell
a request that is too large from a rate limit, and retried a request that can
never succeed. The detail is the same on every API, whether the request is
over the limit on the wire, once decompressed, or in the URL of a ConnectRPC
GET call, and whatever the credential. The message is now always
`request too large`.

REST is unchanged: HTTP 413 with the problem `code` `REQUEST_TOO_LARGE`.

`RequestErrorDetail` and its `RequestErrorCode` enum are new, in
`polyester/api/v1/request_error.proto`. No other message, field or enum value
changed.

## How clients use it

```http
POST /changelog.v1.ChangelogService/ListPosts
Content-Type: application/json
Connect-Protocol-Version: 1

HTTP/1.1 429 Too Many Requests

{"code":"resource_exhausted","message":"request too large","details":[{"type":"polyester.api.v1.RequestErrorDetail","value":"CAEQgIAE","debug":{"code":"REQUEST_TOO_LARGE","limitBytes":"65536"}}]}
```

A gRPC or gRPC-web call gets status 8 (`RESOURCE_EXHAUSTED`) with the same
detail in `grpc-status-details-bin`.

ConnectRPC answers `resource_exhausted` with HTTP 429 by its own protocol
mapping; branch on the detail, not the status. `limit_bytes` is the limit of
the API that refused the request; limits differ between APIs and procedures.
A rate limit carries `polyester.ratelimit.v1.RateLimitDetail` instead, and
never `RequestErrorDetail`.

## Migration

None required: the code and status are unchanged, and the detail is new.
Clients that retry `resource_exhausted` should stop retrying a refusal that
carries `RequestErrorDetail` with `REQUEST_TOO_LARGE`, and send a smaller
request instead.
