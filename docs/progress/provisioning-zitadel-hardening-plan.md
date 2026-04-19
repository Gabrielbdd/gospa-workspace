# Provisioning + ZITADEL Hardening Plan

## Objective

Define and execute a coherent end-to-end architecture for:

- local developer onboarding (`clone` -> `mise run infra:start` -> `mise run dev`)
- operator onboarding of a fresh Gospa deployment (`/install`)
- post-install steady-state auth/login/runtime behavior
- published/deployed Gospa operation (container image, Kubernetes, restart, rerun, rotation)

The goal is not to patch isolated bugs. The goal is to replace the current
drift between docs, starter contract, product runtime, and ZITADEL/OIDC flow
with one explicit, testable, repeatable contract.

This is workspace-level because the fix crosses both submodules:

- `repos/gofra` owns starter/runtime/tooling contracts
- `repos/gospa` owns the product onboarding flow and ZITADEL-backed install/login behavior

## Current State

- Date: 2026-04-19
- Status: 12 slices delivered (S1–S10 + gate fail-closed + 4 hotfixes); S15
  is next; S11–S14 + S16–S17 deferred behind a pause for real-usage signal
- Scope: cross-repo hardening plan for provisioning, onboarding, auth, DX, and deploy contract

## Context Consulted

### Workspace

- `AGENTS.md`
- `README.md`
- `docs/migration-plan.md`
- `docs/adr/0001-explicit-zitadel-api-audience.md`

### Gospa

- `repos/gospa/AGENTS.md`
- `repos/gospa/README.md`
- `repos/gospa/mise.toml`
- `repos/gospa/compose.yaml`
- `repos/gospa/.env.example`
- `repos/gospa/scripts/load-env.sh`
- `repos/gospa/scripts/compose.sh`
- `repos/gospa/scripts/wait-for-zitadel.sh`
- `repos/gospa/scripts/copy-provisioner-pat.sh`
- `repos/gospa/infra/zitadel/steps.yaml`
- `repos/gospa/cmd/app/main.go`
- `repos/gospa/internal/authgate/gate.go`
- `repos/gospa/internal/install/handler.go`
- `repos/gospa/internal/install/orchestrator.go`
- `repos/gospa/internal/publicconfig/resolver.go`
- `repos/gospa/internal/zitadel/client.go`
- `repos/gospa/db/migrations/00001_create_workspace.sql`
- `repos/gospa/db/migrations/00002_create_companies.sql`
- `repos/gospa/proto/gospa/config/v1/config.proto`
- `repos/gospa/proto/gospa/install/v1/install.proto`
- `repos/gospa/web/index.html`
- `repos/gospa/web/embed.go`
- `repos/gospa/web/embed_dev.go`
- `repos/gospa/web/package.json`
- `repos/gospa/web/src/main.tsx`
- `repos/gospa/web/src/lib/auth.ts`
- `repos/gospa/web/src/lib/install-client.ts`
- `repos/gospa/web/src/routes/index.tsx`
- `repos/gospa/web/src/routes/install.tsx`
- `repos/gospa/docs/operations.md`
- `repos/gospa/docs/examples/deploy/kubernetes/README.md`
- `repos/gospa/docs/examples/deploy/kubernetes/deployment.yaml`
- `repos/gospa/docs/examples/deploy/compose/README.md`
- `repos/gospa/docs/blueprint/index.md`
- related tests under `cmd/app`, `internal/authgate`, `internal/install`, `internal/publicconfig`, `internal/zitadel`

### Gofra

- `repos/gofra/AGENTS.md`
- `repos/gofra/docs/02-system-architecture.md`
- `repos/gofra/docs/08-auth.md`
- `repos/gofra/docs/14-tooling.md`
- `repos/gofra/docs/15-docker-compose.md`
- `repos/gofra/docs/17-decision-log.md`
- `repos/gofra/docs/19-implementation-gaps.md`
- `repos/gofra/docs/framework/reference/runtime/auth.md`
- `repos/gofra/docs/framework/reference/runtime/zitadel.md`
- `repos/gofra/docs/framework/tutorials/03-understanding-what-was-generated.md`
- `repos/gofra/internal/scaffold/starter/full/README.md`
- `repos/gofra/internal/scaffold/starter/full/mise.toml.tmpl`
- `repos/gofra/internal/scaffold/starter/full/compose.yaml`
- `repos/gofra/internal/scaffold/starter/full/.env.example`
- `repos/gofra/internal/scaffold/starter/full/scripts/load-env.sh`
- `repos/gofra/internal/scaffold/starter/full/web/embed_dev.go.tmpl`
- `repos/gofra/internal/scaffold/generator_test.go`
- `repos/gofra/runtime/auth/verifier.go`
- `repos/gofra/runtime/zitadel/secret/secret.go`

