---
name: accredify0604-issue-openbadges-batch
description: Issue 1EdTech Open Badges and certificate batches through the Accredify Dashboard API — OAuth grant, batch upload, issue, and email dispatch.
api: Accredify Dashboard API
base_url: https://dashboard.accredify.io/api
generated: '2026-09-06'
method: generated
source: openapi/accredify0604-dashboard-v1-openapi.yaml, openapi/accredify0604-dashboard-v2-openapi.yaml
operations:
  - authorizationCodeGrant
  - refresh
  - issueOpenBadges
  - createOpenBadgesCourse
  - updateOpenBadge
  - uploadBatch
  - uploadBatchExcel
  - uploadIssueBatch
  - createBatch
  - issueBatch
  - getBatches
  - getBatchById
  - getBatchItems
  - sendEmails
---

# Issue Open Badges and certificate batches (Dashboard API)

The Dashboard API is Accredify's **second** API family and it is not a variant of Nexus: it
uses a different auth model, a different error envelope and integer ids instead of UUIDs.
It is also the only family with a published Open Badges surface.

Two versions run side by side with no published deprecation date for v1:

- **v1** — `https://dashboard.accredify.io/api` — OAuth, Open Badges, batch upload, revocation.
- **v2** — `https://dashboard.accredify.io/api/v2` — batches, batch items, courses, documents, templates.

## 1. Get a token

Unlike Nexus, this is an **authorization-code** flow, not client credentials.

- `redirect` — `GET /v1/oauth/redirect` sends the user to authorize your application.
- `authorizationCodeGrant` — `POST /v1/oauth/grant` exchanges the code returned to your
  registered callback for an access token.
- `refresh` — `POST /v1/oauth/refresh` refreshes an expired token.

Present the result as `Authorization: Bearer <jwt>`. No scopes are declared on this family.

## 2. Open Badges

- `createOpenBadgesCourse` — `POST /v1/openbadges/new-course` creates the course (the
  achievement) a badge is issued against.
- `issueOpenBadges` — `POST /v1/openbadges/issue` issues badges to recipients.
- `updateOpenBadge` — `PATCH /v1/openbadges/{assertion}` updates an issued badge.
- `revokeOpenBadge` — `DELETE /v1/openbadges/{assertion}` revokes it (`204`).

`{assertion}` is the **Open Badges assertion identifier**. Store it at issue time; it is the
only handle these three operations accept.

## 3. Batches

Two routes, and choosing the wrong one is the most common mistake here.

**Two-step (v2, preferred)** — you get to inspect before anything is issued:

1. `createBatch` — `POST /batches` (`201`).
2. `getBatchById` / `getBatchItems` — check `status` and the item list.
3. `issueBatch` — `POST /batches/{id}/issue`.

**One-step (v1)** — `uploadIssueBatch` — `POST /v1/uploadIssueBatch` uploads **and issues**
in a single call. There is no review step and no idempotency key. Use it only when the
payload is already validated upstream.

Upload-only v1 variants: `uploadBatch` (`POST /v1/uploadBatch`) and `uploadBatchExcel`
(`POST /v1/uploadBatchExcel`) for spreadsheet input.

## 4. Deliver

`sendEmails` — `POST /v1/sendEmails` triggers recipient emails for issued certificates. A
batch also carries a `subscribers[]` array of email addresses for status notifications.

Email templates are read-only through the API: `GET /email-templates` and
`GET /email-templates/{id}` (v2).

## 5. Undo

- Before issuance: `deleteBatch` — `DELETE /batches/{id}`, allowed only while the batch is
  **not** `issuing`, `revoking`, `issued` or `revoked`, and only for the owner.
- After issuance: `revokeBatchItem` — `DELETE /batch-items/{id}` (a reason is required), or
  `revokeCerts` — `POST /v1/revokeCerts` by certificate hash.

See `accredify0604-revoke-a-credential.md`.

## Conventions that apply

- **Errors:** `{status, message}`, with `errors` (v2) or `error` (v1) on 422 and a `code`
  on 500 — quote that code to `support@accredify.io`. This differs from Nexus.
- **Rate limits:** every Dashboard operation declares `429`. The message example says
  *"Please retry again after 60 seconds"*, but **no `Retry-After` or `RateLimit-*` header
  is published** and no numeric limit is documented. Back off blind.
- **Pagination (v2):** `page` + `per_page` (`10|25|50|100`). `GET /courses` additionally
  accepts `per_page=-1` to return everything unpaginated.
- **Sandbox:** the specs name `https://dashboard.uat.accredify.io/api` as "Accredify
  Sandbox (UAT)". It is reachable but there is no self-service sign-up — see
  `sandbox/accredify0604-sandbox.yml`. Ignore the `dashboard.local.accredify.io` server
  entry; it is a developer loopback left in the published spec.
