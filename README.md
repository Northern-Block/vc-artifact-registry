# vc-artifact-registry

Static file store for Verifiable Credential artifacts published by eAPI's `credential` service, served publicly via GitHub Pages. Wallets, issuers, and verifiers resolve schemas, JSON-LD contexts, VCT definitions, and revocation status lists from URLs in this repo.

## ⚠️ Immutability rule — read before touching anything

**Once a file is published here, it must never be edited, moved, renamed, or deleted** (the one exception is `status/` files — see below).

Every file's GitHub Pages URL (`https://{GITHUB_PAGES_DOMAIN}/...` — domain root, custom domain, **no** `/vc-artifact-registry` segment) is permanently embedded inside credentials already issued to holders' wallets — in the credential's `credentialSchema`, `@context`, VCT `id`/`schema`/`jsonLdContext`, and `bitStringStatusListURL` fields. A wallet cannot be updated after issuance. Moving or changing a published file breaks every credential that references it, with no way to fix it after the fact.

If a schema or artifact needs to change, publish a **new** file (new version, new timestamp suffix) — never overwrite an existing one.

## Structure

```
vc-artifact-registry/
├── core/
│   └── vocab/
│       └── vocab.jsonld
│           # ONE shared JSON-LD vocabulary for the whole repo, not per-environment,
│           # not per-LOB. Referenced by fragment from context files, e.g.:
│           #   https://{GITHUB_PAGES_DOMAIN}/core/vocab/vocab.jsonld#country
│           # `core/` is the container for ANY environment-agnostic content —
│           # not just vocab. Rule: a top-level folder in this repo is either an
│           # environment name, or `core`. Nothing else goes at the root.
│
├── dev/                                          ← environment (from GITHUB_PATH_ENVIRONMENT)
│   └── orga-adarsh-otp-tenant1/                  ← lobEcoSystemName (tenant-chosen at LOB registration)
│       └── insurance-lob/                        ← lobDisplayName
│           │
│           ├── schema/
│           │   └── insurance-policy-1.0.json
│           │       # BASE schema. One per schema+version. No timestamp — this is the
│           │       # canonical schema definition (attributes, governance URL). Created
│           │       # once when a schema is first published; re-published only on a new
│           │       # schemaVersion.
│           │
│           ├── context/
│           │   └── insurance-policy-1778856680321-1.0.jsonld
│           │       # JSON-LD @context. Maps each schema attribute name to a semantic
│           │       # IRI/term (uses core/vocab/vocab.jsonld + external vocabularies). This is
│           │       # what a JSON-LD credential's "@context" array points to.
│           │
│           ├── w3cSchema/
│           │   └── insurance-policy-1778856680321-1.0.json
│           │       # W3C JSON Schema (draft-07 style). Defines the actual JSON shape/
│           │       # validation rules for the credentialSubject. This is what a verifier
│           │       # validates a received credential's payload against.
│           │
│           ├── vct/
│           │   └── insurance-policy-1778856618754-1.0.json
│           │       # "Verifiable Credential Type Definition" — NOT the credential itself,
│           │       # and not a schema. It's a small manifest/pointer document that ties
│           │       # everything else together: it holds
│           │       #   - id: this file's own GitHub Pages URL
│           │       #   - schema: {$ref to the base schema AND the w3cSchema file above}
│           │       #   - jsonLdContext: pointer to the context/ file above
│           │       #   - propertyBindings: which schema field maps to which JSON-LD term
│           │       #   - bitStringStatusListURL: pointer to the status/ file below
│           │       # Issuers/verifiers fetch the VCT first, then follow its pointers to
│           │       # the schema/context/status files. Think of it as the "index card"
│           │       # for one issued credential type — one per cred-def, not per schema.
│           │
│           └── status/
│               └── InsurancePolicy-revocation-1778856680321.json
│                   # BitString Status List credential — a single growing bitstring where
│                   # bit N = revoked/suspended flag for the Nth credential issued under
│                   # this cred-def. A verifier fetches this and checks one bit to know if
│                   # a specific credential has been revoked. One per cred-def (only
│                   # created if supportRevocation is enabled). THIS is the one file type
│                   # that legitimately gets updated in place over time (bits flip as
│                   # credentials are revoked) — it is not a versioned snapshot like the
│                   # other four artifact types.
│
├── qa/       └── {lobEcoSystemName}/{lobDisplayName}/schema|context|w3cSchema|vct|status/...
├── prod/     └── {lobEcoSystemName}/{lobDisplayName}/schema|context|w3cSchema|vct|status/...
├── sandbox/  └── {lobEcoSystemName}/{lobDisplayName}/schema|context|w3cSchema|vct|status/...
├── stg/      └── {lobEcoSystemName}/{lobDisplayName}/schema|context|w3cSchema|vct|status/...
└── cs-prd/   └── {lobEcoSystemName}/{lobDisplayName}/schema|context|w3cSchema|vct|status/...
```