### External References

- ZITADEL OIDC Authorization Code + PKCE:
  <https://zitadel.com/docs/guides/integrate/login/oidc/login-users>
- ZITADEL applications overview:
  <https://zitadel.com/docs/guides/manage/console/applications-overview>
- ZITADEL AddOIDCApp API:
  <https://zitadel.com/docs/reference/api/management/zitadel.management.v1.ManagementService.AddOIDCApp>
- ZITADEL roles/audience guidance:
  <https://zitadel.com/docs/guides/integrate/retrieve-user-roles>
- ZITADEL API access guidance:
  <https://zitadel.com/docs/guides/integrate/zitadel-apis/access-zitadel-apis>
- Vite backend integration:
  <https://vite.dev/guide/backend-integration.html>
- Kubernetes init containers:
  <https://kubernetes.io/docs/concepts/workloads/pods/init-containers/>
- Supabase local lifecycle as a mature DX reference:
  <https://supabase.com/docs/reference/cli/getting-started>
- Grafana provisioning-as-code as a deterministic provisioning reference:
  <https://grafana.com/docs/grafana/latest/administration/provisioning/>
- Auth0 PKCE documentation as a browser-OIDC reference:
  <https://auth0.com/docs/get-started/authentication-and-authorization-flow/authorization-code-flow-with-pkce>
- `react-oidc-context` (selected for browser auth):
  <https://github.com/authts/react-oidc-context>

## What Was Validated

The following was exercised directly during investigation:

- `go test ./...` in `repos/gospa`
- `go test ./...` in `repos/gofra`
- `go build -o /tmp/gospa-app ./cmd/app` in `repos/gospa`
- `npm run build` in `repos/gospa/web`
- `mise run infra` in `repos/gospa`
- `mise run infra` again against persisted state in `repos/gospa`
- `mise run dev` in `repos/gospa`
- live HTTP checks against the running app (`/`, `/_gofra/config.js`, and a protected RPC)
- code-level exploration of auth gate, install orchestrator, persisted schema,
  browser auth slice, starter dev contract, and K8s deploy story (2026-04-19)

## Current Findings

### 1. Auth gate eager activation is bugged after restart

`cmd/app/main.go:159-169` calls `gate.Activate(ctx, ws.ZitadelProjectID.String)`
for established workspaces (`install_state = ready`). But the activation runs
**before** `app.Use(gate.Middleware)` is invoked at `cmd/app/main.go:187`.

Inside `internal/authgate/gate.go:71-73`, `Activate` reads
`g.passthrough.Load()` which is only populated when `Middleware` is mounted.
The load returns `nil`, `Activate` returns `ErrMiddlewareNotMounted`, and the
caller logs the error at `warn` level (`main.go:165`) and continues.

Result: a restarted ready workspace serves private RPCs unauthenticated.
The `OnReady` path at `main.go:125-129` is correct (it runs after the wizard
completes, by which time the middleware is mounted) — only the eager
post-restart path is broken.

**Fix is trivial: move the `:159-169` block to after `app.Use(gate.Middleware)`.**
This is a runtime correctness bug.

### 2. Browser login is incomplete end-to-end (~20% complete)

`web/src/lib/auth.ts` builds an authorize URL with PKCE (state + code verifier
stashed in `sessionStorage`). That is all.

What is missing:

- no `/auth/callback` route registered in `web/src/routes/`
- no token exchange handler
- no token storage / session restore
- no refresh logic (despite `offline_access` being requested)
- no logout flow
- `react-oidc-context` is not in `web/package.json`

The product claims OIDC login works; the implementation only covers the very
first redirect step.

