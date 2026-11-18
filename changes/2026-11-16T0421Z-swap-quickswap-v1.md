---
services:
  - swap.quickswap.v1.QuickSwapService
kind: changed
breaking: false
wire_compatible: true
---

## What and why

Brokers that know the end user's wallet before a swap (for example a broker
running inside the user's wallet) can now have it risk screened before
Polyester gives a quote or a deposit address, so a flagged wallet is refused
before any funds move.

- `QuoteQuickSwapRequest` gains an optional `source_wallet` (field 5), with
  the same rules as `CreateQuickSwapRequest.source_wallet`: a valid address
  on the source network, at most 256 UTF-8 bytes. An invalid one fails with
  `SOURCE_WALLET_INVALID`.
- A declared `source_wallet`, on the quote or on the create, is risk screened
  before the quote or the deposit address is returned. A wallet that fails
  screening is refused with the new `ErrorCode.SOURCE_WALLET_REJECTED` (56),
  as `PERMISSION_DENIED` (HTTP 403):
  - on `QuoteQuickSwap` nothing is bound;
  - on `CreateQuickSwap` the QuickSwap (its `swap_id` is in the error detail)
    is created with the new terminal status `QuickSwapStatus.REJECTED` (1) and
    the new reason `QuickSwapReason.SOURCE_WALLET_SCREENING_FAILED` (28). It
    never gets a deposit address, and every repeat of the request with the
    same idempotency key returns the same error.
- A REJECTED QuickSwap is returned by `GetQuickSwap` and
  `QuickSwapBrokerService.ListQuickSwaps`, and counted under `REJECTED` in
  `GetBrokerStats`, so a broker can show refused requests.
- Wallets on networks where source wallets cannot be screened in advance,
  and wallets whose screening is unavailable at the time, are quoted and
  created as before. Every deposit is still screened on arrival, whoever
  sent it, and without a `source_wallet` nothing changes.

## How clients use it

Quote with the wallet the user will deposit from:

```http
POST /swap.quickswap.v1.QuickSwapService/QuoteQuickSwap
{"terms": {...}, "brokerBps": 5, "sourceWallet": "0x5aAeb6053F3E94C9b9A09f33669435E7Ef1BeAed"}
```

REST: `POST /v1/quickswaps:quote` with the same body. Then create with the
same `sourceWallet` (`POST /v1/quickswaps`). A refusal is:

```json
{"code": "SOURCE_WALLET_REJECTED", "swapId": "1234"}
```

with HTTP 403 (`swapId` only on the create). Tell the user the swap cannot
go ahead from this wallet; repeating the request does not change the answer.

## Migration

None: the field, status, reason and error code are additive. Clients that
switch on `QuickSwapStatus`, `QuickSwapReason` or `ErrorCode` should treat
unknown values as documented (`is_terminal` is authoritative for status).
