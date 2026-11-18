---
services:
  - chain.lifecycle.v1.LifecycleReadService
  - ledger.read.v1.StatementExportService
  - rewards.v1.RewardCampaignService
kind: changed
breaking: true
wire_compatible: true
---

## What and why

A request whose body is over the size limit now gets the same answer from
account flow reads, statement exports and reward campaigns as from every
other API:

- a ConnectRPC, gRPC or gRPC-web call answers the ConnectRPC code
  `resource_exhausted`;
- a REST request answers HTTP 413 with the problem `code`
  `REQUEST_TOO_LARGE`.

What changes, by credential:

- **API key, every protocol.** These APIs answered HTTP 413 with a body RPC
  clients cannot parse, so ConnectRPC, gRPC and gRPC-web clients decoded it
  as `unknown`. Over REST, statement exports and reward campaigns answered
  plain text with no problem body, and account flow reads answered the
  problem `code` `payload_too_large`.
- **Session, REST.** `POST /v1/statements/exports` answered HTTP 413 with the
  problem `code` `request_too_large` in lower case. Reward campaigns'
  `PUT /v1/rewards/awards/{award_id}/destination` and
  `POST /v1/rewards/awards/{award_id}/acknowledge` answered HTTP 400 with
  `INVALID_ARGUMENT`.

Session ConnectRPC, gRPC and gRPC-web calls already answered
`resource_exhausted` and are unchanged. The account flow REST reads take no
request body, so a session request to them is unchanged. No message, field
or enum value changed.

## How clients use it

```http
POST /chain.lifecycle.v1.LifecycleReadService/ListAccountFlows
HTTP/1.1 429 Too Many Requests

{"code":"resource_exhausted","message":"request too large"}
```

```http
POST /v1/statements/exports
HTTP/1.1 413 Request Entity Too Large

{"type":"https://polyester.dev/problems/request-too-large","title":"Request Entity Too Large","status":413,"detail":"request too large","code":"REQUEST_TOO_LARGE"}
```

ConnectRPC answers `resource_exhausted` with HTTP 429 by its own protocol
mapping; branch on the code, not the status or the message, whose text
differs between API-key and session calls. The limits are unchanged: 64 KiB
for account flow reads, 1 KiB for statement exports and 128 KiB for reward
campaigns.

## Migration

1. Where a REST client of `GET /v1/chain/account-flows` or
   `GET /v1/chain/account-flows/{flow_id}` matches the problem `code`
   `payload_too_large`, match `REQUEST_TOO_LARGE` instead.
2. Where a REST client of `POST /v1/statements/exports` matches the problem
   `code` `request_too_large`, match `REQUEST_TOO_LARGE` instead.
3. Where a REST client of the reward campaign destination or acknowledge
   writes treats an oversized body as HTTP 400 `INVALID_ARGUMENT`, expect
   HTTP 413 `REQUEST_TOO_LARGE` instead.
4. Where an API-key ConnectRPC client of these three services detects an
   oversized request by matching `unknown`, match `resource_exhausted`
   instead.

In every case, do not retry the request unchanged: send a smaller one. Other
REST clients that branch on HTTP 413 need no change.
