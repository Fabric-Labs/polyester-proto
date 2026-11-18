---
services:
  - chain.withdraw.v1.WithdrawService
  - chain.withdraw.v1.AutoWithdrawService
kind: changed
breaking: false
wire_compatible: true
---

## What and why

`destination_chain_id` in withdraw requests is the **Polyester chain id** of
the destination network: `ChainConfig.chain_id` from
`GetDepositWithdrawConfig`. It is not the network's native chain id
(`ChainConfig.native_chain_id`, such as `11155111` for Ethereum Sepolia).
The field comments now say so on `ValidateWithdrawDestinationRequest`,
`TradingWithdrawIntentPayload` and `AutoWithdrawDestination`.

A chain id that is not a Polyester chain id used to be answered as a
temporary failure: HTTP 503 / `unavailable`,
`ERROR_CODE_DESTINATION_VALIDATION_UNAVAILABLE`. Retrying never helped. It is
now refused as the caller's mistake, with a message naming the id to use:

- `ValidateWithdrawDestination` answers `valid: false`,
  `code: UNSUPPORTED_CHAIN`.
- `CreateTradingWithdraw` and `CreateWalletTradingWithdraw` refuse it with
  `ERROR_CODE_INVALID_DESTINATION_CHAIN` (INVALID_ARGUMENT, HTTP 400).

The message for both is "destination_chain_id must be a Polyester chain id
from GetDepositWithdrawConfig (chains[].chainId), not the native chain id".
`ERROR_CODE_DESTINATION_VALIDATION_UNAVAILABLE` remains for a real,
temporary validation outage.

## How clients use it

Look up the destination network's `chain_id` in the deposit-withdraw config,
then pass it as `destination_chain_id`:

```text
GET /v1/chain/deposit-withdraw/config
  chains[]: { "chainId": 2, "code": "ethereum-sepolia", "nativeChainId": "11155111", ... }

POST /v1/chain/withdraws:validate-destination
POST /chain.withdraw.v1.WithdrawService/ValidateWithdrawDestination
{ "destinationChainId": "2", "destinationAddress": "0x..." }
-> { "valid": true, "code": "VALID", ... }

{ "destinationChainId": "11155111", "destinationAddress": "0x..." }
-> { "valid": false, "code": "UNSUPPORTED_CHAIN",
     "message": "destination_chain_id must be a Polyester chain id from GetDepositWithdrawConfig (chains[].chainId), not the native chain id" }
```

`POST /v1/chain/trading-withdraws` and `POST /v1/chain/trading-withdraws:wallet`
with the native id answer HTTP 400, `code: INVALID_DESTINATION_CHAIN`.

## Migration

None for clients that already send `ChainConfig.chain_id`. A client that sent
a native chain id was always refused. It now gets a non-retryable 400 or
`valid: false` instead of a 503, so it must not retry, and should send the
Polyester chain id instead.
