---
name: observepoint-update-audit-configuration
description: Read an ObservePoint audit's configuration, change it (for example its starting URLs or webhook destination), and write it back safely.
api: ObservePoint
apis:
  - ObservePoint V2 API
operations:
  - getAllWebAudits
  - getWebAudit
  - updateWebAudit
  - createWebAudit
  - getWebAuditAvailableLocations
  - getWebAuditAvailableFrequencies
  - getWebAuditAvailableUserAgents
  - getWebAuditRules
  - updateWebAuditRules
generated: '2026-08-26'
method: generated
source: >-
  Grounded in openapi/observepoint-v2-api-openapi.yml (operationIds verbatim) plus
  https://api-docs.observepoint.com/sections/api-recipes/update-starting-urls,
  https://api-docs.observepoint.com/sections/choosing-api-endpoints and
  https://api-docs.observepoint.com/sections/webhook.
---

# Update an ObservePoint audit's configuration

Audit and journey CONFIGURATION lives on the v2 paths even though reporting lives on v3 — ObservePoint's
own "Choosing API Endpoints" guide points here for creating and updating audits. v2 is fully supported
with no planned deprecation.

## The read-modify-write rule

`updateWebAudit` is a **PUT**, not a PATCH. It replaces the audit configuration with the body you send.
Always `getWebAudit` first, mutate the object you got back, and send the whole thing. Sending a partial
body silently drops every field you omitted — starting URLs, rules, schedule, webhook destination.

## Steps

1. **Locate the audit.** `getAllWebAudits` — `GET /web-audits` (or `getAudits` —
   `GET /domains/{domainId}/web-audits` to scope to one sub-folder). Note: in v2 a "domain" is what the
   UI and the v3 API call a **sub-folder**. It is not a DNS domain.

2. **Read the current configuration.** `getWebAudit` — `GET /web-audits/{webAuditId}`.
   Optional `withRuns` / `runsLimit` query parameters if you also want recent runs.

3. **Check the allowed values before you set them.** Scan locations, frequencies and user agents are
   enumerated by the API, not free text:
   - `getWebAuditAvailableLocations` — `GET /web-audits/locations`
   - `getWebAuditAvailableFrequencies` — `GET /web-audits/frequencies`
   - `getWebAuditAvailableUserAgents` — `GET /web-audits/user-agents`

4. **Modify the object.** Typical changes:
   - starting URLs, for the bulk "keep many audits in sync" case;
   - `webHookUrl`, to receive an HTTPS notification when the run completes;
   - schedule, limits, or blocking configuration.

5. **Write it back.** `updateWebAudit` — `PUT /web-audits/{webAuditId}` with
   `Content-Type: application/json` and the FULL object.

6. **Rules are a separate surface.** `getWebAuditRules` — `GET /web-audits/{webAuditId}/rules` and
   `updateWebAuditRules` — `PUT /web-audits/{webAuditId}/rules`. Changing the audit does not change its
   rules and vice versa.

7. **Creating instead of updating.** `createWebAudit` — `POST /web-audits`. There is no idempotency key,
   so a retry after a timeout can create a duplicate audit. Before retrying, call `getAllWebAudits` and
   check whether the audit you were creating already exists.

## Rules to obey

- **You cannot mutate an item while it is running.** `423` with
  `This item has active run. Please stop & discard active run first` means exactly that — either wait,
  or `stopAuditRun` (`DELETE /web-audits/{webAuditId}/runs/{runId}`) first.
- **`422` is validation**, and the body names the problem (`Empty name`,
  `Some of provided folder id's is not found`). Fix the payload rather than retrying.
- **`409` on rules is a name collision** (`Rule name is not unique`).
- **Deleting an audit is recoverable, but only partly.** `deleteWebAudit` archives it, and
  `undeleteWebAudit` (`PATCH /v3/web-audits/undelete`) restores the CONFIGURATION into a folder and
  sub-folder you choose. The original parents are not reinstated and **historical run data is not
  restored**. ObservePoint publishes no time limit on restoring, so do not promise one.
- No dry-run mode exists. There is no way to preview a configuration change before applying it — read
  first, diff locally, then write.
