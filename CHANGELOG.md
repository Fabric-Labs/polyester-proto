# Changelog

Public contract publications to `develop`, newest first. Each entry lists
the change notes it published; read them under `changes/`.

## 2026-11-10 — develop-20261110T055512Z

API keys read account data with granular permissions, internal transfer lookup with flow IDs, network memo requirements, broker vault balances, cross-device reward acknowledgements and path and body ID mismatch errors.

- [2026-11-08T2129Z-orders-v1](changes/2026-11-08T2129Z-orders-v1.md): changed `orders.v1.OrdersService`, `triggers.v1.TriggersService`. **Breaking.**
- [2026-11-08T2135Z-transfer-v1](changes/2026-11-08T2135Z-transfer-v1.md): changed `transfer.v1.InternalTransferService`, `chain.withdraw.v1.WithdrawService`, `claims.v1.ClaimsService`, `ledger.read.v1.LedgerReadService`, `orders.v1.OrdersReadService`, `auth.v1.PolicyService`.
- [2026-11-08T2157Z-claims-v1](changes/2026-11-08T2157Z-claims-v1.md): changed `claims.v1.ClaimsService`.
- [2026-11-08T2222Z-orders-v1](changes/2026-11-08T2222Z-orders-v1.md): changed `orders.v1.OrdersService`, `triggers.v1.TriggersService`. **Breaking.**
- [2026-11-08T2232Z-marketdata-v1](changes/2026-11-08T2232Z-marketdata-v1.md): changed `marketdata.v1.HeatmapService`.
- [2026-11-08T2245Z-auth-v1](changes/2026-11-08T2245Z-auth-v1.md): changed `auth.v1.MFAService`, `auth.v1.SubaccountService`, `auth.v1.PolicyService`, `auth.v1.AddressBookService`. **Breaking.**
- [2026-11-08T2305Z-auth-v1](changes/2026-11-08T2305Z-auth-v1.md): changed `auth.v1.PolicyService`.
- [2026-11-09T0345Z-orders-v1](changes/2026-11-09T0345Z-orders-v1.md): changed `orders.v1.OrdersService`.
- [2026-11-09T0958Z-ledger-v1](changes/2026-11-09T0958Z-ledger-v1.md): changed `ledger.read.v1.LedgerReadService`, `orders.v1.OrdersReadService`. **Breaking.**
- [2026-11-09T1438Z-rewards-v1](changes/2026-11-09T1438Z-rewards-v1.md): added `rewards.v1.RewardCampaignService`.
- [2026-11-09T1448Z-auth-v1](changes/2026-11-09T1448Z-auth-v1.md): changed `auth.v1.ApiKeyService`, `auth.v1.PolicyService`, `auth.v1.SubaccountRoleService`, `auth.v1.SocialVerificationService`, `vip.v1.VIPService`.
- [2026-11-09T1454Z-transfer-v1](changes/2026-11-09T1454Z-transfer-v1.md): added `transfer.v1.InternalTransferService`.
- [2026-11-09T1511Z-chain-zipper-v1](changes/2026-11-09T1511Z-chain-zipper-v1.md): added `chain.zipper.v1.ZipperService`, `swap.quickswap.v1.QuickSwapService`.
- [2026-11-09T1632Z-swap-quickswap-v1](changes/2026-11-09T1632Z-swap-quickswap-v1.md): added `swap.quickswap.v1.QuickSwapBrokerService`.
- [2026-11-09T1817Z-auth-v1](changes/2026-11-09T1817Z-auth-v1.md): changed `auth.v1.PolicyService`, `auth.v1.ProfileService`, `auth.v1.ResolveService`, `chain.withdraw.v1.WithdrawService`, `claims.v1.ClaimsService`, `checkpoints.v1.CheckpointService`, `rewards.v1.RewardCampaignService`. **Breaking.**
- [2026-11-09T2102Z-rewards-v1](changes/2026-11-09T2102Z-rewards-v1.md): changed `rewards.v1.RewardCampaignService`. **Breaking.**
- [2026-11-10T0200Z-swap-quickswap-v1](changes/2026-11-10T0200Z-swap-quickswap-v1.md): changed `swap.quickswap.v1.QuickSwapService`.