### 3. Public runtime auth config is incomplete

The install flow persists `zitadel_org_id`, `zitadel_project_id`,
`zitadel_spa_app_id`, and `zitadel_spa_client_id` (see
`db/migrations/00001_create_workspace.sql:25-28` and
`internal/install/orchestrator.go:118-125`). `publicconfig/resolver.go` only
mutates `auth.orgId` on top of static config. Issuer URL, management URL, API
audience, and the audience-request scope are all sourced from static config
(`cfg.Auth.*`, `cfg.Zitadel.AdminAPIURL`), not from the workspace row.

### 4. Developer experience contract is incoherent

`mise run dev` starts Go and Vite in parallel
(`repos/gofra/internal/scaffold/starter/full/mise.toml.tmpl:47-62`), but the
Go binary is not built with `-tags dev`. Without that build tag,
`web/embed.go` (production embed) is selected at compile time, so the browser
keeps receiving the embedded `dist` even while Vite is also running on 5173.

The documented "backend + frontend together" loop is not the loop the user is
exercising in the browser.

### 5. Docs and runtime disagree about Kubernetes behavior

`OnReady` at `cmd/app/main.go:125-129` already activates auth in-place via
`gate.Activate(...)`. `docs/operations.md` (Scenario K1) describes this
correctly. But `docs/examples/deploy/kubernetes/README.md:51-63` still
prescribes a manual `kubectl rollout restart deployment/gospa` after install.

The runtime is correct; the deploy README lies.

### 6. Provisioning and steady-state runtime are mixed

Bootstrap PAT (`scripts/copy-provisioner-pat.sh`,
`scripts/wait-for-zitadel.sh`) is loaded once at startup
(`cmd/app/main.go:69-74`) and reused for both:

- install-time provisioning (`Orchestrator.Run` → `Zitadel.SetUpOrg`, etc.)
- steady-state runtime (`companies/handler.go:60` calls `Zitadel.AddOrganization`)

There is no separation between bootstrap credential and runtime credential.
PAT rotation requires a process restart.

### 7. K8s example contradicts already-implemented in-place activation

(Subset of Finding 5 but called out separately because the fix is doc-only.)
`docs/examples/deploy/kubernetes/README.md:51-63` instructs operators to
`kubectl rollout restart` after install. The runtime does not require this.
Removing that instruction is a single-PR doc fix.

### 8. `/install` is publicly invokable without any token

`proto/gospa/install/v1/install.proto:6-8` declares the install routes as
"intentionally public". `internal/install/handler.go` enforces no token, no
HMAC, no shared secret. The only protection is the operator's discipline in
keeping ingress private until install completes.

This is the most serious security debt today and is not cosmetic.

### 9. chi `app.Use` is lazy — gate cannot rely on mount order

`gate := authgate.New(...)` followed by `app.Use(gate.Middleware)` does
**not** invoke the middleware function. chi compiles the chain only when the
first request arrives (or when the router is asked to serve). The original
`Gate` design stored `passthrough` from inside the middleware function, so
`Activate()` called at startup raced ahead of any mount and silently failed
with `ErrMiddlewareNotMounted`.

Discovered while validating the eager activation path on Pod restart.
Resolved by refactoring `authgate.Middleware` to lazy-cache the
authenticated chain on demand: `Activate` only stores the verifier, the
middleware compiles its chain on the next request that flows through. Order
between `app.Use` and `Activate` is now irrelevant.

### 10. ZITADEL OIDC apps default to **opaque** access tokens

`AddOIDCApp` without `accessTokenType` makes ZITADEL issue
`OIDC_TOKEN_TYPE_BEARER` (opaque) tokens. The `runtime/auth` verifier is
JWT-only — it tries to decode the bearer as a JWT and returns
`401 invalid token` on every authenticated RPC even when the user just
logged in. Manifests as: Companies probe panel reports
"Authenticated RPC failed: invalid token" right after a successful login;
the bearer arrives, the gate just can't decode it.

Resolved by setting `accessTokenType: "OIDC_TOKEN_TYPE_JWT"` on the
`AddOIDCApp` call.

### 11. `SetUpOrg` without `password` triggers ZITADEL email init flow

