---
name: budibase-provision-workspace-app
description: >-
  Create a Budibase workspace and application, verify it, and publish it — the standard
  provisioning flow through the Budibase Public API. Use when an agent must stand up a new
  Budibase app rather than operate an existing one.
api: Budibase Public API
base_url: https://budibase.app/api/public/v1
operations:
  - workspaceCreate
  - workspaceGetById
  - appCreate
  - appGetById
  - appPublish
  - appUnpublish
  - workspacePublish
  - workspaceUnpublish
generated: '2026-09-04'
method: generated
source: openapi/budibase-public-api-openapi.yml
---

# Provision a Budibase workspace and application

## Before you start

- You need an API key from the Budibase portal user dropdown ("View API key"). Send it as
  `x-budibase-api-key` on every request. Generating a new key invalidates the old one —
  there is no overlap window.
- The key inherits the RBAC role of the user who created it. Provisioning requires
  Admin or Builder. A lesser role returns `403 Workspace Admin/Builder user only endpoint.`
- Base URL is `https://budibase.app/api/public/v1` for Budibase Cloud. For a self-hosted
  instance substitute your own host and keep the `/api/public/v1` suffix.

## Steps

1. **Create the workspace.** `POST /workspaces` with `{ "name": "<name>", "url": "<url-encoded-path>" }`
   (`workspaceCreate`). The response carries `_id`. Keep it — this ID is what every later
   data call sends in the `x-budibase-app-id` header.

2. **Create the application.** `POST /applications` (`appCreate`) with the same body shape.
   Note the returned `_id` and `status`, which will be `development`.

3. **Confirm before you act further.** `GET /applications/{appId}` (`appGetById`) and check
   `status`, `version` and `updatedAt`. There is no dry-run mode anywhere in this API, so
   reading state back is the only rehearsal available.

4. **Publish.** `POST /applications/{appId}/publish` (`appPublish`). The published copy is
   addressed by the same ID with the `_dev_` segment removed: `app_dev_XXX` in development
   becomes `app_XXX` when published. Two IDs, one logical app — pick deliberately.

5. **If you need to undo, unpublish.** `POST /applications/{appId}/unpublish`
   (`appUnpublish`) is a true inverse and there is no time limit on it. It takes the live
   app down; it does not roll back to an earlier version.

## Rules that apply throughout

- **Rate limit: 10 requests per second.** The API returns `x-ratelimit-limit`,
  `x-ratelimit-remaining` and `x-ratelimit-reset` on every response, including errors.
  There is no `Retry-After` header, so pace against `x-ratelimit-remaining` yourself.
- **No idempotency.** There is no `Idempotency-Key` header. If `appCreate` times out, do
  NOT blind-retry — call `appSearch` with the name first, or you will create a duplicate.
- **Auth failures come back as 400, not 401.** `{"message":"Invalid API key provided...",
  "status":400}`. Do not key your retry logic on 401 alone.
- **Deletion is permanent.** `DELETE /applications/{appId}` has no undo and no trash. If
  you may need to reverse it, call `POST /applications/{appId}/export` first and keep the
  artifact — that export/import pair is the only rollback path the API offers.