### Quick reference: what each artifact type is for

| Folder | Describes | Created | Mutability |
|---|---|---|---|
| `schema/` | Data *shape*, loosely typed | Once per schema version | Immutable |
| `w3cSchema/` | Data *shape*, W3C-strict (what a verifier validates payloads against) | Once per JSON-LD schema publish | Immutable |
| `context/` | *Meaning* — JSON-LD term mappings | Once per JSON-LD schema publish | Immutable |
| `vct/` | *Manifest* — pointers to schema + w3cSchema + context + status, plus property bindings | Once per credential definition | Immutable |
| `status/` | *Live revocation state* for one cred-def | Once per cred-def (only if revocation enabled) | **Mutable** — bits flip as credentials are revoked |

### Top-level: environment

The top-level folder is the deployment environment that published the artifact. It comes from the `credential` service's `GITHUB_PATH_ENVIRONMENT` env var — by convention one of `dev`, `qa`, `prod`, `sandbox`, `stg`, `cs-prd`, though this is not enforced by an enum in code; whatever string is configured for a given deployment is what gets written.

### `core/`

`core/` holds anything environment-agnostic — content that's identical across dev/qa/prod and shouldn't be duplicated per environment. Today that's just `core/vocab/vocab.jsonld`, a shared JSON-LD vocabulary referenced by term fragments (e.g. `vocab.jsonld#country`, `#locality`, `#religion`) from generated `@context` files — but any future global asset belongs here too, not at the repo root.

`core/vocab/vocab.jsonld` is a byte-identical duplicate of `northern-block/test-jsonld/vocab.jsonld`, carried over with the same term fragments so that credentials issued against the old repo and the new repo resolve to the same semantic terms.

### Per-LOB folders

Below the environment, structure is unchanged from the legacy `northern-block` layout: `{lobEcoSystemName}/{lobDisplayName}/{artifactType}/{fileName}`. `lobEcoSystemName` and `lobDisplayName` are tenant-supplied at LOB registration; `artifactType` is one of the five described above.

## Publishing

There is no build step here — this repo *is* the GitHub Pages content. Files land under `https://{GITHUB_PAGES_DOMAIN}/<path>` (domain root, no `/vc-artifact-registry` segment) as soon as they're merged to `main`.

**Requires a `CNAME` file + custom domain configured in this repo's Settings → Pages** for `GITHUB_PAGES_DOMAIN` to actually resolve — not yet set up as of this writing. Until that's done, generated URLs will not be reachable; do not let `credential` publish real artifacts against this repo before it is.


## Who writes here

The `credential` service (`orbit-enterpriseapi-client/apps/credential`) is the only writer, via `GithubService` (`orbit-enterpriseapi-internal/packages/shared-enum/src/github`). Path construction lives in `JSONLdContextUtils` (`apps/credential/src/{v1,v2}/schema/jsonld-context.utils.ts`).

Relevant config in `credential`'s `.env`:

```
GITHUB_OWNER=<owner>
GITHUB_REPO=vc-artifact-registry
GITHUB_BRANCH=main
GITHUB_TOKEN=<token>
GITHUB_PATH_ENVIRONMENT=<dev|qa|prod|sandbox|stg|cs-prd>   # required — app fails to start if unset
```
