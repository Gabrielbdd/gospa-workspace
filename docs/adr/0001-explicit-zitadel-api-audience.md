# ADR 0001 — Explicit ZITADEL API Audience Contract

- Status: Accepted
- Date: 2026-04-19
- Scope: Workspace-level, cross-repo (`repos/gofra` and `repos/gospa`)

## Context

The provisioning and auth investigation found that Gospa currently infers too
much of its auth runtime from loosely related ZITADEL identifiers.

In practice, three different concerns are being conflated:

- the provisioned ZITADEL project identifier
- the backend JWT audience contract
- the browser-side scope used to request that audience

This creates avoidable ambiguity in:

- backend token verification
- browser login configuration
- install-time persistence
- restart/recovery behavior
- documentation and operator reasoning

The current implementation persists `zitadel_project_id`, but does not persist a
complete auth contract for the backend and browser. That leaves the runtime to
infer audience-related behavior instead of reading it from explicit persisted
state.

This ambiguity is especially dangerous because:

- `project_id` is a provisioning/resource identifier
- `api_audience` is a security/runtime contract
- the scope used to request that audience is a separate OIDC concern

These values may align initially, but they are not the same concept and should
not be modeled as if they were.

## Decision

The auth contract will model the backend audience explicitly and persist it as
first-class install/runtime state.

The workspace-level target contract is:

- `zitadel_org_id`
- `zitadel_project_id`
- `zitadel_spa_app_id`
- `zitadel_spa_client_id`
- `zitadel_issuer_url`
- `zitadel_management_url`
- `zitadel_api_audience`
- `zitadel_api_audience_scope`

Definitions:

- `zitadel_project_id`: provisioned ZITADEL resource identifier
- `zitadel_api_audience`: value the backend requires in the JWT `aud` claim
- `zitadel_api_audience_scope`: OIDC scope the browser requests in order to
  obtain that audience
- `zitadel_issuer_url`: OIDC discovery/JWKS/browser issuer base
- `zitadel_management_url`: ZITADEL admin/management API base

Initial value policy:

- `zitadel_api_audience` may initially equal `zitadel_project_id`
- `zitadel_api_audience_scope` may initially be derived as
  `urn:zitadel:iam:org:project:id:{project_id}:aud`

Even in that initial shape, both values remain explicitly persisted fields.

## Rationale

This keeps the runtime contract explicit and prevents hidden inference.

Benefits:

- backend verification reads `issuer_url + api_audience` directly
- browser login reads `issuer_url + spa_client_id + api_audience_scope`
- management calls read `management_url`
- install persists the full auth contract needed after restart
- docs can describe one stable contract instead of implementation folklore
- future changes to ZITADEL project layout or audience handling do not require
  re-encoding security behavior as indirect inference

This also preserves a clean semantic separation:

- provisioning identifiers stay provisioning identifiers
- auth requirements stay auth requirements

## Options Considered

### Option A — Treat audience as the project ID forever

Model:

- persist only `zitadel_project_id`
- derive backend audience from it
- derive the audience-request scope from it

Pros:

- fewer fields
- simple initial implementation
- close to ZITADEL's project-centric model

Cons:

- mixes provisioning and auth semantics
- hides security-relevant behavior behind inference
- makes browser/backend/runtime docs harder to reason about
- increases coupling if the product later needs a different audience contract

### Option B — Persist explicit audience contract

Model:

- persist `zitadel_api_audience`
- persist `zitadel_api_audience_scope`
- allow their initial values to equal/derive from `project_id`

Pros:

- clean semantics
- safer runtime contract
- easier migration path
- easier browser/backend separation
- better fit for long-term product evolution

Cons:

- more fields to persist and migrate
- slightly more install/runtime wiring

## Outcome

Option B is accepted.

The system will treat audience as an explicit persisted contract, not as an
implicit property of `zitadel_project_id`.

## Consequences

### For Gospa

- install must persist the full auth contract
- public runtime config must expose the browser-required subset
- backend verification must consume explicit persisted values
- migrations must support existing workspaces
- tests must assert the explicit contract rather than inferred behavior

### For Gofra

- framework docs should distinguish issuer, management URL, audience, and
  browser login contract clearly
- if a generic runtime primitive is extracted later, it should assume an
  explicit auth contract rather than infer from project identifiers

### For Operators and Developers

- the system becomes easier to reason about after install and after restart
- deploy docs can describe a deterministic auth contract
- browser login and backend verification become easier to test end-to-end

## Migration Guidance

Existing workspaces should migrate without requiring reinstall.

Minimum migration policy:

1. Add explicit persisted fields.
2. Backfill `zitadel_api_audience` from the existing `zitadel_project_id`.
3. Backfill `zitadel_api_audience_scope` from the existing project-based ZITADEL
   audience scope convention.
4. Keep runtime behavior compatible during the migration window.
5. Move browser and backend consumers to the explicit fields.

## Validation Required

This ADR is not complete until the implemented system proves:

- the browser requests the expected audience scope
- the ZITADEL-issued access token contains the expected `aud`
- the backend validates against the explicit persisted `api_audience`
- restart/recovery behavior does not depend on runtime re-derivation

## Cross-Impact

- Impact on `repos/gospa`: direct and required
- Impact on `repos/gofra`: documentation now, runtime primitive possibly later

## References

- [Provisioning + ZITADEL Hardening Plan](../progress/provisioning-zitadel-hardening-plan.md)
- <https://zitadel.com/docs/guides/integrate/login/oidc/login-users>
- <https://zitadel.com/docs/guides/integrate/retrieve-user-roles>
- <https://zitadel.com/docs/guides/integrate/zitadel-apis/access-zitadel-apis>

## Addendum — v1 Implementation Refinement (S5, 2026-04-19)

The "Decision" section above lists eight target fields for the
persisted auth contract. During S5 implementation (gospa migration
`00003_add_workspace_auth_contract.sql`, commit `38893b3`), the
scope was trimmed to **seven persisted fields plus one derived
helper** for the v1 round.

Persisted columns added to `workspace`:

- `zitadel_issuer_url`
- `zitadel_management_url`
- `zitadel_api_audience`

(In addition to the four columns that already existed:
`zitadel_org_id`, `zitadel_project_id`, `zitadel_spa_app_id`,
`zitadel_spa_client_id`.)

**`zitadel_api_audience_scope` was deferred from the schema.** It
lives instead as `internal/zitadelcontract.AudienceScope(audience)`,
a deterministic server-side helper that computes
`urn:zitadel:iam:org:project:id:{audience}:aud` at request time and
appends it to the published `PublicAuthConfig.Scopes` so the browser
never has to learn the URN format.

Why deferred:

- The value is mechanically derivable from `zitadel_api_audience`
  via a fixed format ZITADEL controls. A persisted column would
  carry no information the helper does not already compute.
- No real configurability need has appeared. If one emerges (e.g.
  a deploy that wants to request a different audience scope than
  the one ZITADEL would accept by default), promoting the helper to
  a column is straightforward — add the migration, populate from
  the helper at install + read-repair, and switch the publicconfig
  mutator to read the persisted value.
- Anchoring the scope on the **same field the JWT verifier
  validates** (`zitadel_api_audience`, not `zitadel_project_id`)
  was made explicit during S9 (commit `a04cbed`) so that even if
  the two persisted IDs ever diverge, the scope the browser asks
  for and the audience the gate accepts stay aligned.

The original eight-field decision is preserved here as the long-term
target; this addendum records that the v1 implementation chose to
keep `api_audience_scope` as code rather than data, with a path back
if the trade-off changes.
