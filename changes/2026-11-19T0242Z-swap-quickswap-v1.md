---
services:
  - swap.quickswap.v1.QuickSwapBrokerService
kind: changed
breaking: false
wire_compatible: true
---

## What and why

Documentation only; no message, field, enum or route changes. The REST
reference for `GET /v1/quickswap/brokers/{broker}/quickswaps/{swap_id}`
(`QuickSwapBrokerService.GetQuickSwap`) described its success response as an
empty object. The API has always answered the broker's QuickSwap, so the
reference now documents that body: `{"quickSwap": {...}}`, with the same fields
`QuickSwapService.GetQuickSwap` documents: its quote, deposit instructions,
deposits, execution progress, withdrawals and fees. Like every QuickSwap REST
body it is the ProtoJSON of the response: 64-bit integers, such as `swapId`,
`accountId` and `version`, are decimal strings; 32-bit integers, such as the
asset IDs `sourceZippedAssetId`, `destinationZippedAssetId`,
`refundZippedAssetId` and `chargedFees[].assetId`, are JSON numbers; enums are
their protobuf names; timestamps are RFC 3339 strings; and fields with a zero
value are omitted.

## How clients use it

Read one of the broker's QuickSwaps with a broker API key or a member's
session token:

```text
GET https://<api-host>/v1/quickswap/brokers/acme/quickswaps/18239957911229591281
-> {"quickSwap": {"swapId": "18239957911229591281", "status": "AWAITING_DEPOSIT",
    "terms": {...}, "depositInstructions": {...}, "chargedFees": [...], ...}}

POST https://<api-host>/swap.quickswap.v1.QuickSwapBrokerService/GetQuickSwap
{"broker": "acme", "swapId": "18239957911229591281"}
```

Another broker's swap answers `SWAP_NOT_FOUND`, like an unknown ID.

## Migration

None: the answer on the wire is unchanged. REST clients generated from the
reference now get a typed `quickSwap` instead of an empty response type;
code that read the body untyped keeps working.