## 2026-11-08 — develop-20261108T161328Z

Received transfers in account flow lists, a trading WebSocket cancel-all-after command, per-request receive windows and per-key replay capacity.

- [2026-11-07T1512Z-chain-lifecycle-v1](changes/2026-11-07T1512Z-chain-lifecycle-v1.md): changed `chain.lifecycle.v1.LifecycleReadService`. **Breaking.**
- [2026-11-07T2151Z-tradews-v1](changes/2026-11-07T2151Z-tradews-v1.md): added `orders.v1.OrdersService`.
- [2026-11-07T2154Z-orders-v1](changes/2026-11-07T2154Z-orders-v1.md): changed `orders.v1.OrdersService`, `triggers.v1.TriggersService`, `auth.v1.AuthService`.
- [2026-11-08T0007Z-triggers-v1](changes/2026-11-08T0007Z-triggers-v1.md): changed `triggers.v1.TriggersService`, `orders.v1.OrdersService`, `orders.v1.OrdersReadService`.
- [2026-11-08T0338Z-auth-v1](changes/2026-11-08T0338Z-auth-v1.md): changed `auth.v1.AuthService`, `auth.v1.ApiKeyService`, `orders.v1.OrdersService`, `triggers.v1.TriggersService`, `chain.deposit.v1.DepositAddressService`, `chain.withdraw.v1.WithdrawService`, `chain.withdraw.v1.AutoWithdrawService`, `chain.guard.v1.GuardSignerService`, `swap.v1.SwapService`, `transfer.v1.InternalTransferService`.

## 2026-11-07 — develop-20261107T200655Z

Uniform request-size and rate-limit problems, editable changelog posts, QuickSwap small-amount outcomes and a wallet-already-linked sign-in error.

- [2026-11-04T1214Z-chain-deposit-v1](changes/2026-11-04T1214Z-chain-deposit-v1.md): changed `chain.deposit.v1.DepositAddressService`, `chain.withdraw.v1.WithdrawService`, `chain.guard.v1.GuardSignerService`, `swap.v1.SwapService`, `swap.quickswap.v1.QuickSwapService`, `transfer.v1.InternalTransferService`, `auth.v1.AuthService`, `auth.v1.ApiKeyService`, `auth.v1.SubaccountService`, `auth.v1.SubaccountViewService`, `auth.v1.PolicyService`, `auth.v1.AddressBookService`, `fees.v1.FeeService`, `checkpoints.v1.CheckpointService`. **Breaking.**
- [2026-11-04T1630Z-ledger-read-v1](changes/2026-11-04T1630Z-ledger-read-v1.md): changed `ledger.read.v1.LedgerReadService`. **Breaking.**
- [2026-11-04T1633Z-triggers-v1](changes/2026-11-04T1633Z-triggers-v1.md): changed `triggers.v1.TriggersService`.
- [2026-11-05T1045Z-auth-v1](changes/2026-11-05T1045Z-auth-v1.md): changed `auth.v1.AuthService`, `auth.v1.ApiKeyService`, `auth.v1.AddressBookService`, `auth.v1.MFAService`, `auth.v1.PolicyService`, `auth.v1.ProfileService`, `auth.v1.SocialVerificationService`, `auth.v1.SubaccountService`. **Breaking.**
- [2026-11-05T1248Z-chain-withdraw-v1](changes/2026-11-05T1248Z-chain-withdraw-v1.md): changed `chain.withdraw.v1.WithdrawService`, `transfer.v1.InternalTransferService`, `auth.v1.PolicyService`.
- [2026-11-05T1252Z-auth-v1](changes/2026-11-05T1252Z-auth-v1.md): changed `auth.v1.AuthService`, `auth.v1.SubaccountRoleService`, `vip.v1.VIPService`, `ratelimit.v1.RateLimitService`, `changelog.v1.ChangelogService`. **Breaking.**
- [2026-11-05T1327Z-orders-v1](changes/2026-11-05T1327Z-orders-v1.md): changed `orders.v1.OrdersReadService`, `orders.v1.OrdersService`.
- [2026-11-05T1425Z-orders-v1](changes/2026-11-05T1425Z-orders-v1.md): changed `orders.v1.OrdersService`, `triggers.v1.TriggersService`, `auth.v1.AuthService`, `auth.v1.SubaccountRoleService`, `vip.v1.VIPService`, `ratelimit.v1.RateLimitService`, `changelog.v1.ChangelogService`. **Breaking.**
- [2026-11-05T1442Z-swap-quickswap-v1](changes/2026-11-05T1442Z-swap-quickswap-v1.md): changed `swap.quickswap.v1.QuickSwapService`.
- [2026-11-05T2125Z-swap-quickswap-v1](changes/2026-11-05T2125Z-swap-quickswap-v1.md): changed `swap.quickswap.v1.QuickSwapService`.
- [2026-11-05T2145Z-marketoverview-v1](changes/2026-11-05T2145Z-marketoverview-v1.md): changed `marketoverview.v1.MarketOverviewService`.
- [2026-11-05T2208Z-swap-quickswap-v1](changes/2026-11-05T2208Z-swap-quickswap-v1.md): changed `swap.quickswap.v1.QuickSwapService`.
- [2026-11-05T2224Z-changelog-v1](changes/2026-11-05T2224Z-changelog-v1.md): changed `changelog.v1.ChangelogService`.
- [2026-11-05T2300Z-swap-quickswap-v1](changes/2026-11-05T2300Z-swap-quickswap-v1.md): changed `swap.quickswap.v1.QuickSwapService`.
- [2026-11-06T0002Z-triggers-v1](changes/2026-11-06T0002Z-triggers-v1.md): changed `triggers.v1.TriggersService`.
- [2026-11-07T1018Z-auth-v1](changes/2026-11-07T1018Z-auth-v1.md): changed `auth.v1.AuthService`.

