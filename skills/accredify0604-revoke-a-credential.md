---
name: accredify0604-revoke-a-credential
description: Revoke an issued Accredify credential — a Nexus document, a Dashboard batch item, an Open Badge, or a certificate by hash — and understand which revocations can be taken back (none of them).
api: Accredify Nexus API, Accredify Dashboard API
generated: '2026-09-06'
method: generated
source: openapi/accredify0604-nexus-workflow-openapi.yaml, openapi/accredify0604-dashboard-v1-openapi.yaml,
  openapi/accredify0604-dashboard-v2-openapi.yaml
operations:
  - revoke-document
  - revokeBatchItem
  - revokeOpenBadge
  - revokeCerts
  - delete-workflow-run
  - deleteBatch
---

# Revoke an Accredify credential

> **Read this first.** Accredify's spec states plainly of `revoke-document`:
> *"This action cannot be undone."* There is **no un-revoke operation anywhere in the
> published contract** — not in Nexus, not in Dashboard v1, not in v2. Confirm the target
> identifier before you call any operation on this page.

Which operation you need depends on which system issued the credential. Nexus and Dashboard
do **not** share an identifier space.

## Nexus: revoke a document

`revoke-document` — `POST /api/v1/documents/{document}/revoke`. Scope: `documents:write`.

`{document}` is the document UUID. Body:

```json
{ "reason": "Document contains incorrect information" }
```

`reason` is **required** and capped at 200 characters. It is recorded on the document as
`revocation_reason` alongside `revoked_at`.

If you hold your own key rather than the Accredify UUID, resolve it first:
`get-documents-by_recipient` — `GET /api/v1/documents/by_recipient`, or `get-documents`
(`GET /api/v1/documents`) filtered by workflow / workflow run, then match on
`external_reference_id`.

## Dashboard v2: revoke a batch item

`revokeBatchItem` — `DELETE /batch-items/{id}` on `https://dashboard.accredify.io/api/v2`.

Revokes the batch item **and its associated documents**, and requires a revocation reason.
Covers both internal certificates and badges. `{id}` is an integer, not a UUID.

## Dashboard v1: revoke an Open Badge

`revokeOpenBadge` — `DELETE /v1/openbadges/{assertion}` on
`https://dashboard.accredify.io/api`.

`{assertion}` is the Open Badges assertion identifier, not an Accredify id. Returns `204`.

## Dashboard v1: revoke certificates by hash

`revokeCerts` — `POST /v1/revokeCerts`.

Revokes certificates **by certificate hash**, and also supports removal of the certificate
from MySF. This is the operation to use when what you hold is the credential's hash rather
than any Accredify-side identifier.

## Delete vs revoke — the two that have a real window

Deletion is different from revocation, and these are the only two Accredify operations that
state a bound in the spec:

| Operation | Path | Stated window |
|---|---|---|
| `deleteBatch` | `DELETE /batches/{id}` | Only while the batch is **not** in an `issuing`, `revoking`, `issued` or `revoked` status, and only for the batch owner. |
| `delete-workflow-run` | `DELETE /api/v1/workflow-runs/{run}` | Only while the run has issued **no** documents **and** is in `failed`, `paused`, `cancelled` or `rejected` status. Requires the `run-workflow` scope. |

Once a batch has issued, deletion is closed and revocation is the only route left. That is
the decision point: **delete before issuance, revoke after.**

## Errors

- `404` — wrong identifier, or wrong identifier *space* (a Nexus UUID sent to a Dashboard
  integer path).
- `403` — the token lacks `documents:write` (Nexus) or the caller does not own the batch
  (Dashboard).
- `422` — a missing or over-length `reason`, or a state precondition not met.

See `conventions/accredify0604-conventions.yml` (`reversibility`) and
`errors/accredify0604-problem-types.yml`.
