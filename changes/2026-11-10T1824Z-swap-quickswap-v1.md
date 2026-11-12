---
services:
  - swap.quickswap.v1.QuickSwapBrokerService
kind: added
breaking: false
wire_compatible: true
---

## What and why

Brokers can read their vaults' payouts: the scheduled payout runs of the
default vault and every affiliate vault, what the next run of each would pay,
and each vault's payout rule. Every broker role and the broker's API keys can
read them, without step-up.

- `ListPayouts` and `GetPayout`: payout runs, newest first. Each `Payout` has
  a `status` (`PayoutStatus`: `PAYOUT_PAID`, `PAYOUT_PARTIALLY_PAID`,
  `PAYOUT_IN_PROGRESS` with a `stage`, `PAYOUT_ROLLED_OVER`, `PAYOUT_HELD`,
  `PAYOUT_FAILED`) and a machine-readable `reason`, its slot, start and end,
  and one `PayoutStep` per asset: amount, destination chain and address,
  state, reason, `tx_hash`, and `charged_network_fee`, the fee charged on a
  completed withdrawal, or kept on one the network's bridge rejected or
  failed to execute. A rolled-over step carries the balance evaluated, its
  USD value and the minimum it was held to.
- `ListUpcomingPayouts`: each vault's next run, previewed from its balances
  now, with `estimated_network_fee`, the network's latest sampled fee, and
  `payable_from` for a destination still cooling down
  (`destination_cooling_down`: added to the vault's external withdraw
  whitelist less than 24 hours ago).
- `ListVaultPayoutRules`: each vault's rule (schedule, destinations with
  their minimum, ceiling, current floor and `estimated_network_fee`, pause)
  and `assets_without_destination`, the assets the vault holds or was
  credited in that no destination pays.
- `ErrorCode.PAYOUT_NOT_FOUND` (46): no payout run of the broker's vaults has
  the requested `run_id`.

A charged and an estimated network fee have distinct names. A value that is
not known is absent, never zero. A failed payout makes no claim about where
its funds are: show "the payout failed" with its `reason` and, when set,
`tx_hash`.

## How clients use it

```http
POST https://<api-host>/swap.quickswap.v1.QuickSwapBrokerService/ListPayouts
Authorization: Bearer <broker API key or member session token>
Content-Type: application/json

{"broker": "acme", "statuses": ["PAYOUT_FAILED"], "limit": 50}
```

REST: `GET /v1/quickswap/brokers/{broker}/payouts?status=PAYOUT_FAILED&limit=50`,
`GET /v1/quickswap/brokers/{broker}/payouts/{run_id}`,
`GET /v1/quickswap/brokers/{broker}/upcoming-payouts` and
`GET /v1/quickswap/brokers/{broker}/payout-rules`.

- `ListPayouts` filters by `vault_account_id` (one vault, or zero for all),
  `statuses` (at most 6) and `start_time`/`end_time` on the scheduled slot,
  start inclusive and end exclusive. `limit` 0 means 50, at most 100. Order:
  scheduled slot, then `run_id`, both descending. `page_token` is an opaque
  exclusive keyset cursor bound to the broker and the filters; reuse it with
  the same filters. Each run that sorts after the previous page is listed
  exactly once; a run started meanwhile for a newer slot appears only when
  listing again from the first page.
- `ListUpcomingPayouts` and `ListVaultPayoutRules` page vaults as
  `ListVaultBalances` does: `vault_account_id` selects one, `limit` 0 means
  20, at most 200, and every vault without a rule is listed too.
- A vault that is not the broker's is `NOT_FOUND` / `VAULT_NOT_FOUND`; a run
  of another broker's vault is `NOT_FOUND` / `PAYOUT_NOT_FOUND`.
- Amounts are `TokenAmount` at 18 decimals, USD values too. `run_id` and
  `vault_account_id` are public fixed64 IDs, decimal strings in JSON.

## Migration

None: the RPCs, messages, enums and error code are new.