## 2026-11-04 — develop-20261104T112923Z

Fee reserve rates and breakdown, removal of the holds list, canonical REST problem types and stricter trigger checks.

- [2026-11-03T1601Z-orders-v1](changes/2026-11-03T1601Z-orders-v1.md): changed `orders.v1.OrdersReadService`.
- [2026-11-03T1601Z-triggers-v1](changes/2026-11-03T1601Z-triggers-v1.md): changed `triggers.v1.TriggersService`.
- [2026-11-03T1943Z-auth-v1](changes/2026-11-03T1943Z-auth-v1.md): changed `auth.v1.AuthService`, `auth.v1.PolicyService`, `auth.v1.SubaccountViewService`, `auth.v1.SubaccountRoleService`, `chain.lifecycle.v1.LifecycleReadService`, `orders.v1.OrdersService`, `orders.v1.OrdersReadService`, `swap.quickswap.v1.QuickSwapService`, `swap.quickswap.v1.QuickSwapBrokerService`.
- [2026-11-03T2227Z-triggers-v1](changes/2026-11-03T2227Z-triggers-v1.md): changed `triggers.v1.TriggersService`, `orders.v1.OrdersService`.
- [2026-11-04T0002Z-triggers-v1](changes/2026-11-04T0002Z-triggers-v1.md): changed `triggers.v1.TriggersService`.
- [2026-11-04T0020Z-triggers-v1](changes/2026-11-04T0020Z-triggers-v1.md): changed `triggers.v1.TriggersService`, `orders.v1.OrdersService`. **Breaking.**
- [2026-11-04T0201Z-orders-v1](changes/2026-11-04T0201Z-orders-v1.md): changed `orders.v1.OrdersService`, `orders.v1.OrdersReadService`, `marketdata.v1.MarketDataService`, `triggers.v1.TriggersService`, `ledger.read.v1.LedgerReadService`, `fees.v1.FeeService`, `ratelimit.v1.RateLimitService`, `chain.deposit.v1.DepositAddressService`, `chain.withdraw.v1.WithdrawService`, `transfer.v1.InternalTransferService`.
- [2026-11-04T0211Z-ledger-read-v1](changes/2026-11-04T0211Z-ledger-read-v1.md): removed `ledger.read.v1.LedgerReadService`. **Breaking.**
- [2026-11-04T0229Z-fees-v1](changes/2026-11-04T0229Z-fees-v1.md): added `fees.v1.FeeService`.
- [2026-11-04T0312Z-triggers-v1](changes/2026-11-04T0312Z-triggers-v1.md): changed `triggers.v1.TriggersService`.
- [2026-11-04T0421Z-triggers-v1](changes/2026-11-04T0421Z-triggers-v1.md): changed `triggers.v1.TriggersService`. **Breaking.**
- [2026-11-04T0430Z-claims-v1](changes/2026-11-04T0430Z-claims-v1.md): changed `claims.v1.ClaimsService`, `swap.quickswap.v1.QuickSwapService`, `swap.quickswap.v1.QuickSwapBrokerService`, `chain.analytics.v1.ChainAnalyticsService`, `chain.deposit.v1.DepositAddressService`, `chain.lifecycle.v1.LifecycleReadService`, `chain.zipper.v1.ZipperService`, `chain.withdraw.v1.WithdrawService`, `chain.withdraw.v1.AutoWithdrawService`, `transfer.v1.InternalTransferService`, `orders.v1.OrdersService`, `triggers.v1.TriggersService`. **Breaking.**
- [2026-11-04T0604Z-transfer-v1](changes/2026-11-04T0604Z-transfer-v1.md): added `transfer.v1.InternalTransferService`, `chain.deposit.v1.DepositAddressService`, `chain.withdraw.v1.WithdrawService`, `chain.withdraw.v1.AutoWithdrawService`, `rewards.v1.RewardCampaignService`, `auth.v1.AuthService`, `auth.v1.ApiKeyService`, `auth.v1.SubaccountService`, `auth.v1.SubaccountViewService`, `auth.v1.SubaccountRoleService`, `auth.v1.PolicyService`, `auth.v1.AddressBookService`, `vip.v1.VIPService`, `ratelimit.v1.RateLimitService`, `fees.v1.FeeService`, `checkpoints.v1.CheckpointService`.

