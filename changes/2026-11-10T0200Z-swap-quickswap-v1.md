---
services:
  - swap.quickswap.v1.QuickSwapService
kind: changed
breaking: false
wire_compatible: true
---

## What and why

Documentation only: no message, field, enum value, RPC or behavior changes.
The contract now states plainly that a QuickSwap quote's output and fees are
estimates, not fixed amounts.

A QuickSwap is priced when its deposit is credited, which can be hours or
days after the quote: the deposit must be detected before
`funding_expires_at`, and its confirmation, credit and execution can finish
later. Each trading fee charged is its rate (`bps`) applied to what the leg
actually receives at that time. The fee charged and the output follow the
actual deposit, execution prices and rounding, so they can be higher or
lower than the quoted `QuoteFee.amount` and `expected_amount_out`. For
example, a sell that fills at a better price than quoted can be charged a
fee above the quoted line while the end user also receives more than
quoted. The rate is what is fixed.

`QuoteFee.amount` used to say that the charged fee never exceeds the quoted
one. That held only when the swap executed exactly at the quoted prices, and
it is removed. The updated comments:

- `QuoteFee` and `QuoteFee.amount`: the amount is an estimate and not a cap;
  `QuoteFee.bps` is the rate that is charged.
- `FeeKind.DEPOSIT_NETWORK`: the quote shows an estimate; the fee withheld
  is the network's fee when the deposit arrives.
- `QuickSwapQuote`, `QuickSwapQuote.expected_amount_out` and
  `QuickSwapQuote.fees`: prices, amounts and fees are estimates at
  `quoted_at`; only the protection fields (`min_amount_out`, `min_rate`)
  are guarantees.
- `QuickSwap.quote` and `QuickSwap.charged_fees`: the quote fixed at creation
  stays an estimate; `execution`, `withdrawals` and `charged_fees` report the
  actual output and fees.
- `QuoteQuickSwapResponse.quote`, the `QuoteQuickSwap` RPC comment and its
  OpenAPI description: the quote's output and fees are estimates.

## How clients use it

Quote and create QuickSwaps as before:

```http
POST https://<api-host>/swap.quickswap.v1.QuickSwapService/QuoteQuickSwap
Content-Type: application/json

{"terms": {...}, "brokerBps": 20, "affiliates": [{"accountId": "...", "bps": 10}]}
```

REST: `POST https://<api-host>/v1/quickswaps:quote`.

Show quoted fees and `expected_amount_out` as estimates, for example
"≈ 0.25 USDT (0.10%)", and show the rate (`bps`) as the fixed part. Show what
was actually charged and delivered from the QuickSwap
(`GET /v1/quickswaps/{swap_id}` or `GetQuickSwap`): `charged_fees` for the
broker and affiliate fees, `execution.output_amount` and each withdrawal's
`amount` and `network_fee`. Treat only `min_amount_out` and `min_rate` as
guarantees.

## Migration

None for code: no field or behavior changes. Clients that label quoted fees
or the expected output as final amounts, or that check the charged fee
against the quoted fee, should label them as estimates and compare the
charged fee with the rate instead.
