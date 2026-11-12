---
services:
  - swap.quickswap.v1.QuickSwapService
  - swap.quickswap.v1.QuickSwapBrokerService
kind: changed
breaking: false
wire_compatible: true
---

## What and why

A Quick Swap can now carry a referral partner's code. Polyester's own Quick
Swap app sends the code from a partner's referral link with the quote and
the create. The user gets a discount and the partner earns a commission, both
out of the broker fee; the service fee is unchanged. Rates are whole basis
points per route: bridging when nothing is traded, stablecoin when every leg
trades two stablecoins, normal otherwise.

- `QuoteQuickSwapRequest.referral_code` and
  `CreateQuickSwapRequest.referral_code`: an optional partner code, at most
  64 bytes, case-insensitive.
- `QuickSwapQuote.referral`: a new `ReferralPricing{base_broker_bps,
  discount_bps, commission_bps}`, set only when a code was applied. On a
  created QuickSwap, `quote.referral` is the referral fixed at creation.
- `FeeKind.REFERRAL`: the partner's commission, as a quote fee line per leg
  and as a `charged_fees` entry. It has no `account_id`.
- New `ErrorCode` values: `REFERRAL_NOT_SUPPORTED`, `REFERRAL_CODE_UNKNOWN`
  and `REFERRAL_PARTNER_INACTIVE`.
- `FeeSplit.referral_fee` (`QuickSwapBrokerService.GetBrokerStats`): the
  referral commissions charged in the period. They were never broker vault
  credits, so `affiliate_fee` does not include them, and `GetFeeTotals` and
  the `ListQuickSwaps` vault filter count only the broker's own vaults.

## How clients use it

Only a broker with a referral program may send a code; today that is
Polyester's own app. Every other broker gets `INVALID_ARGUMENT` /
`REFERRAL_NOT_SUPPORTED`, and its requests are unchanged otherwise.

```http
POST https://<api-host>/swap.quickswap.v1.QuickSwapService/QuoteQuickSwap
Authorization: Bearer <broker API key>
Content-Type: application/json

{"terms": {...}, "brokerBps": 20, "referralCode": "alice-fx"}
```

With a 20 bps broker fee on a normal swap with a 10 bps service fee, the
quote has `"referral": {"baseBrokerBps": 20, "discountBps": 3,
"commissionBps": 6}` and, per leg, a `SERVICE` line at 10 bps, a `BROKER`
line at 11 bps and a `REFERRAL` line at 6 bps. The user pays 27 bps instead
of 30. Show the base fee (service plus `base_broker_bps`) struck through,
next to the charged fee (that less `discount_bps`).

`CreateQuickSwap` takes the same field and fixes the referral at creation. A
retry returns that QuickSwap, whatever changed since. The code is part of the
request: the same `idempotency_key` with another code, or with and without
one, is `ALREADY_EXISTS` / `IDEMPOTENCY_CONFLICT`. Letter case doesn't count.

Each of these refusals binds nothing. Repeat the request without the code:

- the request also has affiliates: `INVALID_ARGUMENT` / `FEES_INVALID`;
- the code is not 4 to 16 letters, digits and hyphens: `INVALID_ARGUMENT` /
  `REFERRAL_CODE_INVALID`;
- no partner's current code matches it, for example a code the partner has
  replaced: `FAILED_PRECONDITION` / `REFERRAL_CODE_UNKNOWN`;
- the partner is not yet approved, or is disabled: `FAILED_PRECONDITION` /
  `REFERRAL_PARTNER_INACTIVE`;
- `broker_bps` is below the discount plus commission of every route:
  `INVALID_ARGUMENT` / `FEES_INVALID`. A route it does not cover is not
  offered.

## Migration

None. Every field and value is new. A request without `referral_code`
behaves and is compared exactly as before, so existing idempotency keys
resume their QuickSwaps. Clients that switch over `FeeKind` should handle
`REFERRAL`.