## 2026-11-03 — develop-20261103T203939Z

Self-trade prevention for attached exits, sub-account balances and consistent REST trigger fields.

- [2026-11-03T0019Z-orders-v1](changes/2026-11-03T0019Z-orders-v1.md): changed `orders.v1.OrdersService`.
- [2026-11-03T0209Z-auth-v1](changes/2026-11-03T0209Z-auth-v1.md): changed `auth.v1.SubaccountViewService`.
- [2026-11-03T0259Z-orders-v1](changes/2026-11-03T0259Z-orders-v1.md): changed `orders.v1.OrdersReadService`, `orders.v1.OrdersService`, `triggers.v1.TriggersService`.
- [2026-11-03T1149Z-orders-v1](changes/2026-11-03T1149Z-orders-v1.md): changed `orders.v1.OrdersService`. **Breaking.**
- [2026-11-03T1158Z-triggers-v1](changes/2026-11-03T1158Z-triggers-v1.md): changed `triggers.v1.TriggersService`. **Breaking.**
- [2026-11-03T1911Z-orders-v1](changes/2026-11-03T1911Z-orders-v1.md): changed `orders.v1.OrdersReadService`.
- [2026-11-03T1911Z-triggers-v1](changes/2026-11-03T1911Z-triggers-v1.md): changed `triggers.v1.TriggersService`.

## 2026-11-03 — develop-20261103T152920Z

Exact order execution totals, refused trigger fire events and API key authentication error codes.

