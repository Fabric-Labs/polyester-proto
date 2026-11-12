---
services:
  - swap.quickswap.v1.QuickSwapBrokerService
kind: added
breaking: false
wire_compatible: true
---

## What and why

Brokers can read the ledger of their vaults: every movement of a vault's
funds, newest first, for the default vault, one affiliate vault or all of
them. Every broker role and the broker's API keys can read it, without
step-up.

- `ListVaultLedger` returns `VaultLedgerEntry` items. Each has a `kind`
  (`VaultLedgerEntryKind`), `occurred_at`, the vault, the asset, a positive
  `amount` at 18 decimals, and one detail:
  - `FEE_CREDIT` / `fee_credit`: a broker or affiliate fee a QuickSwap
    credited. One entry per credit, with its `credit_id`, the `swap_id`, the
    `fee_kind` (`BROKER` or `AFFILIATE`), the rate `fee_bps` (absent when not
    known), the route and the swap's quoted, received and delivered amounts.
  - `PAYOUT_DEBIT` / `payout_debit`: one asset's payout that left the vault,
    listed once its withdrawal ended. It carries the `run_id` (read the run
    with `GetPayout`), the run's `payout_status`, destination, `tx_hash`, step
    `state` and `reason`, `charged_network_fee`, and `close` (`PayoutClose`):
    `CLOSE_PAID`, `CLOSE_REFUNDED_TO_SMART_ACCOUNT`, `CLOSE_SENT_TO_RECOVERY`,
    or `CLOSE_UNKNOWN` for a failed payout whose destination is not known:
    it ended before this was recorded, its outcome was not known when it
    ended, or the records of its end disagree. A payout that paid nothing, or
    was released before anything left, has no entry.
  - `PAYOUT_REFUND` / `payout_refund`: the net of a payout the destination
    network's bridge rejected, refunded to the vault's own smart account. It
    is not part of the vault balance `ListVaultBalances` reads.
- `PayoutStatus.PAYOUT_FAILED`'s comment now points to the ledger for where
  a failed payout's funds went.

The ledger has no running balance: anchor balances on `ListVaultBalances`.
A value that is not known is absent, never zero.

## How clients use it

```http
POST https://<api-host>/swap.quickswap.v1.QuickSwapBrokerService/ListVaultLedger
Authorization: Bearer <broker API key or member session token>
Content-Type: application/json

{"broker": "acme", "vaultAccountId": "7433029815562201289", "limit": 25}
```

REST: `GET /v1/quickswap/brokers/{broker}/vault-ledger?vaultAccountId=...&limit=25`.

- Filters: `vault_account_id` (one vault, or zero for all) and
  `start_time`/`end_time` on `occurred_at`, start inclusive and end
  exclusive. `limit` 0 means 25, at most 100.
- Order: newest first by `occurred_at`. At one time, payouts come before fee
  credits and a refund before its debit; otherwise entries of the same time
  follow a stable server-defined order, the same on every page, not
  `credit_id` or `run_id`. A debit and its refund occur at the payout run's
  scheduled slot.
- `page_token` is an opaque exclusive keyset cursor bound to the broker, the
  vault filter and the time bounds; reuse it with the same filters. A page
  may hold fewer than `limit` entries while `next_page_token` is set:
  continue until it is empty. Each entry that sorts after the previous page
  is listed exactly once; one that appears meanwhile at a newer time is
  listed only when listing again from the first page.
- A key for an entry: `fee_credit.credit_id`, or `run_id` with `asset_id`
  and `kind`.
- A vault that is not the broker's is `NOT_FOUND` / `VAULT_NOT_FOUND`.

## Migration

None: the RPC, messages and enums are new.
