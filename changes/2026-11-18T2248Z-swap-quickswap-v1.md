---
services:
  - swap.quickswap.v1.QuickSwapBrokerService
kind: changed
breaking: false
wire_compatible: true
---

## What and why

Documentation of an existing status is extended; no message, field or enum changes.
`VaultAllowlistChange.expires_at` now states that a payout address allowlist change can stay
`PENDING` longer than shortly past `expires_at` when its outcome on chain cannot be confirmed yet.
Such a change stays `PENDING` while it is investigated, and the vault takes no other change meanwhile.
A change whose outcome is not known is never reported as `NOT_APPLIED`, which means nothing changed
on the vault's allowlist. When the outcome is confirmed, the change becomes `APPLIED`.

While such a change is pending, `RequestVaultAllowlistChange` for the same vault answers the existing
`ALLOWLIST_CHANGE_PENDING` error code, as it does for any pending change: a vault keeps at most one
pending change.

## How clients use it

Read a vault's changes and show a pending one as in progress, without a deadline:

```text
POST /swap.quickswap.v1.QuickSwapBrokerService/ListVaultAllowlistChanges
{"broker": "acme", "vaultAccountId": "..."}
-> {"changes": [{"status": "PENDING", "expiresAt": "...", ...}, ...]}
```

A `RequestVaultAllowlistChange` for that vault answers `FAILED_PRECONDITION` with
`ALLOWLIST_CHANGE_PENDING` until the pending change becomes `APPLIED` or `NOT_APPLIED`.

## Migration

None: nothing on the wire changes. Clients that treat a `PENDING` change past its `expires_at` as
failed should keep showing it as pending.