- [2026-11-02T0050Z-orders-v1](changes/2026-11-02T0050Z-orders-v1.md): changed `orders.v1.OrdersService`. **Breaking.**
- [2026-11-02T0208Z-auth-v1](changes/2026-11-02T0208Z-auth-v1.md): changed `auth.v1.SubaccountService`. **Breaking.**
- [2026-11-02T1348Z-orders-v1](changes/2026-11-02T1348Z-orders-v1.md): changed `orders.v1.OrdersReadService`, `orders.v1.OrdersService`.
- [2026-11-02T1520Z-changelog-v1](changes/2026-11-02T1520Z-changelog-v1.md): added `changelog.v1.ChangelogService`.
- [2026-11-02T1534Z-triggers-v1](changes/2026-11-02T1534Z-triggers-v1.md): changed `triggers.v1.TriggersService`.
- [2026-11-02T1650Z-orders-v1](changes/2026-11-02T1650Z-orders-v1.md): changed `orders.v1.OrdersService`, `orders.v1.OrdersReadService`.
- [2026-11-02T1721Z-triggers-v1](changes/2026-11-02T1721Z-triggers-v1.md): changed `triggers.v1.TriggersService`.
- [2026-11-02T1734Z-chain-withdraw-v1](changes/2026-11-02T1734Z-chain-withdraw-v1.md): changed `chain.withdraw.v1.AutoWithdrawService`, `chain.withdraw.v1.WithdrawService`.
- [2026-11-02T1743Z-marketoverview-v1](changes/2026-11-02T1743Z-marketoverview-v1.md): changed `marketoverview.v1.MarketOverviewService`.
- [2026-11-02T1747Z-auth-v1](changes/2026-11-02T1747Z-auth-v1.md): changed `orders.v1.OrdersReadService`, `orders.v1.OrdersService`, `triggers.v1.TriggersService`, `ledger.read.v1.LedgerReadService`.
- [2026-11-02T1755Z-chain-withdraw-v1](changes/2026-11-02T1755Z-chain-withdraw-v1.md): changed `chain.withdraw.v1.WithdrawService`, `chain.withdraw.v1.AutoWithdrawService`, `ledger.read.v1.LedgerReadService`, `chain.guard.v1.GuardSignerService`, `vip.v1.VIPService`, `checkpoints.v1.CheckpointService`, `auth.v1.SubaccountRoleService`. **Breaking.**
- [2026-11-02T1759Z-orders-v1](changes/2026-11-02T1759Z-orders-v1.md): changed `orders.v1.OrdersService`. **Breaking.**
- [2026-11-02T1803Z-triggers-v1](changes/2026-11-02T1803Z-triggers-v1.md): changed `triggers.v1.TriggersService`.
- [2026-11-02T1812Z-orders-v1](changes/2026-11-02T1812Z-orders-v1.md): changed `orders.v1.OrdersService`.
- [2026-11-02T1921Z-orders-v1](changes/2026-11-02T1921Z-orders-v1.md): changed `orders.v1.OrdersReadService`.
- [2026-11-02T1922Z-orders-v1](changes/2026-11-02T1922Z-orders-v1.md): changed `orders.v1.OrdersReadService`, `orders.v1.OrdersService`, `triggers.v1.TriggersService`.
- [2026-11-02T2243Z-marketdata-v1](changes/2026-11-02T2243Z-marketdata-v1.md): changed `marketdata.v1.MarketDataService`.
- [2026-11-02T2343Z-orders-v1](changes/2026-11-02T2343Z-orders-v1.md): changed `orders.v1.OrdersService`.
- [2026-11-02T2348Z-orderbook-v1](changes/2026-11-02T2348Z-orderbook-v1.md): changed `orderbook.v1.OrderbookService`.
- [2026-11-03T0123Z-triggers-v1](changes/2026-11-03T0123Z-triggers-v1.md): changed `triggers.v1.TriggersService`, `orders.v1.OrdersReadService`.
- [2026-11-03T0157Z-changelog-v1](changes/2026-11-03T0157Z-changelog-v1.md): added `changelog.v1.ChangelogService`.
- [2026-11-03T0308Z-orders-v1](changes/2026-11-03T0308Z-orders-v1.md): changed `orders.v1.OrdersReadService`.
- [2026-11-03T0320Z-orders-v1](changes/2026-11-03T0320Z-orders-v1.md): changed `orders.v1.OrdersService`, `triggers.v1.TriggersService`.
