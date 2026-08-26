---
name: observepoint-query-grid-reports
description: Query any ObservePoint report as rows and columns using the Grid Reporting API, with filters, sorting, grouping and pagination.
api: ObservePoint
apis:
  - ObservePoint Grid Reporting API
operations:
  - getGridSchema
  - getGridData
  - createSavedReport
  - getSavedReportsList
  - deleteSavedReport
generated: '2026-08-26'
method: generated
source: >-
  Grounded in openapi/observepoint-grid-reports-api-openapi.yml (operationIds verbatim) plus
  https://api-docs.observepoint.com/sections/grid-api-intro, /grid-api-filters, /grid-api-sorting,
  /grid-api-grouping, /grid-api-pagination and /grid-api-examples.
---

# Query ObservePoint data with the Grid Reporting API

ObservePoint's own docs call this "the fastest and most flexible way to get ObservePoint results data",
and prefer it over the per-run report endpoints. Use it for anything analytical: tag inventories, cookie
prevalence, console-log sweeps, network-request inventories, journey failure lists.

## The one rule that trips everything else up

**Read the schema before you write the query.** Column ids are defined per grid entity type and are not
guessable. Always call `getGridSchema` first and pick `columnId` values from what it returns.

## Steps

1. **Discover columns.** `getGridSchema` — `GET /v3/reports/grid/{gridEntityType}/schema`.
   `gridEntityType` names the dataset (audit runs, web journey runs, pages, cookies, tags, network
   requests, console logs). The response gives you the available columns and their types
   (`string`, `number`, `timestamp`, `entity_reference`).

2. **Query.** `getGridData` — `POST /v3/reports/grid/{gridEntityType}` with a JSON body. The body carries
   the columns you want, the filters, the sort, the grouping and the pagination — nothing goes in the
   query string.

   Filter conditions have the shape:

   ```json
   {
     "filters": {
       "conditions": [
         { "filteredColumn": { "columnId": "FINAL_PAGE_URL" },
           "operator": "string_contains",
           "arg": "example.com" }
       ]
     }
   }
   ```

   The operator set is deliberately small — match the operator to the column TYPE from step 1:

   | Operator | Column types |
   | --- | --- |
   | `string_contains`, `string_regex`, `string_contains_multi` | `string` |
   | `integer_in`, `number_between` | `number` |
   | `date_time_between`, `date_time_relative` | `timestamp` |
   | `is_present` | all |
   | `integer_list_contains` | `entity_reference` arrays |

   `string_contains` alone covers exact match, starts-with, ends-with and contains — reach for
   `string_regex` only when you genuinely need a pattern.

3. **Paginate.** Put `page` (zero-based) and `size` (10–10,000) in the request BODY, not the query string —
   this differs from the v2/v3 report endpoints, where they are query parameters and the minimum is 50.
   The response carries `metadata.pagination` with `totalCount` and `totalPageCount`. Datasets reach
   billions of rows, so always set `size` explicitly rather than accepting the default.

4. **Group when you want counts, not rows.** Turn on group mode and use the aggregate functions to
   summarise — e.g. cookie prevalence across a domain — instead of pulling every row and counting locally.

5. **Save a configuration you will reuse.** `createSavedReport` — `POST /v3/reports/grid/saved`;
   list them with `getSavedReportsList` — `GET /v3/reports/grid/saved`; remove one with
   `deleteSavedReport` — `DELETE /v3/reports/grid/saved/{id}`. A saved report is a stored query, so a
   scheduled job can pull the same dataset without re-encoding the filter tree.

## Rules to obey

- `Authorization: api_key <YOUR_API_KEY>` on every request; `Content-Type: application/json` on POST.
- **`deleteSavedReport` is irreversible.** There is no restore path for saved reports — unlike audits and
  journeys, which can be undeleted. Confirm before deleting.
- **No idempotency key exists.** `createSavedReport` retried after a timeout can create a duplicate;
  list first with `getSavedReportsList` before retrying.
- Rate limits are per API key, 100–1,000 rpm, and nothing in the response tells you how much is left.
  On `429`, back off for seconds up to 5 minutes.
- Time-bounded filters (`date_time_relative`) beyond the account's retention window (13 months by
  default) return nothing, because the underlying results data has been deleted.
