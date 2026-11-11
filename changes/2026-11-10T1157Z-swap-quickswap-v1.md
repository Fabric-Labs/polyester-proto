---
services:
  - swap.quickswap.v1.QuickSwapReferralService
kind: added
breaking: false
wire_compatible: true
---

## What and why

Two new RPCs let a referral partner read the Quick Swaps its link brought in
and show how its commission was worked out, for Quick Swaps in progress and
done:

- `ListReferredQuickSwaps`: the caller's referred Quick Swaps, newest first,
  filtered by status and creation time, in keyset pages.
- `GetReferredQuickSwap`: one referred Quick Swap with a `ReferralBreakdown`:
  the fee rate without the referral (`base_fee_bps`), the rate the user pays
  (`charged_fee_bps`), the user's discount (`discount_bps`), the partner's
  commission rate (`commission_bps`) and its basis (`PRE_DISCOUNT_FEE`: a
  share of the fee before the discount), every commission credit, and the
  commission's US dollar value at the current price.

Each `ReferredQuickSwap` row carries the `swap_id`, status, `is_terminal`,
created and terminal times, the source and destination zipped assets, the
quoted amounts, the amounts received and delivered once known, the rates and
the commission credited so far per asset. It returns no identity of the
person who swapped and no account, address, memo, transaction hash or flow
ID. Amounts are exact; every time is rounded down to the minute.

New messages: `ReferredQuickSwap`, `ReferralCommission`, `ReferralCredit`,
`ReferralCommissionValue`, `ReferralBreakdown` and the request and response
of each RPC. New enum: `ReferralCommissionBasis`. `ErrorCode.SWAP_NOT_FOUND`
now also answers a `swap_id` the caller did not refer.

## How clients use it

Both RPCs take the caller's interactive session token and read only the
caller's own partner record; a broker API key is refused with
`PERMISSION_DENIED` / `ACTION_DENIED`, a person who has not enrolled gets
`NOT_FOUND` / `REFERRAL_PARTNER_NOT_FOUND`. ConnectRPC only; no REST binding.

```http
POST https://<api-host>/swap.quickswap.v1.QuickSwapReferralService/ListReferredQuickSwaps
Authorization: Bearer <session token>
Content-Type: application/json

{"statuses": ["EXECUTING", "COMPLETED"], "startTime": "2026-11-01T00:00:00Z", "limit": 50}
```

```http
POST https://<api-host>/swap.quickswap.v1.QuickSwapReferralService/GetReferredQuickSwap
Authorization: Bearer <session token>
Content-Type: application/json

{"swapId": "<swap_id from the list>"}
```

- Paging: `limit` 0 means 100, at most 200. Pass `next_page_token` back as
  `page_token` with the same filters; it is empty on the last page. Order is
  newest minute first; within a minute it is fixed but does not follow
  creation time. A Quick Swap whose creation completed after the first page
  was read may be missing from later pages.
- Times: `created_at`, `terminal_at` and `accrued_at` are rounded down to the
  minute. The window is whole minutes too: `start_time` (inclusive) is
  rounded down and `end_time` (exclusive) rounded up.
- Rates are whole basis points of what each trading leg receives, fixed when
  the Quick Swap is created. A two-leg route shows the sum of both legs'
  rates; a bridge has one rate.
- Amounts: `quoted_*` are estimates from creation; `source_amount` is set
  once a deposit is detected and `destination_amount` once a destination
  payout completes, both in their route's token units. Commission amounts
  are 18-decimal base units of the credited asset (`asset_id`).
- `commission_value.usd` is a decimal string, rounded down to 6 decimal
  places, valued now (not when credited); `as_of` is the oldest price used.
  It is unset when nothing is credited or a credited asset has no current
  price; the amounts are still returned.
- `credited` lists at most 100 credits, oldest first;
  `credited_truncated` says when there are more. The per-asset sums in
  `ReferredQuickSwap.commission` always cover every credit.
- A Quick Swap's `swap_id` is a 64-bit public ID, a decimal string in JSON.

## Migration

None: the RPCs and messages are new; existing RPCs are unchanged.
