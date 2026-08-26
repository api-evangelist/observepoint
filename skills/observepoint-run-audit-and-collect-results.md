---
name: observepoint-run-audit-and-collect-results
description: Trigger an ObservePoint web audit on demand, wait for the run to finish, and pull its scores and page-level results.
api: ObservePoint
apis:
  - ObservePoint V2 API
  - ObservePoint V3 API
operations:
  - getAllWebAudits
  - runAuditNow
  - getRuns
  - getRun
  - getAuditRunInfo
  - getAuditRunScores
  - getAuditRunPages
  - stopAuditRun
generated: '2026-08-26'
method: generated
source: >-
  Grounded in openapi/observepoint-v2-api-openapi.yml and openapi/observepoint-v3-api-openapi.yml
  (every operationId above is verbatim from those specs) plus https://api-docs.observepoint.com/,
  /sections/rate-limiting and /sections/webhook.
---

# Run an ObservePoint audit and collect its results

Use this when something outside ObservePoint should trigger a scan — a deploy, a release gate, a CI job —
and then act on what the scan found.

## Before you start

- Base URL: `https://api.observepoint.com`. Configuration lives on `/v2` paths, reporting on `/v3`.
- Auth on every request: `Authorization: api_key <YOUR_API_KEY>`. Keys are per USER, so the run only sees
  what that user can see. A missing header returns `401` with the plain-text body `Bearer token is absent`.
- `POST`/`PUT` bodies are `application/json`. `PATCH` bodies are `application/json-patch+json`.
- There is **no idempotency key**. `runAuditNow` is not safe to blind-retry — see step 2.

## Steps

1. **Find the audit.** `getAllWebAudits` — `GET /web-audits`. Optional `withRuns` and `runsLimit` query
   parameters return recent runs inline, which saves a round trip in step 3. Match on the audit name you
   care about and keep its `id` as `webAuditId`.

2. **Start the run.** `runAuditNow` — `POST /web-audits/{webAuditId}/runs`. No request body.
   - This consumes subscription run quota.
   - If a run is already in flight you get `409` (`Journey is already running` on the journey equivalent),
     or `423` with `This item has active run. Please stop & discard active run first`. Treat both as
     "already running" — do **not** retry into them, poll instead.
   - On a network timeout with no response, do **not** re-POST. Call `getRuns` first and look for a run
     created in the last few seconds; a duplicate POST starts a second billed run.

3. **Poll until the run completes.** `getRuns` — `GET /web-audits/{webAuditId}/runs` — then
   `getRun` — `GET /web-audits/{webAuditId}/runs/{runId}` — for the specific run. Poll on a backoff of
   30s or more; audits crawl real websites and take minutes to hours.
   - Prefer a webhook to polling if you control an HTTPS endpoint: set `webHookUrl` on the audit and
     ObservePoint POSTs `{ itemId, runId, itemType }` on completion, signed with `ObservePoint-Signature`.
     See `asyncapi/observepoint-webhooks.yml`.

4. **Read the run header.** `getAuditRunInfo` — `GET /v3/web-audits/{auditId}/runs/{runId}/info`
   (`includeFilters` optional). `auditId` here is the same integer as `webAuditId` in step 2 — the two
   API versions just name the path parameter differently.

5. **Read the scores.** `getAuditRunScores` — `GET /v3/web-audits/{auditId}/runs/{runId}/scores`. This is
   the headline pass/fail signal to gate a pipeline on.

6. **Read the pages.** `getAuditRunPages` — `GET /v3/web-audits/{auditId}/runs/{runId}/pages`.
   Paginate with `size` (50–10,000, use 100) and `page` (zero-based). Stop when the records you have
   collected equal `metadata.pagination.totalCount`. Optional `sortBy`.

7. **Abort if you need to.** `stopAuditRun` — `DELETE /web-audits/{webAuditId}/runs/{runId}` stops and
   discards an in-flight run. Once the run has completed this returns `409`
   `The audit run is already stopped or completed` — completion is the point of no return.

## Rules to obey

- **Rate limits.** 100–1,000 requests per minute per API key depending on endpoint. On `429`, wait and
  retry — usually seconds, occasionally up to 5 minutes. There are **no** `RateLimit-*` or `Retry-After`
  headers, so you cannot see your remaining budget; back off blind. Rate-limited requests are not billed.
- **Errors.** `application/json`, not RFC 9457. Two shapes: `{ timestamp, message, details, validationReport }`
  and `{ errorCode, message }`. `403` almost always means the key's user lacks folder access, not that
  the object is missing. See `errors/observepoint-problem-types.yml`.
- **Data retention.** Results data older than the account's retention window (13 months by default) is
  gone. Do not expect old runs to still answer.
