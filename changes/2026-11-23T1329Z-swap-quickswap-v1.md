---
services:
  - swap.quickswap.v1.QuickSwapService
  - swap.quickswap.v1.QuickSwapReferralService
kind: changed
breaking: false
wire_compatible: true
---

## What and why

A referral partner's old code now keeps working after the partner changes
it with `SetReferralCode`. `QuoteQuickSwap` and `CreateQuickSwap` accept a
`referral_code` that the partner has since replaced, and credit the same
partner at the referral program's current rates, exactly as the current
code does. Links a partner already shared, and codes a visitor's browser
saved from them, keep earning after a code change.

Before, a replaced code failed with `FAILED_PRECONDITION` /
`REFERRAL_CODE_UNKNOWN`. That error now means only that no partner ever had
the code.

Unchanged:

- A replaced code stays reserved for good. No partner can take it
  (`SetReferralCode` and `CheckReferralCode` still answer
  `REFERRAL_CODE_RESERVED` / `CODE_RESERVED`), so it can never credit
  anyone else.
- A partner can still change its code at most 9 times
  (`ReferralPartner.code_changes_remaining`, `REFERRAL_CODE_LIMIT`).
- `GetReferralPartner` shows only the current code and its link.
- A partner that is not active is still refused with
  `REFERRAL_PARTNER_INACTIVE`, whichever of its codes the request carries.

The documentation of `referral_code`, `REFERRAL_CODE_UNKNOWN` and
`SetReferralCode` now says this. No field, message, enum value or error
code was added or changed.

## How clients use it

No new call. Send the code from the referral link as before, current or
replaced:

```http
POST /swap.quickswap.v1.QuickSwapService/QuoteQuickSwap
Content-Type: application/json

{"terms": {...}, "brokerBps": 20, "referralCode": "old-code"}
```

The REST routes are unchanged: `POST /v1/quickswaps:quote` and
`POST /v1/quickswaps`. The quote's `referral` carries the program's current
rates for the route. The code is part of the create request as sent, so the
same `idempotency_key` with the old and then the new code is still
`IDEMPOTENCY_CONFLICT`.

## Migration

None: requests that used to succeed still do. A client that dropped a code
on `REFERRAL_CODE_UNKNOWN` can keep doing so; that error now only follows
a code no partner ever had. A partner-facing app that warned that the old
code stops working after a change should say that links with the old code
keep working.