If `SetUpOrg`'s human payload omits `password`, ZITADEL marks the user as
needing an init code and sends it by email. On any deploy without
configured SMTP — including the local docker-compose setup — that email
never arrives and the operator is locked out at the consent screen with
"Verify your e-mail with the code below and set your password."

Resolved by collecting the admin password in the wizard and threading it
through to `SetUpOrg`.

### 12. Pre-install gate passthrough exposed private RPCs at the handler

The original `Gate.Middleware` inactive branch was full passthrough — any
private Connect RPC reached its handler pre-install and was barred only by
per-handler `requireReady` guards. Defense in depth with the first layer
missing: a private RPC added without that guard would leak.

Resolved by making the gate's inactive branch fail-closed for private
Connect procedures (anything not in `PublicProcedures`). `requireReady`
stays as second-layer defense.

## Architectural Principles

The following principles are locked for this work unless evidence emerges that
they are materially wrong:

1. The product must have one explicit contract for each lifecycle:
   local dev, first install, post-install runtime, and deploy/publish.
2. Auth/runtime contracts must be explicit persisted data, not inference from
   "close enough" ZITADEL identifiers.
3. Local developer UX must be deterministic. Silent fallbacks are not
   acceptable.
4. Documentation must describe the real contract, not a nearby aspiration.
5. The Gofra starter must not teach a broken contract that Gospa then works
   around locally.
6. Provisioning must be exerciseable in tests, not only explained in prose.
7. Security-sensitive bootstrap paths need product-level design, not only
   operational mitigation.
8. ZITADEL audience semantics are fixed by
   [ADR 0001](../adr/0001-explicit-zitadel-api-audience.md).

## Target Architecture

### A. Lifecycle Model

There are four distinct lifecycle states and they must remain distinct in code
and docs:

1. Local infrastructure lifecycle
2. Application runtime lifecycle
3. Workspace install/onboarding lifecycle
4. Published deployment lifecycle

They currently leak into each other. The target architecture makes each of them
explicit.

### B. Explicit Auth Contract

The product must persist and consume an explicit auth/runtime contract. The
following matrix is the v1 target — the rationale per field is recorded in
ADR 0001 (`docs/adr/0001-explicit-zitadel-api-audience.md`).

| Field | Persisted today | v1 Target | Rationale |
|-------|-----------------|-----------|-----------|
| `zitadel_org_id` | ✓ | ✓ persisted | Provisioning identifier; required by management calls |
| `zitadel_project_id` | ✓ | ✓ persisted | Provisioning identifier; basis for audience scope |
| `zitadel_spa_app_id` | ✓ | ✓ persisted | Provisioning identifier |
| `zitadel_spa_client_id` | ✓ | ✓ persisted | Browser PKCE client identifier |
| `zitadel_issuer_url` | ✗ (static) | ✓ persisted | OIDC discovery + browser authorize endpoint. Source: explicit auth config, fallback to `cfg.Zitadel.AdminAPIURL` |
| `zitadel_management_url` | ✗ (static, equals issuer) | ✓ persisted | Admin API endpoint; explicitly separate from issuer because real deploys put the public issuer and the cluster-internal admin endpoint on different hosts |
| `zitadel_api_audience` | ✗ (static, equals project) | ✓ persisted | Backend JWT validation contract; per ADR 0001 |
| `zitadel_api_audience_scope` | ✗ (derived) | **server helper, not persisted in v1** | OIDC scope the browser requests to obtain the audience-bearing token. v1 derives this in a server-side helper from `project_id` (`urn:zitadel:iam:org:project:id:{project_id}:aud`). Promote to a persisted field in a later round if a real configurability need emerges. Browser **does not** receive this as a separate field — scopes already carry the value when correct |

Notes:

- `zitadel_project_id` is a provisioning/resource identifier.
- `zitadel_api_audience` is a backend auth contract.
- `zitadel_api_audience_scope` is the browser-side OIDC request contract for
  obtaining that audience.
