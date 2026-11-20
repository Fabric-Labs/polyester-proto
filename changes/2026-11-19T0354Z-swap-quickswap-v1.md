---
services:
  - swap.quickswap.v1.QuickSwapService
  - swap.quickswap.v1.QuickSwapBrokerService
kind: changed
breaking: false
wire_compatible: true
---

## What and why

A QuickSwap read now reports the fees actually charged, so a broker or end user can reconcile the
delivered amount with the quote's estimated fees.

- `QuickSwap.charged_fees` and `QuickSwapSummary.charged_fees` also list the service fee, as
  `ChargedFee` entries of kind `SERVICE`: one per asset, with `account_id` zero, because the service
  fee is credited to Polyester rather than to a vault. They come first, before `BROKER`, then
  `AFFILIATE` and `REFERRAL`. Each has its 18-decimal `amount` and, once valued, its `value_usd`, like
  the other kinds. `ChargedFee` now describes a fee charged, not only what a vault was credited.
- New field `Deposit.network_fee` (`TokenAmount`): the deposit network fee that Zipper withheld when it
  took the deposit in, in the deposit's `zipped_asset_id` token units; zero when it charged none. It is
  unset until Zipper has taken the deposit in, and on a deposit it never takes in: a deposit returned or
  transferred out without being credited, whose deductions are not reported.

With these, a QuickSwap whose deposit was taken in reports an actual amount for every fee kind of
`quote.fees`: `SERVICE`, `BROKER`, `AFFILIATE` and `REFERRAL` in `charged_fees`, `DEPOSIT_NETWORK` in
`Deposit.network_fee` and `WITHDRAWAL_NETWORK` in `Withdrawal.network_fee`. A deposit returned or
transferred out without being credited reports no network fee.

## How clients use it

```text
GET /v1/quickswaps/{swap_id}
-> {"quickSwap": {
     "deposits": [{"amount": {"baseUnits": "100000000", "decimals": 8},
                   "networkFee": {"baseUnits": "2000", "decimals": 8}, ...}],
     "chargedFees": [
       {"kind": "SERVICE", "accountId": "0", "assetId": 2, "amount": {"baseUnits": "...", "decimals": 18}},
       {"kind": "BROKER", "accountId": "...", "assetId": 2, "amount": {"baseUnits": "...", "decimals": 18}}],
     ...}}
```

The same fields appear on `POST /swap.quickswap.v1.QuickSwapService/GetQuickSwap`, on
`QuickSwapBrokerService.GetQuickSwap` and, for `charged_fees`, on each `ListQuickSwaps` summary.

## Migration

None for clients that select `charged_fees` entries by `kind`. A client that sums every
`charged_fees` entry as the broker's earnings must now skip `SERVICE` entries: the broker's earnings
are its `BROKER` and `AFFILIATE` entries. `GetFeeTotals` is unchanged and still totals only the
broker's vaults.
