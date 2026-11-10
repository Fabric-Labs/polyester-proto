---
services:
  - chain.zipper.v1.ZipperService
  - swap.quickswap.v1.QuickSwapService
kind: added
breaking: false
wire_compatible: true
---

## What and why

Each `ChainConfig` returned by `GetDepositWithdrawConfig` adds `memo_requirement`,
`memo_label`, and `memo_format_hint`. `MemoRequirement` is `REQUIRED`, `OPTIONAL`,
or `NOT_SUPPORTED`. An unknown requirement is absent in REST and
`MEMO_UNSPECIFIED` in ConnectRPC.
Clients can render and validate memo inputs before creating a swap or withdrawal.
XRP targets accept an optional destination tag. Other currently supported networks
(EVM, Bitcoin, Bitcoin Cash, Dogecoin, Litecoin, Solana, TRON) reject memos.

## How clients use it

Call `chain.zipper.v1.ZipperService/GetDepositWithdrawConfig` with `{}` or
`GET /v1/chain/deposit-withdraw/config`. The response is a finite config snapshot.
The new fields use camelCase in both REST and Connect JSON. `memoRequirement` is
optional in REST when metadata is unknown. An XRP chain includes:

```json
{
  "memoRequirement": "OPTIONAL",
  "memoLabel": "Destination tag",
  "memoFormatHint": "Unsigned 32-bit decimal integer (0–4294967295)"
}
```

Join an asset variant's `chainId` to the chain config. Require a nonempty memo for
`REQUIRED`; allow an empty value for `OPTIONAL`; hide and clear the field for
`NOT_SUPPORTED`. An absent REST requirement or ConnectRPC `MEMO_UNSPECIFIED`
means unknown; treat that network's memo behavior as unavailable.
The format hint is display text, not a regular expression. For XRP, a tag is
optional; when provided, validate an unsigned decimal integer from 0 through
4294967295, inclusive; zero is valid.

For QuickSwap, send the bare address and separate `destinationMemo`, `returnMemo`,
or `refundMemo`. For Trading withdrawals, use a bare address when no tag is needed,
or the existing `address:tag` destination encoding when the receiving service
supplies a tag. Deposit instructions continue to supply their own memo.
A source wallet identifies the sender and does not require a destination tag.

## Migration

Regenerate client types and remove hardcoded memo-network lists. Untagged XRP
payout targets remain valid, including existing swaps and withdrawal mandates.
Show an optional destination-tag input for XRP; require a tag only when the
receiving service instructs the user to supply one. Existing destination,
return, and refund address error codes remain unchanged. The contract change is
additive. Networks without a deposit-address family advertise `NOT_SUPPORTED`.
A configured family that is missing or has an unknown protocol affects only that
network; other networks stay available.
