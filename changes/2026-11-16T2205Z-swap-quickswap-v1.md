---
services:
  - swap.quickswap.v1.QuickSwapExplorerService
kind: changed
breaking: false
wire_compatible: true
---

## What and why

Each completed QuickSwap the public Explorer lists now shows the
transactions and addresses of its deposit and of its payout, so an explorer
page can link both ends of a swap to their networks. The same transactions
and addresses are already public on those networks.

- `ExplorerQuickSwap.deposits` (new, repeated `ExplorerDeposit`): the
  credited deposit that funded the QuickSwap, one in practice. `tx_hash` is
  the source-network transaction that sent it and `address` the QuickSwap
  deposit address it was paid into.
- `ExplorerQuickSwap.withdrawals` (new, repeated `ExplorerWithdrawal`): the
  finalized payouts to the destination, one in practice. `tx_hash` is the
  destination-network transaction and `address` the destination address.
  `destination_amount` is their total.
- Transactions and addresses are in each network's canonical text form, as
  `QuickSwapService.GetQuickSwap` reports them. A payout's `tx_hash` can be
  corrected for up to 24 hours after `completed_at`.
- Still never shown: destination tags and memos, return and refund addresses,
  refunds and returns, deposits that were returned or held, the broker, fees,
  idempotency keys and the quote.

## How clients use it

Unchanged calls; read the new fields from each listed QuickSwap.

```bash
curl -sS 'https://<api-host>/v1/quickswap/explorer/quickswaps?limit=1'
```

```json
{"quickSwaps":[{"swapId":"18239957911229591281","completedAt":"2026-11-16T12:00:00Z",
  "sourceZippedAssetId":3,"destinationZippedAssetId":58,
  "sourceAmount":{"baseUnits":"1000000","decimals":6},"destinationAmount":{"baseUnits":"1000","decimals":9},
  "legs":[{"symbolId":1}],
  "deposits":[{"txHash":"<source tx>","address":"<deposit address>"}],
  "withdrawals":[{"txHash":"<destination tx>","address":"<destination address>"}]}],
 "nextPageToken":"..."}
```

ConnectRPC `/swap.quickswap.v1.QuickSwapExplorerService/ListCompletedQuickSwaps`
returns the same fields. Paging, filters and limits are unchanged.

## Migration

None: the fields are new and optional to read; existing clients ignore them.
