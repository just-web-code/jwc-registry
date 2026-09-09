# jwc-registry — Roadmap

Sprint-by-sprint plan that grew out of the jwc-lang Sprint 9-11
"Registry server" lane. Each sprint is sized for one focused session.

## Sprint Tracker

| # | Sprint | Status | Notes |
|---|--------|--------|-------|
| R1 | Skeleton + axum + healthz | ✅ | Cargo workspace, Dockerfile, docker-compose, `/healthz`, env-driven `Config`. |
| R2 | Google OAuth + JWT sessions | ✅ | Auth-code flow, userinfo upsert, signed JWT, Bearer extractor. |
| R3 | Package model + upload/download/list/delete | ✅ | Postgres schema, blob store, multipart upload, owner-only delete. |
| R4 | jwc-lang client integration (`jwc publish` / `jwc login`) | ✅ | `jwc-lang/src/registry.rs`. Verified against this server, not a stub — see below. |
| R5 | Operations (rate limit, OTel, backup) | ⬜ | `governor` per-IP, `tracing-opentelemetry`, pg-dump cron, Grafana board. |
| R6 | Production deploy on registry-jwc.1kb.uz | ⏳ | Deployed via musanna-soft/k8s-gitops (apps/jwc-registry) + ArgoCD; DNS pointing to cluster ingress, Let's Encrypt via cert-manager. |

## R4 — what was verified, and against what

`jwc-lang/tests/registry.rs` drives the client against a **stub** that
jwc-lang wrote from its own reading of this contract. That is two
implementations, each tested against its own idea of the other, so the
whole client surface was run against this server on 2026-09-09
(jwc-lang 0.9.952, this repo at `ddfd4ce`):

| Command | Result |
|---|---|
| `jwc login --token jwc_… --registry …` | writes `~/.jwc/credentials.json`, keyed by registry URL |
| `jwc publish` | 997 bytes uploaded, sha256 `2bad40e3…`, stored under that digest |
| `GET /api/v1/pkg/redis` | lists the version with the same checksum and `size_bytes` |
| `jwc add redis` | downloads, checks the digest, vendors to `jwc_packages/redis`, records `^0.2.0` |
| `jwc install` | re-fetches a deleted vendor directory |
| `jwc tree` | `consumer └── redis 0.2.0` |
| `jwc update` | `0.2.0 -> 0.2.1` after a second publish, and rewrites the range |
| `jwc remove` | drops the vendor directory and the manifest entry |

The three refusals matter more than the successes, because a stub cannot
model them honestly — all three came back correct, and the client
surfaced each one as a readable message rather than a panic or a 200:

| Attempt | Answer |
|---|---|
| republish an existing version | `409 {"error":"redis@0.2.1 already published"}` |
| publish with no stored token | refused client-side, before any request |
| publish a package owned by another user | `403 {"error":"you do not own this package"}` |

The `/api/v1/crates/…` aliases added in `602a3fb` for older CLI releases
were checked in the same pass, because nothing ships that calls them any
more and an alias that 404s is worse than no alias. Both halves answer
200, and the download returns the same bytes as the `pkg` route under the
same sha256 — the metadata route is aliased too, which matters, since an
old client asks for versions before it asks for bytes.

One defect fell out, and it was in neither of these repos: the `redis`
package still carried a 0.9.x `redis.jwcproj` with a `pkgVersion` field,
so `jwc publish` answered `no jwcproj.json under .` and the package could
not reach a registry at all. Fixed in `just-web-code/redis`.

## Why Google-only auth (v1)

Less code than rolling our own email/password (no flow for password
reset, lockout, email verification, etc.) and gives every contributor
a recoverable account by default. We can add additional providers
later by extending `src/auth.rs` — the JWT shape stays the same so the
client doesn't need to change.

## Out of scope for v1

- Web UI (CLI-only registry; package discovery via `GET /api/v1/pkg`).
- Yanking / unpublishing across mirrors.
- Mirror federation.
- Quota / billing.
- Owner transfer between users (delete + re-publish workflow for now).
