---
services:
  - swap.quickswap.v1.QuickSwapBrokerService
  - auth.v1.SubaccountViewService
kind: added
breaking: false
wire_compatible: true
---

## What and why

Brokers can manage the payout addresses of their vaults from the
dashboard. Each vault (the default vault or an affiliate vault) has a
payout address allowlist: the only destinations its payouts and
withdrawals may pay. A newly added address becomes payable 24 hours after
it is listed.

- `ListVaultAllowlist` returns the vault's `VaultAllowlistEntry` items:
  `destination_chain_id`, `destination_address` (canonical), `label`,
  `created_by_account_id`, `listed_at`, `payable_from` (24 hours after
  `listed_at`), `cooling_down` and `used_by_payout_rule`. Every broker role
  and the broker's API keys can read it, without step-up.
- `RequestVaultAllowlistChange` asks to add one destination
  (`ADD_DESTINATION`, with an optional `label`) or remove one
  (`REMOVE_DESTINATION`). Only Owner and Finance members can, with a session
  token and a fresh step-up proof in the `X-Auth-Step-Up` header; each proof
  authorizes one change. The change is returned `PENDING`.
- `ListVaultAllowlistChanges` lists a vault's changes, newest first. Each
  `VaultAllowlistChange` is `PENDING`, `APPLIED` or `NOT_APPLIED`, with a
  `failure` (`AllowlistChangeFailure`) for `NOT_APPLIED`. A vault has at most
  one pending change.
- New `ErrorCode` values: `ALLOWLIST_DESTINATION_INVALID`,
  `ALLOWLIST_DESTINATION_EXISTS`, `ALLOWLIST_FULL`,
  `ALLOWLIST_DESTINATION_NOT_LISTED`, `ALLOWLIST_DESTINATION_IN_USE`,
  `ALLOWLIST_CHANGE_PENDING` and `VAULT_ALLOWLIST_NOT_READY`.
- `auth.v1.ActivityEventAction` gains `ACTIVITY_ACTION_REQUESTED` and
  `ACTIVITY_ACTION_APPLIED`: a vault's activity shows each payout address
  change when it is requested (entity `ACTIVITY_ENTITY_DESTINATION`), then
  when it is applied or fails (`ACTIVITY_ACTION_FAILED`).

## How clients use it

```http
POST https://<api-host>/swap.quickswap.v1.QuickSwapBrokerService/RequestVaultAllowlistChange
Authorization: Bearer <member session token>
X-Auth-Step-Up: <fresh step-up token>
Content-Type: application/json

{"broker": "acme", "vaultAccountId": "7433029815562201289", "action": "ADD_DESTINATION",
 "destinationChainId": "8453", "destinationAddress": "0x5a0b54d5dc17e0aadc383d2db43b0a0d3e029c4c",
 "label": "Affiliate payouts"}
```

REST:

- `GET /v1/quickswap/brokers/{broker}/vaults/{account_id}/allowlist?destinationChainId=...`
- `GET /v1/quickswap/brokers/{broker}/vaults/{account_id}/allowlist/changes?limit=20&pageToken=...`
- `POST /v1/quickswap/brokers/{broker}/vaults/{account_id}/allowlist/changes`

Then poll `ListVaultAllowlistChanges` until the change leaves `PENDING`, and
show an added address as usable from its entry's `payable_from` (for
example "usable from 14:00 UTC tomorrow").

- `ListVaultAllowlist` is the vault's whole list, at most 50 destinations per
  destination chain, ordered by chain then address; it is not paginated.
- `ListVaultAllowlistChanges`: `limit` 0 means 20, at most 100. `page_token`
  is an opaque exclusive keyset cursor bound to the broker and vault.
- A removal names an address on the list; one the vault's payout rule pays
  (enabled or paused) is refused `ALLOWLIST_DESTINATION_IN_USE`: change the
  rule first.
- An `UNAVAILABLE` answer to `RequestVaultAllowlistChange` has an unknown
  outcome: read `ListVaultAllowlistChanges` before requesting again.

## Migration

None: the RPCs, messages and enum values are new. A client that renders
`ActivityEventAction` should handle the two new values.
