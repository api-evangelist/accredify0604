---
name: accredify0604-issue-credentials-via-workflow
description: Issue verifiable documents in bulk through an Accredify Nexus workflow — authenticate with client credentials, read the workflow's own input schema, trigger a run, and collect the issued documents.
api: Accredify Nexus API
base_url: https://nexus.accredify.io
generated: '2026-09-06'
method: generated
source: openapi/accredify0604-nexus-workflow-openapi.yaml, openapi/accredify0604-nexus-auth-openapi.yaml
operations:
  - get-workflows
  - get-workflow-by-id
  - trigger-workflow-run
  - get-workflow-runs
  - get-workflow-run-by-id
  - resume-workflow-run
  - get-documents
  - download-workflow-run-documents
---

# Issue credentials through an Accredify Nexus workflow

Issuance in Nexus is not a single "create credential" call. A **workflow** is the issuance
pipeline an Accredify customer has configured, and you run it with a batch of document
payloads. **The payload shape is defined by the workflow, not by the API** — this is the one
thing that trips up an integration written from the spec alone.

## 1. Get a token

`POST /oauth/token` on `https://nexus.accredify.io`, as
`application/x-www-form-urlencoded`, with `grant_type=client_credentials`, `client_id`,
`client_secret` and a space-separated `scope`.

The provider's own instruction is: *"Request only the scopes required by your integration,
separated by spaces."* For this flow that is:

```
run-workflow workflows:read workflow-runs:read documents:read
```

A success returns `{token_type: "Bearer", expires_in: 31536000, access_token: "..."}`. Note
`expires_in` is a **year** — cache the token, do not re-mint it per request. Present it as
`Authorization: Bearer <access_token>`.

## 2. Find the workflow and read its input schema

- `get-workflows` — `GET /api/v1/workflows` (paginated: `page`, `per_page`).
- `get-workflow-by-id` — `GET /api/v1/workflows/{workflow}` where `{workflow}` is the UUID.

The detail response carries `latest_version.api_schema`, with `schema` and
`flattened_schema`. **Read this before building any payload.** The spec says the document
object structure "varies by workflow. Check workflow details for specific schema
requirements." There is no global document schema to code against.

## 3. Trigger the run

`trigger-workflow-run` — `POST /api/workflows/{workflow}/runs`. Scope: `run-workflow`.

> **Path gotcha:** this operation is **not** under `/api/v1/`, while
> `get-workflow-runs` (`GET /api/v1/workflows/{workflow}/runs`) is. Do not derive the POST
> path from the GET path.

Body:

```json
{
  "batch_name": "Graduation ceremony 2026",
  "documents": [ { /* shape per the workflow's api_schema */ } ]
}
```

`documents` is required; `batch_name` is optional but set it — it is how the run is found
and searched later.

> **This writes real credentials, and Accredify publishes no idempotency mechanism.** There
> is no `Idempotency-Key` header on any Accredify operation. If the request times out, do
> **not** blind-retry: call `get-workflow-runs` and check whether a run with your
> `batch_name` already exists.

## 4. Track the run

- `get-workflow-runs` — `GET /api/v1/workflows/{workflow}/runs`
- `get-workflow-run-by-id` — `GET /api/v1/workflow-runs/{run}`

Poll the run's `status` / `status_label`. `current_action_sequence_number` and
`workflow_actions_list` show which step of the pipeline it is on. There is no webhook and no
event surface — polling is the only published mechanism.

If a run reaches a paused state, `resume-workflow-run` — `POST /api/v1/workflow-runs/{run}/resume`
continues it. There is no corresponding pause operation in the published API.

## 5. Collect the documents

- `get-documents` — `GET /api/v1/documents`, filtered by workflow UUID and/or workflow-run.
- `download-workflow-run-documents` — `GET /api/v1/workflow-runs/{run}/documents/download`
  returns every document from the run as a ZIP.

Each `Document` carries `uuid`, `status`, `issued_at`, `share_link`, a `recipient`
(`name`, `email`), a `files[]` array of `{url, label}`, and `external_reference_id` — your
own key, which you should set so you never need to store Accredify UUIDs.

## Conventions that apply

- **Pagination:** `page` (default 1) and `per_page` (default 10); responses carry
  `data` + `links` + `meta`.
- **Errors:** Nexus returns `{message}`, plus an `errors` object on 422. This is **not**
  the Dashboard API's `{status, message}` envelope. See
  `errors/accredify0604-problem-types.yml`.
- **403 means scope, not identity.** The token is valid; it lacks the scope the operation
  needs. See `scopes/accredify0604-scopes.yml`.
- **Rate limits:** the Nexus specs declare no 429 and no rate-limit headers. Back off on
  any 5xx and treat throughput as unknown — see `rate-limits/accredify0604-rate-limits.yml`.
