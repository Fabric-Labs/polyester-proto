---
services:
  - chain.withdraw.v1.WithdrawService
kind: added
breaking: false
wire_compatible: true
---

## What and why

New RPC `WithdrawService.CreateWalletTradingToFundingBatch` and its messages
`CreateWalletTradingToFundingBatchRequest`, `TradingToFundingBatchLeg`,
`CreateWalletTradingToFundingBatchResponse` and
`TradingToFundingBatchLegResult`.

A user can now move up to 10 amounts from one Trading account to Funding
accounts with one wallet signature and one MFA check, instead of one signed
withdraw and one MFA check per amount. The legs move together or not at
all:

- A leg that cannot be accepted, for example for insufficient funds or a
  destination missing from the internal whitelist, refuses the whole batch
  and nothing moves. The error's `field_path` names the leg, such as
  `legs[2]` or `legs[2].destination_address`.
- An accepted batch is paid as one. If the payment fails, every leg is
  refunded to the source Trading account.
- A refused batch is final: a retry under its `idempotency_key` answers the
  same refusal, even once its cause is gone (for example after the
  destination is whitelisted). This holds whatever the refusal's status,
  `unavailable` and `aborted` included: a leg refused because the account's
  capital view was not ready is recorded as `CAPITAL_VIEW_UNAVAILABLE` at
  `legs[1]`, status `unavailable`, and its replay answers exactly that, even
  after the capital view recovers. A corrected or repeated batch needs a new
  key and a new signature.
- A retry of an accepted batch answers the same batch and intents, even
  after the signed deadline has passed.

MFA follows the single Trading to Funding withdraw: when the caller owns the
source account's main account and every leg goes to its own accounts (an
omitted `destination_address`, or a Funding address of the main account or
one of its subaccounts), a recently elevated session is enough, and `AUTH_MFA_ELEVATION_REQUIRED` answers a session that is
not. When any leg goes elsewhere, the request carries one fresh step-up proof
for the whole batch, and `AUTH_STEP_UP_REQUIRED` answers a missing or spent
proof. The proof is single-use: a retry of a batch the server has recorded
needs no new proof, but if the first attempt failed before the batch was
recorded, the retry needs a new one.

## How clients use it

```http
POST /v1/chain/trading-withdraws:wallet-batch
     (ConnectRPC: POST /chain.withdraw.v1.WithdrawService/CreateWalletTradingToFundingBatch)
Authorization: Bearer <session token>
X-Auth-Step-Up: <fresh step-up token, only when a leg leaves the caller's accounts>

{
  "legs": [
    {"assetId": 42, "amount": "0.000000001"},
    {"assetId": 7, "amount": "0.5",
     "destinationAddress": "0x2222222222222222222222222222222222222222"}
  ],
  "subaccountId": "<base58 public subaccount id; omit for the root account>",
  "deadlineTsSec": "1900000000",
  "nonce": "18446744073709551716",
  "idempotencyKey": "bulk-0001",
  "signerWallet": "0x2c7536e3605d9c16a7a3d7b1898e529396a65c23",
  "payloadSignature": "<base64 of the 65-byte signature>"
}
```

On REST, `amount` is a decimal string in whole units of the asset, and the
signed text carries it times 10^18 (`Amount E18`). On ConnectRPC, the leg
carries `amountE18` and the request `nonce` as `polyester.type.v1.U128`
(`{"hi": "0", "lo": "500000000000000000"}`), and `subaccountId` is the public
subaccount ID as a JSON string.

The response lists every leg in request order with its `intentId`, the
identifier a single wallet withdraw returns, so each leg is followed like a
single withdraw:

```json
{"batchId": "twb_...", "legs": [
  {"legIndex": 0, "intentId": "...", "assetId": 42, "amount": "0.000000001"},
  {"legIndex": 1, "intentId": "...", "assetId": 7, "amount": "0.5",
   "destinationAddress": "0x2222222222222222222222222222222222222222"}
]}
```

The wallet signs this UTF-8 text with EIP-191 `personal_sign`. Lines are
joined by one line feed, with none after the last line. A leg's
`Destination` is its `destinationAddress` as sent, in the letter case sent
(a checksummed address stays checksummed), without surrounding whitespace;
an omitted `destinationAddress` leaves nothing after `Destination: ` (the
space stays). `Signer Wallet` is lowercase.
`Account ID` and `Target Account ID` are the public account IDs in base 10,
the target being the source account: the subaccount, or the caller's
account when `subaccountId` is omitted. `Polyester Chain ID` is the EVM chain ID of Polyester Chain, as `eth_chainId`
returns it, and `Verifying Contract` is the same lowercase address a single
wallet withdraw signs (`CreateWalletTradingWithdraw`). `Nonce` is the 128-bit
nonce in base 10. The legs follow in request order, each block after one
empty line:

```text
Polyester Trading Withdrawal Batch

Version: 1
Environment: polyester
Action: TO_FUNDING
Signer Wallet: 0x2c7536e3605d9c16a7a3d7b1898e529396a65c23
Account ID: 1001
Target Account ID: 2002
Polyester Chain ID: 31337
Verifying Contract: 0x1111111111111111111111111111111111111111
Deadline: 1900000000
Nonce: 18446744073709551716
Idempotency Key: bulk-0001
Legs: 2

Leg 1 of 2
Asset ID: 42
Amount E18: 1000000000
Destination: 

Leg 2 of 2
Asset ID: 7
Amount E18: 500000000000000000
Destination: 0x2222222222222222222222222222222222222222
```

Golden vector: the private key
`0x4c0883a69102937d6231471b5dbb6204fe5129617082792ae468d01a3f362318`
(wallet `0x2c7536e3605d9c16a7a3d7b1898e529396a65c23`) signs the text above
as

```text
0xa6aef9e82b6029c93fba8e8c682fc4012e3cc9b741e62e007405b49a53e850491ffd662ed011412565cf1b5a9ea808a905f660647f5063e13d7692f57980e2851c
```

Limits: 1 to 10 legs; legs may repeat an asset or a destination; the deadline
is at most 15 minutes ahead; the idempotency key is 1 to 128 characters,
scoped to the source account. Retrying with the same key and the same signed
batch returns the first answer; the same key with another batch is refused
with `ERROR_CODE_IDEMPOTENCY_CONFLICT`. An attempt still in flight answers
`aborted`; retry it unchanged. Each leg of a new batch counts one unit
against the withdraw rate limit, whose default burst admits a full batch of
10; a retry answered from the stored outcome costs nothing. A refusal
replays with the status and code of its first answer.

## Migration

None: the RPC is new, and `CreateWalletTradingWithdraw` is unchanged.
