---
services:
  - swap.quickswap.v1.QuickSwapReferralService
kind: added
breaking: false
wire_compatible: true
---

## What and why

`QuickSwapReferralService.ListReferralLedger` lists a referral partner's
commission ledger: each commission credited to the partner's earnings
sub-account, with the QuickSwap it was earned on, and each auto-withdraw
payout that left that sub-account, newest first. It returns the same
`VaultLedgerEntry` a broker's vault ledger does, so one ledger view serves
brokers' vaults and referral partners. A commission credit is a
`FEE_CREDIT` entry with `fee_kind` `REFERRAL`.

The doc comments of `VaultLedgerEntry`, `VaultLedgerEntryKind` and
`VaultLedgerFeeCredit` now also describe a referral partner's entries. No
field changed.

The partner's balances, payout settings, upcoming payout and payout history
are those of its earnings sub-account (`ReferralPartner.earnings_subaccount_id`):
read and manage them with `LedgerReadService` and `AutoWithdrawService`, as
for any sub-account.

## How clients use it

Session token only, for the caller's own partner record.

```http
POST /swap.quickswap.v1.QuickSwapReferralService/ListReferralLedger
{"startTime": "2026-11-01T00:00:00Z", "limit": 50}
```

- `limit`: 0 means 25, at most 100. A page may hold fewer entries than
  `limit` while `nextPageToken` is set; keep paging until it is empty.
- `startTime` / `endTime` select whole minutes: start is rounded down, end
  up.
- Order: newest `occurredAt` first; at one time payouts before credits and a
  refund before its debit. A credit's `occurredAt` is rounded down to the
  minute, and the credits of one minute have a fixed order that does not
  follow when they were credited.
- `vaultAccountId` on every entry is the earnings sub-account.
- The ledger has no running balance and no movement the partner makes
  itself; read those with `LedgerReadService`.
- Errors: `NOT_FOUND` / `REFERRAL_PARTNER_NOT_FOUND` without a partner
  record, `PERMISSION_DENIED` / `ACTION_DENIED` for a broker API key,
  `INVALID_ARGUMENT` for a page token of another partner or window,
  `UNAVAILABLE` when payouts cannot be read (retry the same request).

## Migration

None: the RPC and its messages are new, and the shared ledger messages keep
every field.
