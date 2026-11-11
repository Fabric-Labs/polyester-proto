---
services:
  - swap.quickswap.v1.QuickSwapReferralService
kind: added
breaking: false
wire_compatible: true
---

## What and why

A new service, `swap.quickswap.v1.QuickSwapReferralService`, lets a person
become a Quick Swap referral partner and manage their own referral code.
Quick Swaps made through a partner's link on Polyester's own Quick Swap app
give the user a discount and pay the partner a commission, both out of the
broker fee; the rates are whole basis points per route category
(`ReferralProgram`: normal swaps, stablecoin swaps, bridging). Partners are
invite-only: an enrollment starts pending until Polyester approves it.

- `EnrollReferralPartner`: apply with one of the caller's own active
  sub-accounts, which is credited with the commissions, and a display name.
  The partner starts `PARTNER_PENDING` with a random 8-character code.
- `GetReferralPartner`: the caller's partner record: ID, display name,
  status, earnings sub-account, current code, share link path and the rates.
- `CheckReferralCode`: whether the caller may take a code
  (`ReferralCodeAvailability`).
- `SetReferralCode`: replace the code with a vanity code. The old code is
  retired and never given to anyone again.

New messages: `ReferralRates`, `ReferralProgram`, `ReferralPartner` and the
request and response of each RPC. New enums: `ReferralPartnerStatus` and
`ReferralCodeAvailability`. New `ErrorCode` values: `REFERRAL_CODE_TAKEN`,
`REFERRAL_CODE_RESERVED`, `REFERRAL_CODE_INVALID`,
`REFERRAL_PARTNER_NOT_FOUND`, `REFERRAL_PARTNER_EXISTS`,
`EARNINGS_ACCOUNT_INVALID`, `REFERRAL_PARTNER_DISABLED` and
`REFERRAL_CODE_LIMIT`.

## How clients use it

Every RPC takes the caller's interactive session token and acts only on the
caller's own record; a broker API key is refused with `PERMISSION_DENIED` /
`ACTION_DENIED`, and API-key sessions are refused. The service is ConnectRPC
only; it has no REST binding.

```http
POST https://<api-host>/swap.quickswap.v1.QuickSwapReferralService/EnrollReferralPartner
Authorization: Bearer <session token>
Content-Type: application/json

{"earningsSubaccountId": "<public sub-account ID>", "displayName": "Alice"}
```

```http
POST https://<api-host>/swap.quickswap.v1.QuickSwapReferralService/SetReferralCode
Authorization: Bearer <session token>
Content-Type: application/json

{"code": "Alice-2026"}
```

The response carries the partner with `"code": "alice-2026"` and
`"linkPath": "/r/alice-2026"`; the share link is that path on Polyester's
Quick Swap app.

- Codes are 4 to 16 letters, digits and hyphens, starting and ending with a
  letter or digit, without two hyphens in a row; letters are compared and
  stored in lower case. Anything else is `INVALID_ARGUMENT` /
  `REFERRAL_CODE_INVALID`.
- Another partner's current code is `ALREADY_EXISTS` /
  `REFERRAL_CODE_TAKEN`; a brand or broker name, or any code used before
  (including the caller's own retired codes), is `FAILED_PRECONDITION` /
  `REFERRAL_CODE_RESERVED`.
- Setting the current code again changes nothing. A disabled partner cannot
  change its code (`FAILED_PRECONDITION` / `REFERRAL_PARTNER_DISABLED`).
- A partner holds at most 10 codes over its lifetime, the one it enrolled
  with included: at most 9 changes. `ReferralPartner.code_changes_remaining`
  says how many are left; at 0 a new code is `FAILED_PRECONDITION` /
  `REFERRAL_CODE_LIMIT` and the current code stays.
- Every RPC is rate limited per account: over the limit it fails with
  `RESOURCE_EXHAUSTED`, a `polyester.ratelimit.v1.RateLimitDetail` and a
  `Retry-After` header; retry after that delay.
- Repeating `EnrollReferralPartner` with the same details returns the same
  partner; other details are `ALREADY_EXISTS` / `REFERRAL_PARTNER_EXISTS`.
  A sub-account that is not the caller's, not active, or used by another
  partner is `FAILED_PRECONDITION` / `EARNINGS_ACCOUNT_INVALID`.
- Before enrolling, every RPC but `EnrollReferralPartner` returns
  `NOT_FOUND` / `REFERRAL_PARTNER_NOT_FOUND`.
- `ReferralPartner.program` is unset while no referral program runs.

## Migration

None: the service is new. The new `ErrorCode` values appear only on the new
service's RPCs.
