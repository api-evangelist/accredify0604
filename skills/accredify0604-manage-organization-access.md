---
name: accredify0604-manage-organization-access
description: Manage Accredify Nexus organisation users, groups, roles and API tokens — including minting and revoking the long-lived user tokens that carry scoped API access.
api: Accredify Nexus API
base_url: https://nexus.accredify.io
generated: '2026-09-06'
method: generated
source: openapi/accredify0604-nexus-auth-openapi.yaml
operations_note: >-
  The Accredify Nexus Auth module publishes no operationId on any of its eleven operations,
  so there is nothing to cite. Operations are addressed by method and path.
operations: []
operations_by_path:
  - POST /oauth/token
  - GET /api/v1/organization/users
  - POST /api/v1/organization/users
  - GET /api/v1/organization/users/{user_uuid}
  - PATCH /api/v1/organization/users/{user_uuid}
  - GET /api/v1/organization/groups
  - POST /api/v1/organization/groups/{group_uuid}/duplicate
  - GET /api/v1/organization/roles
  - GET /api/v1/organization/users/{user_uuid}/user_tokens
  - POST /api/v1/organization/users/{user_uuid}/user_tokens
  - POST /api/v1/organization/user_tokens/{token_uuid}/revoke
---

# Manage organisation access in Accredify Nexus

The Auth module governs who exists in the organisation and what credentials they hold.

> **Note on operationIds:** the eleven operations in the Auth module specification carry no
> `operationId`. The names in this skill's frontmatter are descriptive labels; **address
> every operation by method and path**, as written below. A generated client for this
> module will produce positional method names that change between spec revisions.

## Scopes

From `POST /oauth/token`, request only what you need:

| Task | Scope |
|---|---|
| Read users | `users:read` |
| Create / update users | `users:write` |
| Read groups | `groups:read` |
| Read roles | `roles:read` |

See `scopes/accredify0604-scopes.yml` for all 20 published scopes.

## Users

- `GET /api/v1/organization/users` — paginated (`page`, `per_page`).
- `POST /api/v1/organization/users` — create (`201`).
- `GET /api/v1/organization/users/{user_uuid}` — read one.
- `PATCH /api/v1/organization/users/{user_uuid}` — partial update.

A `User` carries `uuid`, `name`, `email`, `role`, `status` and `groups[]`. There is **no
delete-user operation** in the published API; deactivation runs through `status` on the
PATCH, and locking a user auto-revokes their tokens (below).

## Groups and roles

- `GET /api/v1/organization/groups` — paginated.
- `POST /api/v1/organization/groups/{group_uuid}/duplicate` — clone a group with its
  configuration (`201`).
- `GET /api/v1/organization/roles` — the role catalogue.

The **group is the tenancy boundary** in Nexus. `Workflow`, `Document`, `DesignTemplate`,
`DocumentTemplate` and `Course` all belong to a group — see
`data-model/accredify0604-data-model.yml`.

## User tokens

This is the sharp edge of the module.

- `GET /api/v1/organization/users/{user_uuid}/user_tokens` — token **metadata** only,
  paginated.
- `POST /api/v1/organization/users/{user_uuid}/user_tokens` — create a token.
- `POST /api/v1/organization/user_tokens/{token_uuid}/revoke` — revoke one.

Three things to hold on to:

1. **The plaintext bearer token is returned exactly once**, on create
   (`CreatedUserTokenSecret`). It cannot be read back from the list operation. Capture it at
   creation or re-mint.
2. **Override the default scopes** by passing `scopes` as a JSON array on create. Left
   alone, the token inherits defaults you did not choose.
3. **Tokens are revoked automatically when the subject user is locked**, in addition to the
   manual revoke. Locking a user is therefore also an access-revocation event, and an
   integration running under that user's token will start failing with `401`.

Revoke takes the **user-token context UUID** (`{token_uuid}`), not the user UUID.

## Errors

`{message}`, plus an `errors` object on `422`. A `403` means the token is valid but the
scope is missing — `401` means the token itself is bad or expired. See
`errors/accredify0604-problem-types.yml`.
