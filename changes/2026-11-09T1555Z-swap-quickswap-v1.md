---
services:
  - swap.quickswap.v1.QuickSwapService
kind: added
breaking: false
wire_compatible: true
---

## What and why

QuickSwap execution progress now includes individual execution slices. Each entry
shows its planned time, first order time, finish time, state, reason, traded input,
net output and average price. Two-leg routes also show the second-leg conversion,
intermediate funds carried forward and the reason a conversion could not proceed.
Market executions have one slice; bridge-only swaps have none. Refused planned
slices remain visible with their reasons. `execution.startedAt` is unset until a
slice is first submitted; planned times and deadline do not imply submission.
The deadline stays tied to the frozen planning window even when submission is later.

If recorded slice accounting is inconsistent, `execution.slices` is omitted and
`execution.slicesUnavailableReason` is `ACCOUNTING_INCONSISTENCY`. Aggregate
amounts, counts, status and withdrawals remain available.

`PRECISION_UNAVAILABLE` means the source, destination or two-leg refund asset's
precision is unknown. The history is omitted instead of guessing zero decimals.
Status, counts and withdrawals still progress; affected aggregate amounts remain
unknown, while aggregates whose precision is known remain available.

## How clients use it

Call GetQuickSwap as usual, then read `execution.slices`:

```http
POST /swap.quickswap.v1.QuickSwapService/GetQuickSwap
Content-Type: application/json

{"swapId":"1234567890123456789"}
```

The list is ordered by zero-based index and contains at most 50 entries.
`execution.slicesTruncated` signals that additional entries exist. Aggregate
amounts and counts always cover the whole execution. When the list is available
and not truncated, its input and output amounts sum exactly to `executedInput` and
`outputAmount`; entries with `completed: true` count to `completedSlices`.
Handle unavailable history explicitly, for example:

```json
{"execution":{"totalSlices":6,"completedSlices":3,"slicesUnavailableReason":"ACCOUNTING_INCONSISTENCY"}}
```

The same response shape can report `"slicesUnavailableReason":"PRECISION_UNAVAILABLE"`.
Show the known aggregate progress and withdrawals while explaining that slice
details are unavailable. Do not treat an unavailable list as zero execution.
Amounts use the same token units as the aggregate fields, with cumulative
rounding across slices. Average prices are decimal strings rounded down to
18 places. A two-leg slice's output can include conversion of funds carried
from earlier slices.

Display `PLANNED` and `EXECUTING` as pending states; `TRADED`, `PARTLY_TRADED`,
`SKIPPED` and `FAILED` are final outcomes. Use the typed `reason` and
`carryReason` values to explain incomplete execution. Times are timestamps;
`executedAt` is absent when no order was sent, and `finishedAt` is absent until
the slice ends. The REST call `GET /v1/quickswaps/{swap_id}` exposes the same fields.

## Migration

The call signature is unchanged. Refresh generated types for the new history and
unavailability enum. `startedAt` now reflects the first actual slice submission;
it can be absent even when execution history exists. Show planned times and
reasons for refused slices. Check `slicesUnavailableReason` before using the list,
and do not interpret a truncated or unavailable list as the full history.
