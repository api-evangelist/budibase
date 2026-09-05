---
name: budibase-manage-users-and-roles
description: >-
  Provision Budibase users and grant or revoke their roles across apps through the Public
  API. Use for onboarding/offboarding automation and access reviews.
api: Budibase Public API
base_url: https://budibase.app/api/public/v1
operations:
  - userSearch
  - userGetById
  - userCreate
  - userUpdate
  - userDestroy
  - roleAssign
  - roleUnAssign
generated: '2026-09-04'
method: generated
source: openapi/budibase-public-api-openapi.yml
---

# Manage Budibase users and roles

## Before you start

Requires an API key belonging to a user with global Admin or Builder rights. Budibase has
no service accounts and no scoped keys — the key you use carries a real person's full
privileges, so use a dedicated Budibase user for automation rather than a staff member's.

## Find a user before creating one

`POST /users/search` (`userSearch`). Email is unique, so search on it first — there is no
idempotency key and a blind `userCreate` retry on a timeout will fail or duplicate work.

## Create a user

`POST /users` (`userCreate`). Fields worth knowing:

- `email` — required and unique.
- `password` — write-only. It is never returned by any read operation.
- `forceResetPassword: true` — makes the user set their own password at first login. Prefer
  this to holding a password in your automation.
- `builder`, `admin` — global privilege flags. `builder` can only be set on a Business or
  Enterprise plan.
- `roles` — per-app role map.

## Grant and revoke access

`POST /roles/assign` (`roleAssign`) is the one genuinely batch-shaped operation in this
API: it takes a `userIds[]` array and applies one change to all of them. The body selects
what to grant:

- `role: { roleId, appId }` — a per-app role such as `BASIC`, `ADMIN`, or a custom role ID.
- `appBuilder: { appId }` — app-builder privileges on one app.
- `builder: true` / `admin: true` — global privileges.

`POST /roles/unassign` (`roleUnAssign`) takes the same body shape and reverses it. This is
the cleanest inverse pair in the Budibase API and it has no time window — a revoke works
whenever you call it.

## Offboarding

For a leaver, prefer `roleUnAssign` over `userDestroy`:

- `roleUnAssign` is reversible and leaves the audit trail intact.
- `DELETE /users/{userId}` (`userDestroy`) is permanent. There is no restore endpoint and
  no undelete window. The only recovery is a workspace backup, which exists on Premium and
  above only (7-day retention on Premium, unlimited on Business and Enterprise).

## Rules that apply throughout

- **10 requests per second.** An access review across many users will need pacing; read
  `x-ratelimit-remaining` off each response.
- **API keys are per-user and privileges are inherited.** Deleting or demoting the user who
  owns an automation's API key breaks that automation.
- **A missing or invalid key returns 400, not 401.** Handle both as auth failure.
- **No idempotency.** Search before you create; `roleUnAssign` before you delete.
