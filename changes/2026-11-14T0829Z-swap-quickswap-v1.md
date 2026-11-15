---
services:
  - swap.quickswap.v1.QuickSwapService
  - swap.quickswap.v1.QuickSwapBrokerService
kind: added
breaking: false
wire_compatible: true
---

## What and why

The REST OpenAPI of QuickSwap now describes the referral fields that the
ConnectRPC contract already carries. REST clients and SDKs built from it can
send a referral code and read the referral pricing and commissions:

- `referralCode` in the `QuoteQuickSwap` and `CreateQuickSwap` request bodies.
- `quote.referral` (`baseBrokerBps`, `discountBps`, `commissionBps`) in the
  `QuoteQuickSwap`, `CreateQuickSwap` and both `GetQuickSwap` responses.
- `referralFee` in each `fees` item of a `GetBrokerStats` period.
- The `400` responses of `QuoteQuickSwap` and `CreateQuickSwap` now name the
  referral errors: `REFERRAL_CODE_INVALID` and `REFERRAL_NOT_SUPPORTED`
  (`INVALID_ARGUMENT`), `REFERRAL_CODE_UNKNOWN` and
  `REFERRAL_PARTNER_INACTIVE` (`FAILED_PRECONDITION`).

The REST bodies and errors are the same as over ConnectRPC, so only the
published REST OpenAPI changes: a schema-validating or generated REST client
no longer rejects or drops these fields. No proto message, field or enum
value changes.

## How clients use it

Send the referral partner's code from a referral link with a quote or a
create. A quote priced with a referral returns its rates; the `REFERRAL` fee
lines carry the partner's commission.

```http
POST /v1/quickswaps:quote
{"referralCode": "alice-7", "terms": {...}}
```

```json
{"quote": {
  "fees": [{"kind": "REFERRAL", "zippedAssetId": 58, "amount": {"baseUnits": "1500", "decimals": 6}, "bps": 20}],
  "referral": {"baseBrokerBps": 50, "discountBps": 10, "commissionBps": 20}
}}
```

`POST /v1/quickswaps` takes the same `referralCode`, and the QuickSwap's
`quote.referral` stays fixed at creation. `GET /v1/quickswaps/{swap_id}` and
`GET /v1/quickswaps:lookup` return it.

`GET /v1/quickswap/brokers/{broker}/stats` reports `referralFee` beside
`serviceFee`, `brokerFee` and `affiliateFee`, in base units of 18 decimals,
and like them always set, zero included.

A request whose `referralCode` fails answers `400` with the error's `code`.
`REFERRAL_CODE_INVALID` means the code is malformed. For
`REFERRAL_NOT_SUPPORTED`, `REFERRAL_CODE_UNKNOWN` and
`REFERRAL_PARTNER_INACTIVE`, repeat the request without the code.

`referral` is present only when the request carried a referral code, and each
of its rates is omitted while zero, like the other QuickSwap REST fields.
`referralCode` is at most 64 bytes.

## Migration

None: the fields are additive. A client that does not use referrals sends no
`referralCode` and keeps working unchanged.