- They may be equal initially, but they are not the same concept.
- `zitadel_issuer_url` and `zitadel_management_url` may be equal initially,
  but they are explicitly separate concepts (decision recorded in
  "Decisions Taken" #2).

The runtime then follows one simple rule set:

- browser login uses `zitadel_issuer_url + zitadel_spa_client_id`
- JWT verifier uses `zitadel_issuer_url + zitadel_api_audience`
- ZITADEL admin/management calls use `zitadel_management_url + bootstrap/runtime credential`

No runtime component should infer these values from loosely related fields.

### C. Browser Auth Model

The browser contract follows the Gofra architecture that is already documented:

- Authorization Code + PKCE
- `react-oidc-context` (selected library; see "Decisions Taken" #9)
- `sessionStorage`
- `offline_access`
- one explicit callback route
- one explicit logout path
- role scope and API audience requested intentionally
- post-install config reload only as a consistency mechanism, not as the core
  auth mechanism

### D. Local Dev Contract

The local developer contract should behave like a mature product CLI story.
Note: this is a **breaking change** to the Gofra starter — `infra` is renamed
to `infra:start` with no alias (see "Decisions Taken" #8).

- `infra:start`
- `infra:status`
- `infra:stop`
- `infra:restart`
- `infra:reset`
- `infra:logs`
- `doctor`
- `dev`

Required properties:

- rerunnable without ambiguity
- restart distinct from reset
- stable ports
- stable browser entrypoint
- same-origin dev through Go when that is the documented contract
- Vite dev port must be strict when the app depends on it
- `dev` task must compile the Go binary with `-tags dev` so the documented
  Vite-driven loop is the loop the user actually exercises in the browser

### E. Published/Deployed Contract

Production/published Gospa must separate:

- build artifact creation
- database migration execution
- bootstrap/install enablement
- steady-state runtime serving

The main application container should not be the place where unrelated concerns
quietly happen because it was "convenient".

### F. Codebase Ownership Boundary

#### Gofra owns

- starter DX contract
- starter dev/prod frontend integration contract
- generic runtime auth contract and docs
- starter smoke tests that guard those contracts

#### Gospa owns

- workspace install state machine
- `/install` wizard
- persisted product auth contract
- ZITADEL-backed install/login semantics
- product-specific docs and operator runbooks

## Comparison to Mature Practices

### Vite

Vite's official backend integration makes dev/prod explicit. In dev, the
backend either proxies Vite correctly or injects Vite dev scripts. In prod, the
backend serves built assets. It does not encourage "start Vite anyway and maybe
the browser still sees dist".

### ZITADEL/Auth0

A browser PKCE flow is a full contract:

- authorize URL
- callback
- token exchange
- storage
- refresh
- logout
- audience/scopes

Stopping at "we can redirect to /authorize" is not a finished login system.

### Supabase

Supabase exposes a clear local lifecycle: start, stop, status, preserve data by
default, wipe only when explicitly requested. Gospa should adopt that level of
explicitness for local infra management.

### Kubernetes

Migrations and setup preconditions belong in init containers or Jobs when they
are startup preconditions. They should not rely on side effects that are hard to
reason about after restarts or rollouts.

### Grafana

Provisioning-as-code is versioned, repeatable, and reviewable. Gospa's install
and bootstrap story should aim for the same predictability even when some parts
remain operator-driven in the MVP.

## Execution Plan

The plan is intentionally sliced into small, independent deliveries (typically
one PR / one agent / one session each). Each slice carries its own validation —
there is no separate "test floor" phase at the end. Cross-impact (gospa,
gofra, none) is declared per slice.

### Status (2026-04-19)

The full install → login → authenticated RPC → PAT rotation cycle is closed
and manually validated end-to-end. Every slice from S1 through S10 plus the
gate fail-closed slice and four unplanned hotfixes have landed.

| Slice | Status | Commit | Notes |
|-------|--------|--------|-------|
| S1  — auth gate ordering + restart regression                          | done    | gospa `fb94f19` | + fail-loud on Activate error |
| S2  — K8s docs drop rollout-restart                                    | done    | gospa `40f7f6b` | doc-only |
| S2.5 — publicconfig injects persisted `spa_client_id`                   | done    | gospa `89e73d5` | unblocks login `Errors.App.NotFound` |
| S3  — install token bootstrap secret + wizard field                     | done    | gospa `f881f23` | env literal / file / auto-gen fallback |
| S3.5 — goose via mise `[tools] ubi:pressly/goose`                       | done    | gofra `0006c6a` + gospa `8fec18b` | superseded a `go run` first attempt |
| S4  — align code/docs to ADR 0001                                       | subsumed by ADR 0001 | n/a | no new ADR; v1 scope refinement folded into 0001 |
| S5  — persist 3-field auth contract + read-repair                       | done    | gospa `38893b3` | `internal/zitadelcontract` helper |
| S6  — publicconfig exposes persisted `issuer`                           | done    | gospa `da16106` | narrow browser surface |
| S7  — authgate consumes persisted contract; eager fail-closed           | done    | gospa `738b142` | gate constructor drops fixed issuer |
| S8  — `react-oidc-context` + `/auth/callback`                           | done    | gospa `e486172` | sessionStorage-backed; logout flow |
| S9  — bearer threading + audience anchored on `api_audience` + `offline_access` | done | gospa `a04cbed` | Companies probe panel exercises end-to-end |
| Gate fail-closed pre-install                                            | done    | gospa `7ad072c` | private Connect procedures 401 from gate, not handler |
| S10 — PAT hot-reload via `internal/patwatch`                            | done    | gospa `454679a` | last-known-good; K8s symlink-swap aware |

Hotfixes that landed during execution (not in the original numbered plan):

| Slice | Status | Commit | Notes |
|-------|--------|--------|-------|
| Install admin password threaded into `SetUpOrg` | done | gospa `a098bd3` | unblocks login on no-SMTP setups |
| OIDC SPA app forced to JWT access tokens         | done | gospa `4233517` | ZITADEL default is opaque; verifier is JWT-only |
| chi-lazy mount fix for authgate                  | done | gospa `98aa638` | gate stores verifier, middleware caches chain on demand |
| Wizard password confirm + ZITADEL `login_hint`   | done | gospa `962c22e` | localStorage; convenience |

Pending slices:

| Slice | Repo | Scope | Size |
|-------|------|-------|------|
| **S15** | gospa | Crash-recovery: best-effort cleanup of ZITADEL org created mid-install when a later step fails. Compensates only resources from the current execution; logs cleanup failures into `install_error`. Not a saga, not a sweep of historical orphans | M |
| **S11** | gofra | Rename `infra` → `infra:start` (breaking, no alias); add `infra:status` and `infra:restart`; bump starter major version | S+ |
| **S12** | gofra | Make `dev` task compile with `-tags dev` (or introduce explicit `dev:proxy` task); enforce strict Vite port | S |
| **S13** | gofra | Add `doctor` task | S |
| **S14** | gospa | Adopt the updated starter (S11–S13) — migrate `mise.toml`, `scripts/load-env.sh`, `.env.example` | S |
| **S16** | gospa | Image build contract + migration execution contract (init container or Job) | M |
| **S17** | gospa | Live K8s manifests + smoke test in CI; first-deploy + post-install steady-state validation | M |

### Slice ordering rationale (post-S10)

- **S15 next.** It is the last gap in confiability of the install path — without it, a mid-install failure leaks ZITADEL orgs forever.
- **Pause for real usage after S15.** S11–S17 are DX/operational hardening; their priorities are easier to set after the system has been used with real workload for a few days.
- **S11 → S14** is one logical block (starter rename + Gospa adoption). Schedule together when DX friction starts hurting.
- **S16 → S17** is the deploy-story block. Schedule when CI/operational pain shows up (e.g., publishing an image without a smoke that catches obvious regressions).

## Agent-Oriented Execution Rules

This plan is designed for incremental AI-agent work. Each agent task should:

1. Pick exactly one slice.
2. Restate the local objective in repo-local terms.
3. Declare cross-impact explicitly:
   - impact on `repos/gofra`
   - impact on `repos/gospa`
   - or "none", with reason
4. Update this progress file after each meaningful change if the work spans
   sessions.
5. Keep docs in the same change whenever the public or operational contract
   changes.
6. Prefer contract-first fixes over tactical local workarounds.
7. Land the slice's validation (test, manual check, doc) in the same PR — no
   "tests later" deferral.

## Risks

### Operational risk

Resolved (with the slice that closed each):

- ~~`/install` publicly invokable without any token~~ — **resolved in S3** (`f881f23`).
- ~~post-restart auth correctness not trustworthy~~ — **resolved in S1** + chi-lazy fix (`98aa638`).
- ~~PAT rotation requires process restart~~ — **resolved in S10** (`454679a`).
- ~~private RPCs reachable pre-install via passthrough~~ — **resolved by gate fail-closed slice** (`7ad072c`).

Remaining:

- **Medium**: shared PAT has IAM_OWNER scope in both bootstrap and runtime
  paths. S10 made rotation invisible but did not reduce blast radius. Real
  reduction requires architecture change (e.g. operator pre-allocates a pool
  of orgs Gospa only assigns from), not just a credential split. Out of
  scope for the current workstream.
- **Medium**: mid-install crash before `MarkWorkspaceProvisioning` leaks the
  org created in step 1. Read-repair flips the workspace back to `failed`
  but the orphan stays in ZITADEL. **S15 closes the in-flow case** (failure
  inside `Run`); kernel-kill style crashes still need manual cleanup.
- **Medium today**: local dev teaches false assumptions about what the browser
  is actually running (will be resolved by S12).

### Delivery risk

- **Medium**: the problem crosses two repos and one workspace-level contract.
  Mostly behind us — only S11–S14 still cross repos.
- **Medium**: S11 is a breaking change to the Gofra starter. Consumers
  (Gospa included via S14) must migrate explicitly. Communicate via release
  notes and starter major bump.

### Scope risk

- **Medium**: it is tempting to jump directly to Restate, multi-replica
  install, or a BFF/session model. Those are separate architecture expansions
  and are not included here.

## Decisions Taken

The following are considered decided for this workstream:

1. Auth contract values must be explicit persisted data.
2. `issuer_url` and `management_url` are different concerns and are persisted
   as separate fields, even when initially equal. (Confirmed 2026-04-19 after
   exploration debate; recorded here so future agents do not collapse them.)
3. `api_audience` must be treated as explicit contract, even if initially equal
   to `project_id`.
4. ~~`api_audience_scope` must be treated as explicit contract, even if initially
   derived from `project_id`.~~ **Revised (S5):** kept as a server-side helper
   in `internal/zitadelcontract.AudienceScope`, not persisted in v1. Promote
   to a column when a real configurability need emerges.
5. Local dev needs explicit lifecycle commands and deterministic port behavior.
6. The starter contract in Gofra must be fixed, not bypassed permanently in
   Gospa.
7. Browser auth must align with the Gofra OIDC architecture rather than invent
   a second parallel auth model in Gospa.
8. `infra` is renamed to `infra:start` in the Gofra starter as a **breaking
   change with no alias**. Consumers migrate at a starter major version bump.
   (Confirmed 2026-04-19.)
9. Browser auth uses `react-oidc-context` rather than a hand-rolled PKCE
   implementation. (Confirmed 2026-04-19.)
10. Install token (S3) ships before the persisted auth contract (S4–S9)
    because `/install` security is the largest current debt and is independent
    of the contract refactor. (Confirmed 2026-04-19.)
11. **S10:** PAT hot-reload via `internal/patwatch` over splitting bootstrap
    vs runtime credentials. Splitting only buys real security once the two
    paths can run with different scopes; today both need IAM_OWNER for
    `AddOrganization`, so a split would be operational complexity for no
    security gain. Revisit when an alternative org-creation flow lets the
    runtime PAT be lower-privilege.
12. **S3.5:** the goose CLI is installed by `mise install` via
    `"ubi:pressly/goose" = "3.27.0"` in the starter `[tools]`, not by
    wrapping every task in `go run github.com/pressly/goose/v3/cmd/goose`.
    The first attempt used `go run`; user pushed back ("por que não instala
    com o mise?") and the corrected approach landed in the next commit.
13. **S8 hotfix:** `login_hint` for the ZITADEL login screen is sourced from
    `localStorage` (written on wizard submit + on every successful login),
    not from a persisted column. Browser-local convenience; no PII leaves
    the device.
14. **JWT hotfix (`4233517`):** OIDC SPA app must be created with
    `accessTokenType: OIDC_TOKEN_TYPE_JWT`. ZITADEL's default for new apps
    is `OIDC_TOKEN_TYPE_BEARER` (opaque), which the JWT-only `runtime/auth`
    verifier cannot decode.
15. **Install password hotfix (`a098bd3`):** the wizard collects the admin's
    initial password and threads it through to `SetUpOrg`. Without it,
    ZITADEL emails an init code that never arrives on a no-SMTP deploy.
16. **Gate fail-closed (`7ad072c`):** the inactive-gate branch blocks
    private Connect procedures with a 401 + Connect-shape JSON body before
    the handler runs. `requireReady` in `companies/handler.go` stays as
    second-layer defense.

## Open Questions

These are the remaining meaningful architectural questions:

1. ~~Install token shape: HMAC server-side, or operator-supplied shared secret?~~
   **Closed:** operator-supplied with auto-gen fallback for try-it-out (S3).
2. ~~PAT separation strategy: bootstrap PAT vs runtime PAT split, or hot-reload?~~
   **Closed:** hot-reload (S10). Split deferred until scopes can differ
   meaningfully (Decision 11).
3. ~~Crash recovery: orphan ZITADEL org cleanup automatic or doc-only?~~
   **Closed:** best-effort opportunistic cleanup for resources created in
   the current execution; cleanup failures recorded in `install_error`. The
   kernel-kill / mid-flow crash case stays operator-cleanup territory.
   Implementation is **S15** (next slice).
4. Should install durability eventually move to Restate? Out of scope for this
   workstream but worth tracking. Re-evaluate after a few weeks of real
   usage post-S15.

## Pending Validations

Each slice carries its own validations (see Execution Plan). High-level
end-to-end validations:

- ~~Full browser validation of install → login → callback → authenticated RPC~~
  **done after S9 + JWT hotfix.** Companies probe panel returns
  "Authenticated RPC OK".
- ~~Restart validation after browser auth is complete~~ **done after S9 +
  chi-lazy fix.** Restart of a `ready` workspace activates the gate via
  persisted contract; `curl` without bearer returns 401.
- Published/deployed validation against the final Kubernetes contract
  (pending S17).
- ~~Implemented ZITADEL token shape confirmed against ADR 0001~~ **done after
  JWT hotfix.** Access tokens are now JWT (verified via `jwt.io` decode of
  `auth.user.access_token`).
- **New:** S15 e2e validation — induce mid-install ZITADEL failure (e.g.
  `docker stop zitadel` after the wizard click) and confirm the orchestrator
  rolled back the created org.

## Next Steps

Recommended immediate execution order:

1. **S15** — orphan ZITADEL org cleanup, best-effort opportunistic (next).
2. **Pause for real usage** — run Gospa with real workload for a few days
   before opening more frontend / DX / deploy frontiers. Real usage usually
   re-prioritises better than the plan does.
3. **S11 → S14** — gofra starter rename + Gospa adoption. Schedule when DX
   friction starts hurting.
4. **S16 → S17** — image build + K8s smoke in CI. Schedule when
   publish/deploy pain shows up.

## Assessment

Current self-assessment for this plan, post-S10:

- Understanding of the problem: 96%
- Context completeness: 94%
- Research quality: 92%
- User experience understanding: 90%
- Proposed solution quality: 92%

What pushed the numbers up since the prior assessment (2026-04-19):

- The full install → login → RPC → rotation cycle is now closed, validated
  end-to-end manually, and persisted in commits with regression tests.
- 12 of the originally-tracked open issues / risks have moved from "open"
  to "resolved with hash"; only 4 remain (PAT scope reduction, mid-install
  kernel-kill crash, S12 dev DX, S17 deploy CI).
- 4 hotfixes that were not in the original plan turned into Decisions
  11–16, so future agents inherit the resolved version of those tradeoffs.

What still keeps the score under 95%:

- S15 design depends on ZITADEL `RemoveOrg` cascade behavior on
  projects/apps; if cascade is partial, the cleanup needs to chain
  `RemoveProject` + `RemoveApp` before `RemoveOrg`. Verified in docs but
  not exercised live yet.
- S11 (starter rename) is breaking; the migration story for existing
  consumers needs more than a release note in practice.
- The "use it for a few days" pause is still pending — until the system has
  real workload, some of the operational decisions are theoretical.
