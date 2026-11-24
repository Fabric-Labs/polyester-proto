---
services:
  - chain.withdraw.v1.WithdrawService
  - chain.withdraw.v1.AutoWithdrawService
  - swap.quickswap.v1.QuickSwapService
kind: changed
breaking: true
wire_compatible: true
---

## What and why

A QuickSwap deposit address can no longer receive a withdrawal. A
withdrawal could reach it after its swap stopped accepting deposits, and
those funds could then be neither swapped nor returned. A Polyester
account's own deposit address is not affected: it stays a valid withdrawal
and QuickSwap destination.

Added to `chain.withdraw.v1`:

- `WithdrawDestinationValidationCode.SWAP_DEPOSIT_ADDRESS` (7):
  `ValidateWithdrawDestination` answers `valid: false` with this code for a
  QuickSwap deposit address in any spelling of the same address: letter case
  where the network ignores it (EVM, bech32 and CashAddr; on Solana and
  other case-sensitive Base58 networks another case is another address), a
  destination tag with leading zeros, or another encoding (a Bitcoin Cash
  legacy address for its CashAddr, and the reverse). A TRON destination is
  validated in Base58Check form only: a hex TRON address is
  `INVALID_ADDRESS`. An address that no longer accepts QuickSwap deposits
  still counts.
- `ErrorCode.ERROR_CODE_DESTINATION_SWAP_DEPOSIT_ADDRESS` (51), REST
  `DESTINATION_SWAP_DEPOSIT_ADDRESS`, `INVALID_ARGUMENT` / HTTP 400:
  `CreateTradingWithdraw` and `CreateWalletTradingWithdraw` refuse such a
  destination before anything is reserved, and `SetAutoWithdrawRule` refuses
  it with `ErrorDetail.field_path` naming the destination.

Changed in `swap.quickswap.v1` (documentation of existing codes only):
`CreateQuickSwap` refuses a QuickSwap deposit address as
`destination_address` (`DESTINATION_ADDRESS_INVALID`), and any Polyester
deposit address, a QuickSwap one or an account's own, as `return_address`
(`RETURN_ADDRESS_INVALID`) or `refund_address` (`REFUND_ADDRESS_INVALID`).
Nothing is bound to the idempotency key. No field or enum value was added
to that package.

## How clients use it

Validate a destination before showing it as usable:

```http
POST /chain.withdraw.v1.WithdrawService/ValidateWithdrawDestination
Content-Type: application/json

{"destinationChainId": "5", "destinationAddress": "0x1111111111111111111111111111111111111111"}
```

answers, when that address is a QuickSwap deposit address,

```json
{"valid": false, "code": "SWAP_DEPOSIT_ADDRESS", "message": "destination address is a QuickSwap deposit address and cannot receive withdrawals"}
```

The REST route is `POST /v1/chain/withdraws:validate-destination`. A
withdrawal to that address fails with `ErrorDetail.code`
`ERROR_CODE_DESTINATION_SWAP_DEPOSIT_ADDRESS` (REST problem `code`
`DESTINATION_SWAP_DEPOSIT_ADDRESS`).

## Migration

A client that treats `ValidateWithdrawDestination.code` or the withdraw
`ErrorCode` as a closed set must accept the new values: show
`SWAP_DEPOSIT_ADDRESS` / `DESTINATION_SWAP_DEPOSIT_ADDRESS` as "this address
is a QuickSwap deposit address; enter the address that should receive the
funds". An unknown value should already be treated as an invalid
destination. QuickSwap clients need no change: the error codes are the ones
they already handle.
