---
name: observepoint-privacy-and-consent-review
description: Pull the cookie, tag and request privacy findings for an ObservePoint audit run and reconcile them against the account's declared consent categories.
api: ObservePoint
apis:
  - ObservePoint V3 API
operations:
  - getAuditRunCookiePrivacyCompliance
  - getAuditRunTagPrivacyCompliance
  - getAuditRunRequestPrivacyCompliance
  - getCookies
  - getAuditRunCookieInventoryCookies
  - getAuditRunConsentCategories
  - getConsentCategoryLibrary
  - getConsentCategory
  - getConsentCategoryCookieEntries
generated: '2026-08-26'
method: generated
source: >-
  Grounded in openapi/observepoint-v3-api-openapi.yml (operationIds verbatim) plus
  https://api-docs.observepoint.com/ and https://www.observepoint.com/solutions/web-privacy/.
---

# Review privacy and consent findings for an audit run

This is ObservePoint's core privacy-assurance flow: an audit run has already observed every cookie, tag
and network request on a site, and you want the subset that violates what the account declared.

## Steps

1. **Read the declared standard.** `getAuditRunConsentCategories` —
   `GET /v3/web-audits/{auditId}/runs/{runId}/consent-categories` — returns the consent categories that
   were in force for THIS run. Use the run-scoped call — not the account-level
   `getConsentCategoryLibrary` (`GET /v3/consent-categories/library`) or
   `getConsentCategory` (`GET /v3/consent-categories/{consentCategoryId}`) — when you are explaining a
   finding: categories change over time and a past run was judged against the snapshot that applied then.
   Drill into a category's declared entries with `getConsentCategoryCookieEntries`
   (`GET /v3/consent-categories/{consentCategoryId}/cookies`), `getConsentCategoryTags` and
   `getConsentCategoryRequestDomains`.

2. **Cookie privacy.** `getAuditRunCookiePrivacyCompliance` —
   `POST /v3/web-audits/{auditId}/runs/{runId}/reports/cookie-privacy/compliance`. JSON body carries the
   filter; `size` and `sortBy` are query parameters. This is the "which cookies are non-compliant" answer.

3. **Tag privacy.** `getAuditRunTagPrivacyCompliance` —
   `POST /v3/web-audits/{auditId}/runs/{runId}/reports/tag-privacy/compliance`. Same shape. Tags that
   fired outside their approved consent category.

4. **Request privacy.** `getAuditRunRequestPrivacyCompliance` —
   `POST /v3/web-audits/{auditId}/runs/{runId}/reports/request-privacy/compliance`. Network requests that
   sent data to destinations outside the declared standard. Pair it with
   `getAuditRunRequestPrivacyLocations` when the geography of the destination matters.

5. **Get the full cookie inventory for context.** `getCookies` —
   `GET /v3/web-audits/{auditId}/runs/{runId}/cookies` for the simple list, or
   `getAuditRunCookieInventoryCookies` —
   `POST /v3/web-audits/{auditId}/runs/{runId}/reports/cookie-inventory/cookies` for the filterable
   report with attributes and page counts. A violation is only actionable once you know how many pages
   set the cookie.

6. **Optional — pull it as one dataset instead.** Every report above is also reachable through the Grid
   Reporting API as rows and columns, which is faster when you want to join across runs or audits. See
   `skills/observepoint-query-grid-reports.md`.

## Rules to obey

- `Authorization: api_key <YOUR_API_KEY>`; `Content-Type: application/json` on the POST reports.
- These are all POST-with-body READ operations. They are safe to retry — nothing here mutates state.
- Paginate with `size` (50–10,000) and `page` (zero-based); iterate until you have
  `metadata.pagination.totalCount` records. A truncated pull silently understates the violation count,
  which is the single most damaging mistake in this flow.
- `403` means the API key's user cannot see that folder or sub-folder, not that the audit is clean.
  Never report "no findings" on a `403`.
- Findings age out with the account's data-retention window (13 months by default).
