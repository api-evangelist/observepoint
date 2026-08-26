---
name: observepoint-handle-completion-webhook
description: Receive, verify and act on an ObservePoint audit or journey completion webhook, including HMAC-SHA256 signature verification and secret rotation.
api: ObservePoint
apis:
  - ObservePoint V2 API
  - ObservePoint V3 API
  - ObservePoint Grid Reporting API
operations:
  - createWebAudit
  - updateWebAudit
  - getAuditRunInfo
  - getAuditRunScores
  - getGridData
generated: '2026-08-26'
method: generated
source: >-
  Grounded in https://api-docs.observepoint.com/sections/webhook (payload, signature scheme, rotation
  endpoint and the provider's own Python/JavaScript verifiers) plus openapi/observepoint-v2-api-openapi.yml
  and openapi/observepoint-v3-api-openapi.yml for the operationIds.
---

# Handle an ObservePoint completion webhook

ObservePoint POSTs a small JSON notification to an HTTPS URL you control when an audit or journey run
finishes — successfully **or not**. The payload is a pointer, not a result: it tells you *what* finished
so you can go and read it.

## Subscribe

Set `webHookUrl` on the audit or journey. Either:

- at creation — `POST /v2/web-audits` (`createWebAudit`) or `POST /v2/web-journeys`; or
- on an existing item — `PUT /v2/web-audits/{auditId}` (`updateWebAudit`) or
  `PUT /v2/web-journeys/{journeyId}`. Remember this is a full-object PUT: read first, then write.

Subscription is **per audit and per journey**. There is no account-wide subscription.

## Provision the signing secret (once, admin only)

`POST https://api.observepoint.com/v3/webhooks/rotate-secret` returns
`{ "sharedSecret": "string", "accountId": number }`.

- Requires **Admin** permission.
- The secret is shown **once**. Store it immediately.
- Calling it again rotates the account-wide secret **immediately**, with no dual-signing grace period.
  Every verifier must already hold the new secret or verification starts failing at once. This is
  irreversible — there is no way to recover the previous secret.
- This endpoint is documented in prose only; it does **not** appear in any of ObservePoint's published
  OpenAPI documents, so you will not find it by reading the spec.

## Verify every request

Header: `ObservePoint-Signature: t=<unix-timestamp>,sigv1=<base64 signature>`

1. Parse `t` and `sigv1` out of the header.
2. Build the signing string: `"<t>" + "." + <raw request body>`. Use the RAW bytes — not a re-serialised
   copy of the parsed JSON, or the signature will not match.
3. Base64-**decode** the stored `sharedSecret` and use those bytes as the HMAC key.
4. Compute HMAC-SHA256 over the signing string and compare, constant-time, against the base64-decoded
   `sigv1`.
5. On mismatch, respond `403` and process nothing.

ObservePoint publishes working Python and Node.js verifiers on the webhook page — use them rather than
writing your own.

> **Gap worth closing on your side:** the timestamp is signed, but ObservePoint publishes no tolerance
> window and its sample verifiers do not check the timestamp's age. Reject requests whose `t` is older
> than a few minutes if you want replay protection.

## Act on the payload

```json
{ "itemId": 12345, "itemType": "audit", "runId": 98765 }
```

- `itemType` is `"audit"` or `"web-journey"` — this is the only discriminator; the events are not named.
- With `itemId` + `runId`, call back into the API:
  - `getAuditRunInfo` — `GET /v3/web-audits/{auditId}/runs/{runId}/info` for the run header;
  - `getAuditRunScores` — `GET /v3/web-audits/{auditId}/runs/{runId}/scores` for the pass/fail signal;
  - `getGridData` — `POST /v3/reports/grid/{gridEntityType}` when you want the findings as a dataset.
- The webhook fires whether the run succeeded or failed, so check the run's status before treating an
  empty finding set as a clean result.

## Rules to obey

- Respond quickly and do the API callbacks asynchronously. ObservePoint publishes **no** retry, backoff
  or dead-letter behaviour for failed deliveries, so a delivery you drop may be the only one you get —
  keep a reconciliation poll (`getRuns`) as a backstop.
- Callbacks are subject to the normal per-key rate limit (100–1,000 rpm, `429` on exhaustion, no
  headers). A burst of simultaneous completions can rate-limit your own handler.
- Documented uses ObservePoint itself suggests: file unapproved cookies/tags into Jira, email a page
  details report, trigger a Tableau import, or post rule failures into Slack or Teams.
