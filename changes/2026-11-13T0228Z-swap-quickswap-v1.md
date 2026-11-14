---
services:
  - swap.quickswap.v1.QuickSwapService
kind: changed
breaking: false
wire_compatible: true
---

## What and why

A broker can now have a fee ceiling agreed with Polyester: an upper bound on
Polyester's service fee for that broker's QuickSwaps. Every trading leg's
`SERVICE` fee is the leg's pair fee capped by that ceiling, and a bridge's
`SERVICE` fee is the bridge fee capped the same way. The ceiling never limits
the broker or affiliate fees, and their 300 bps cap is unchanged.

A QuickSwap keeps the ceiling in force when it was created, exactly as it
keeps its `fee_schedule_version`: a later change to the ceiling never
re-prices an existing QuickSwap, and repeating a create returns the original
QuickSwap with its original fees.

Only comments change: `FeeKind.SERVICE`, `QuickSwap.fee_schedule_version` and
`ReferralRates`. No message, field or enum value changes.

## How clients use it

Nothing new to call. Quotes and QuickSwaps already report the rate charged
in each `SERVICE` fee line's `bps`; under a ceiling it is the capped rate.

```http
POST /swap.quickswap.v1.QuickSwapService/QuoteQuickSwap
{"terms": {...}, "brokerBps": 5}
```

```json
{"quote": {"fees": [
  {"kind": "SERVICE", "zippedAssetId": 58, "amount": {"baseUnits": "325400", "decimals": 6}, "bps": 4},
  {"kind": "BROKER", "zippedAssetId": 58, "amount": {"baseUnits": "406750", "decimals": 6}, "bps": 5}
]}}
```

## Migration

None: no wire or behavior change for clients. A client that assumed the
`SERVICE` rate equals the published pair or bridge fee should read the rate
from the fee line's `bps`, as it already carries the rate charged.
