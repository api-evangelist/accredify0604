---
name: accredify0604-verify-a-document
description: Verify an Accredify-issued or third-party verifiable document and extract its verifiable data keys, using the Accredify Nexus Verification module.
api: Accredify Nexus API
base_url: https://nexus.accredify.io
generated: '2026-09-06'
method: generated
source: openapi/accredify0604-nexus-verification-openapi.yaml, openapi/accredify0604-nexus-auth-openapi.yaml
operations:
  - verify-document
  - extract-keys
---

# Verify a document with Accredify

The Verification module is the smallest and most self-contained surface Accredify publishes:
two operations, one scope, no pagination, no identifiers to track.

## 1. Get a token

`POST /oauth/token` on `https://nexus.accredify.io` with `grant_type=client_credentials`,
your `client_id` and `client_secret`, and:

```
scope=verification-suite
```

`verification-suite` is the **only** scope either verification operation requires. If you
are building a verifier and nothing else, mint a client that holds only this scope.

## 2. Verify the document

`verify-document` — `POST /verification/v1/verify`. Scope: `verification-suite`.

The request body is the verifiable document itself. The spec's own example is an
OpenAttestation 2.0 wrapped document (`"version":
"https://schema.openattestation.com/2.0/schema.json"`) carrying an OpenCerts transcript
schema, with salted `data` fields and an `issuers[]` block using a `DNS-DID` identity proof
and a `did:ethr:` controller.

Practical consequence: **pass the document through unmodified.** The salted
`uuid:string:value` encoding and the `signature` block are what the verification is
performed against; re-serialising or normalising the JSON will change the document's proof
and it will fail verification for reasons that have nothing to do with its validity.

Responses: `200` on a completed verification, `422` when the payload is not a document the
module can process. Note that a `200` means the *verification ran* — read the result body
for the verdict; do not treat the status code as the answer.

## 3. Extract verifiable data keys

`extract-keys` — `POST /verification/v1/extract-keys`. Scope: `verification-suite`.

Returns the verifiable data keys from the document, which is the supported path for
**verified data extraction** — pulling specific attested fields out of a document once its
integrity has been established. Responses: `200`, `422`, `500`.

Run `verify-document` **before** `extract-keys`. Extracting fields from a document you have
not verified gives you data with no attestation behind it, which defeats the purpose of the
credential.

## Standards this surface touches

Accredify's live OID4VCI issuer metadata at
`https://nexus.accredify.io/.well-known/openid-credential-issuer` declares `mso_mdoc`
credential configurations for ISO/IEC 18013-5 mDL (`org.iso.18013.5.1.mDL`) and ISO/IEC
23220 photo ID (`org.iso.23220.photoid.1`). See
`conformance/accredify0604-conformance.yml` for what is evidenced by a fetched artifact and
what is only a prose claim.

## Errors

`422` returns `{message, errors}`. `500` on `extract-keys` is declared; there is no
documented rate limit or 429 on this surface.
