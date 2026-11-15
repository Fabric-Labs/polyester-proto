---
services:
  - swap.quickswap.v1.QuickSwapBrokerService
kind: changed
breaking: false
wire_compatible: true
---

## What and why

An invite to a broker now holds only while the member who sent it may still
grant its role. A member may grant only a role whose actions it holds itself,
and only the Owner grants Finance. Before, an invite could still be accepted
after its sender left the broker, was given a smaller role, or handed over
ownership, and so grant a role that sender could no longer grant.

A change that takes the right away now cancels those invites at once, in the
same change: `SetMemberRole` on the sender, `RemoveMember` of the sender or
the sender leaving, and `TransferOwnership` by an Owner who sent Finance
invites. The cancelled invites leave `ListMyBrokers` and `ListMembers`, and
`RespondToInvite` answers `INVITE_NOT_FOUND` for them. Invites the sender's
new role may still grant stay pending.

`RespondToInvite` with `accept: true` also checks the sender's current role,
for any invite that is void without such a change (for example one sent
before the rule). It is refused with the new error code
`swap.quickswap.v1.ErrorCode.INVITE_VOID` (55). The invite grants nothing and
is cancelled, and a repeated call answers `INVITE_NOT_FOUND`. Declining an
invite is unchanged.

## How clients use it

`INVITE_VOID` is `FAILED_PRECONDITION` over ConnectRPC and `400` over REST,
with the code in the error detail and in the problem's `code`.

```http
POST /swap.quickswap.v1.QuickSwapBrokerService/RespondToInvite
{"inviteId": "...", "accept": true}

400 {"code": "failed_precondition", "message": "...", "details": [{"type": "swap.quickswap.v1.ErrorDetail", "value": "..."}]}
```

The `swap.quickswap.v1.ErrorDetail` detail carries `code: INVITE_VOID`.

```http
POST /v1/quickswap/invites/{invite_id}/respond
{"accept": true}

400 {"code": "INVITE_VOID", "status": 400, ...}
```

Tell the person the invite is no longer valid and to ask a broker member who
may grant the role for a new one. Refresh the invite list: the invite is
gone. After `SetMemberRole`, `RemoveMember` or `TransferOwnership`, refresh
`ListMembers` too: the member's invites it voided are gone.

## Migration

None: the code is additive. A client that does not know it sees an
unrecognized error code with `FAILED_PRECONDITION` and `400`; the invite
granted nothing.
