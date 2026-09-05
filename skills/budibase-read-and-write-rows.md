---
name: budibase-read-and-write-rows
description: >-
  Search, read, create, update and delete rows in a Budibase table through the Public API,
  including cursor pagination and the structured query language. Use for any agent task
  that moves data in or out of Budibase.
api: Budibase Public API
base_url: https://budibase.app/api/public/v1
operations:
  - tableSearch
  - tableGetById
  - rowSearch
  - rowGetById
  - rowCreate
  - rowUpdate
  - rowDestroy
  - rowViewSearch
generated: '2026-09-04'
method: generated
source: openapi/budibase-public-api-openapi.yml
---

# Read and write Budibase rows

## Before you start

Every operation here needs TWO headers:

- `x-budibase-api-key` — your key.
- `x-budibase-app-id` — the workspace/app ID. Omit it and you get
  `403 This request required a workspace id.` or `400 Invalid app ID provided`.

Get the workspace ID from the builder URL after `/builder/workspace/`. Remove the `_dev_`
segment to address published data instead of development data.

## Discover the shape first

Rows are schemaless in the contract — `row` is an object with `additionalProperties` of any
type. You cannot know a row's fields from the OpenAPI. So:

1. `POST /tables/search` (`tableSearch`) with `{ "name": "<partial name>" }` to find the
   table. The match is case-insensitive starts-with.
2. `GET /tables/{tableId}` (`tableGetById`) and read `schema` — that object is the real
   column definition, including `type: link` columns which are relationships to other
   tables. Build your field expectations from this, at runtime, every time.

## Search rows

`POST /tables/{tableId}/rows/search` (`rowSearch`). The `query` object supports typed
operators — use the right one rather than filtering client-side:

- `string` — starts-with match, a map of column name to value.
- `fuzzy` — substring match (searching `dib` matches `Budibase`).
- `range` — bounded numeric or date ranges.
- `equal` / `notEqual`, `empty` / `notEmpty`, `oneOf`, `contains` / `notContains` /
  `containsAny`.
- `allOr: true` switches the whole query from AND to OR. Default is AND.

Sort with `sort`, `sortOrder` and `sortType`.

## Paginate correctly

Send `paginate: true` and a `limit`. The response carries:

- `data` — the rows, each with `_id`.
- `bookmark` — the cursor for the next page. It is typed `oneOf [string, integer]`, so
  handle both.
- `hasNextPage` — loop while this is true, passing the previous `bookmark` back in.

Only `rowSearch` and `rowViewSearch` paginate this way. `queryExecute` uses a different
`pagination: { page, limit }` model, and the other search operations return an unpaged
`data[]` with no documented ceiling — do not assume you have seen everything.

## Write

- **Create:** `POST /tables/{tableId}/rows` (`rowCreate`) with the row body matching the
  table schema.
- **Update:** `PUT /tables/{tableId}/rows/{rowId}` (`rowUpdate`). This is a full-document
  write, not a patch. Read the row with `rowGetById` first and send the complete object, or
  you will drop fields.
- **Delete:** `DELETE /tables/{tableId}/rows/{rowId}` (`rowDestroy`).

## Rules that apply throughout

- **10 requests per second.** Bulk loads are slow by construction — there is no batch row
  endpoint. Pace against `x-ratelimit-remaining`; there is no `Retry-After`.
- **No idempotency key.** A retried `rowCreate` creates a second row. Before retrying a
  timed-out create, `rowSearch` for it.
- **Row deletion and row update are both irreversible.** There is no trash, no soft delete,
  no restore endpoint, and `rowUpdate` does not return the prior value. If the work must be
  reversible, export the app first (`POST /applications/{appId}/export`).
- **Errors carry no code.** The body is `{ "message": <string>, "status": <int> }` with no
  machine-readable identifier — branch on the HTTP status, and treat 400 and 401 as the
  same auth outcome. The 404 body is plain text `Not Found`, not JSON, so guard your parse.
