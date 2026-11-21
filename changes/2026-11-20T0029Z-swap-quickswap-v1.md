---
services:
  - swap.quickswap.v1.QuickSwapReferralService
kind: added
breaking: false
wire_compatible: true
---

## What and why

`QuickSwapReferralService.GetReferralStats` reads a referral partner's
activity over time: the QuickSwaps it referred and the commission they
credited it, in hourly, daily or monthly buckets, with an optional breakdown
by route, source asset or destination asset. It answers in the same columnar
shape as `QuickSwapBrokerService.GetBrokerStats`: `GetReferralStatsResponse`
wraps a `GetBrokerStatsResponse`, so one stats view serves brokers and
referral partners.

Every fee in the answer is the partner's own commission. `fees`,
`fees_usd_q4` and `fees_usd` count only the commission; `referral_fee_usd`
and `cell_referral_fee` carry it; the service, broker and affiliate fee
columns are empty.

The doc comments of `BrokerStatsSeries` and `GetBrokerStatsResponse` now
also describe a referral partner's answer. No field changed.

## How clients use it

Session token only, for the caller's own partner record.

```http
POST /swap.quickswap.v1.QuickSwapReferralService/GetReferralStats
{"groupBy": "HOUR", "startTime": "2026-11-12T00:00:00Z", "endTime": "2026-11-19T00:00:00Z", "breakdown": "ROUTE"}
```

- Windows: `HOUR` up to 7 days, `DAY` up to 366, `MONTH` up to 3660.
- **"All"** is one `MONTH` read from `ReferralPartner.created_at`
  (`GetReferralPartner`) to now; its totals series is the lifetime totals.
- **Times are minute-granular**, as in the other referral reads, so a
  partner can never time a referred QuickSwap below a minute (which could tie
  it to its on-chain deposit or payout). Both `startTime` and `endTime` are
  rounded down to the minute: a QuickSwap created in the window's last
  partial minute (the current minute when `endTime` is now) counts from the
  next minute on. A window shorter than a minute once rounded is
  `INVALID_ARGUMENT`.
- **`asOfTsSec`** is rounded up: the end of the minute the newest change
  counted was published in, or the time of the read while that minute is
  current. The answer never holds data newer than it.
- **Rankings:** one call per breakdown (`ROUTE`, `SOURCE_ASSET`,
  `DESTINATION_ASSET`). `VAULT` is `INVALID_ARGUMENT`. `series` holds, in
  order: the totals (`grouping.totals`); every group with activity, ranked
  by deposited USD; a drill-down group without activity, when one was asked
  for; and the `others` aggregate of the groups without per-bucket columns,
  present only when some group lacks them (with exactly 11 active groups
  and a drill-down into the 11th, every group has columns and there is no
  aggregate). For a top-5 list, first select
  the ranked groups: the series whose `grouping` is the breakdown's kind,
  without the totals, the `others` aggregate or an idle drill-down group.
  Take the first five of those, and sum the window totals of the rest of
  them for "others". Summing every series after the fifth double-counts,
  because the `others` aggregate already repeats groups that are also
  listed.
- `includeCells` adds the totals' exact per-bucket, per-asset amounts
  (`cellDeposited`, `cellDelivered`, `cellReferralFee`).
- Errors: `NOT_FOUND` / `REFERRAL_PARTNER_NOT_FOUND` without a partner
  record, `PERMISSION_DENIED` / `ACTION_DENIED` for a broker API key,
  `INVALID_ARGUMENT` for the window, breakdown or drill-down,
  `RESOURCE_EXHAUSTED` / `BROKER_LIMIT` with a `Retry-After` header when the
  caller's read rate is spent (one budget with `GetBrokerStats`, not the
  `RateLimitDetail` the other referral RPCs answer), `UNAVAILABLE` when the
  stats cannot be read (retry the same request).

## Migration

None: the RPC and its two messages are new, and the shared stats messages
keep every field.
