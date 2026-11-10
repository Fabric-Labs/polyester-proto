---
services:
  - swap.quickswap.v1.QuickSwapBrokerService
kind: added
breaking: false
wire_compatible: true
---

## What and why

`ListVaultBalances` adds current balances for a broker's default and affiliate
vaults. All broker roles and the broker's API keys can read without step-up.
The new messages are `ListVaultBalancesRequest`, `ListVaultBalancesResponse`,
`VaultBalance` and `VaultAssetBalance`. No existing method changes behavior.

## How clients use it

ConnectRPC, with the broker API key or member session bearer token:

```http
POST /swap.quickswap.v1.QuickSwapBrokerService/ListVaultBalances
Content-Type: application/json
Authorization: Bearer <token>

{"broker":"acme","limit":20}
```

REST: `GET /v1/quickswap/brokers/acme/vault-balances?limit=20` with the same
authentication and response shape. Optional `vaultAccountId` selects one
public subaccount ID; omit it for the default and all affiliate vaults,
including disabled ones. Another broker's vault or the root account returns
NotFound. No USD valuation is included.

The default limit is 20, maximum 200. Page tokens are at most 128 characters;
limit may change between pages. Follow `nextPageToken` as `pageToken`
until empty, retaining the broker and vault filter. Vaults are ordered by
account creation ID, independently of encoded public IDs; assets by asset ID.
An unfunded vault has an empty asset list, not a missing vault.

Each asset's `amount.baseUnits` is the settled balance of its Funding and
Trading accounts at `amount.decimals = 18`. Pending credits are excluded.
`pendingOutgoing` uses the same scale. `amount` still counts these funds and
has **not** been reduced by `pendingOutgoing`. Subtract `pendingOutgoing`
from `amount` to obtain the balance not yet committed to outgoing transfers.
This figure excludes order holds and does not guarantee withdrawability.
Do not sum unlike assets before valuation.

`asOf` is each vault read's completion time. Vaults/pages are separate reads,
not a single atomic snapshot. A failed balance read fails the whole page with
Unavailable; do not display that error as a zero balance. This capability is
pending deployment and SDK publication; availability must be announced per
environment when released.

## Migration

None: this is an additive read. Existing calls and stored identifiers remain
unchanged.
