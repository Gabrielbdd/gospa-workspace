## Context

Gospa today has:

- A successful `/install` flow that mints a ZITADEL org, project, SPA OIDC app,
  and an admin user with `ORG_OWNER` role on the org. Persisted on
  `workspace`.
- A `companies` table with a "one company → one ZITADEL org, eagerly created"
  rule. The handler currently treats every authenticated request as equal.
- An `authgate` middleware that validates JWT signature/audience/issuer and
  blocks pre-install requests, but does not look at any role information.
  There is no `internal/authz` package.
- A frontend that has only `/`, `/install`, `/auth-callback`. No app shell,
  no settings, no team management. (The design system tokens at
  `web/src/styles/` shipped on 2026-04-19 ahead of this change; see D8.)
- A `slug` field on `companies` and on `workspace` that is collected at
  install / company creation, validated as required and unique, and never
  consulted by any code path.

The MSP exists in ZITADEL (org + admin user) but has **no representation in
Gospa's database** — there is no row that says "this is who we are". The
`workspace` singleton holds operator-supplied descriptive fields (name, slug,
timezone, currency) but nothing that ties it to the admin who installed the
deploy.

Two product features want this foundation simultaneously: team management
(invite a technician, demote an admin, suspend an account) and the upcoming
contacts surface (people at client companies). They share the same primitive
— a person record — and they share the same authorization story (admins do
admin things; technicians do everything else).

External constraint: ZITADEL is the source of truth for human **identity**
(email, password, MFA, passkeys, federated IdPs in the future). Gospa must
not become a parallel identity store. But operations like "list the team",
"who can be assigned to this ticket", and "@-mention this person" all
demand a fast local query that survives ZITADEL latency. The existing
`companies` row pattern (ZITADEL identifier + product-side mirror) is the
precedent and is kept.

A previous review iteration proposed reverting to a separate
`team_members` + `contacts` model (the legacy PSA pattern). That direction
was rejected after explicit deliberation: it forces ticket and assignment
flows to choose between polymorphic FKs or duplicated entities, it splits
"the MSP" between `workspace` and `companies` shapes, and it triples the
work to add client portal login later. One productive idea from that
review survived into this design: the
`active`/`not_signed_in_yet`/`suspended` status enum with auto-activation
on first sign-in.

A second deliberation challenged whether **roles belong in ZITADEL at all**.
The original direction had ZITADEL own project roles, ship them in the JWT
via `urn:zitadel:iam:org:project:roles`, and have Gospa parse the claim. The
challenge was: "authz is entirely on our side; why does ZITADEL need to know
about roles?" The answer was that it doesn't — ZITADEL handles authentication
(login, password, MFA, future federation) and Gospa owns authorization end-to-end
via the local `workspace_grants` table. This eliminates an entire class of
mirror-sync bugs, removes the role-staleness window that JWT-embedded roles
would impose, and reduces ZITADEL coupling to "identity, full stop". The
trade-off is one additional DB join per authenticated request — already in
scope of the contact lookup the middleware does anyway. See D3.

A third deliberation, after Slice 0 was implemented, removed read-repair
entirely. The earlier design called for a `teamrepair` startup pass that
backfilled the MSP company + admin contact + admin grant on workspaces
installed before this change, mirroring an existing
`install.RepairAuthContract` pattern. Both were over-engineering for pre-v1:
there are no production users to protect from schema drift, and dev workflow
is "destroy and rebuild constantly". The pattern is now banned by an
explicit `repos/gospa/AGENTS.md` rule. See D6.

## Goals / Non-Goals

**Goals:**

- Make the MSP a first-class entity in Gospa's data model, not a synthetic
  derived from "the user who happens to be logged in".
- Establish one directory primitive (`contacts`) that is shared by team
  members and client-side people, so that "technician opens a ticket as
  requester" stays a single FK and never becomes a polymorphic mess.
- Introduce role-based authorization (`admin`, `technician`) backed by the
  local `workspace_grants` table, replacing the current "any authenticated
  request passes" default.
- Ship a Linear-style Team page (keyboard-first, slideover invite, inline
  detail) that is the first piece of real product UI.
- Add the minimum address fields companies need to be useful for contacts and
  ticketing.
- Remove `slug` cleanly in two waves (UI/API first, schema second) so
  rollback is always possible.
- Lay groundwork (`commandRegistry`, `identity_source` columns) for Cmd+K
  and Microsoft 365 / SCIM integration without building either now.

**Non-Goals:**

- No `sites` table.
- No client portal login.
- No fine-grained ABAC.
- **No ZITADEL project roles, no ZITADEL user grants, no roles claim in the
  JWT, no token introspection.** Authz lives in Gospa's database. ZITADEL
  is identity-only. See D3.
- No Cmd+K palette UI (only the `commandRegistry` foundation).
- No bulk operations on the team page.
- No audit log of role changes.
- No email-based invite flow.
- No SCIM, no Entra ID federation code.
- No new framework primitives in `repos/gofra`. Default expectation: zero
  gofra impact.
- **No read-repair, no backfill, no migration data shims, no mid-install
  recovery code.** Pre-v1 every schema or state change assumes a fresh
  install (`mise run infra:reset` + reinstall). See D6.

## Decisions

### D1. The MSP is a `companies` row with `is_workspace_owner = TRUE`

The MSP company shares the `companies` table with client companies. A
boolean column distinguishes it. The row is materialized during `/install`
in the same transaction that flips `install_state` to `ready`. The
`zitadel_org_id` of this row equals `workspace.zitadel_org_id` — no new
ZITADEL org is created (the workspace org from `/install` is reused).

`ListCompanies` filters `is_workspace_owner = FALSE`. The MSP company is
edited via `Settings > Workspace`, which calls a separate RPC
(`UpdateWorkspaceCompany`), not the generic `UpdateCompany`.

A partial unique index `companies_one_workspace_owner ON companies
(is_workspace_owner) WHERE is_workspace_owner = TRUE` enforces "exactly
one MSP" at the database layer.

**Alternatives considered:**

- *Keep MSP info on `workspace` singleton, never in `companies`*: simpler
  schema today (no flag column). Rejected: forces a polymorphic "ticket
  requester is a workspace user OR a contact" model on every downstream
  feature. Mirrors the legacy PSA design that motivated the unified pivot.
- *Separate `workspace_company` table, structurally similar to `companies`*:
  redundancy without insulation. Rejected.

### D2. `contacts` is the directory primitive, `workspace_grants` is the authz primitive

Two tables, orthogonal concepts:

```text
contacts (id, company_id, full_name, email, phone, mobile, job_title, notes,
          zitadel_user_id?, identity_source, external_id,
          archived_at)

workspace_grants (contact_id PK + FK,
                  role ENUM('admin','technician'),
                  status ENUM('active','not_signed_in_yet','suspended'),
                  last_seen_at TIMESTAMPTZ NULL,
                  granted_at, granted_by_contact_id?)
```

A team member is a `contacts` row at the workspace company **plus** a
`workspace_grants` row. A client-side contact is a `contacts` row at a
client company with no grant. A future client-portal user is a contact
with `zitadel_user_id` set but no `workspace_grants` row (they would have
a row in a future `company_grants` table that does not exist yet).

The lifecycle status (`active`/`not_signed_in_yet`/`suspended`) lives on
`workspace_grants` rather than on `contacts` because it describes "this
person's ability to operate the workspace", not "this person exists in
the directory".

`last_seen_at` lives on `workspace_grants` for the same reason. Updated
by the authz middleware (D3, throttled to ~60s per grant) so it never
adds write amplification to the request hot path.

App-level invariant (handler-enforced, not DB-enforced):
`workspace_grants.contact_id` must reference a contact whose
`company_id = <MSP company id>` and `zitadel_user_id IS NOT NULL`.

**Alternatives considered:**

- *`role` column on `contacts`*: simpler schema, fewer joins. Rejected: ties
  identity-shape to authorization-shape.
- *Separate `team_members` table parallel to `contacts`*: the legacy PSA
  pattern. Rejected for the polymorphism reason in D1.

### D3. Authz lives in Gospa's database, not in JWT claims

ZITADEL handles authentication only. Gospa owns authorization end-to-end:

- `workspace_grants.role` is the **single source of truth**.
- The middleware in `internal/authz/middleware.go` runs **after** Gofra's
  existing `runtime/auth` JWT verifier. It receives `runtime/auth.User`
  from context (which Gofra populated from the validated `sub` claim) and
  performs **one** SQL query:

  ```sql
  SELECT contacts.id        AS contact_id,
         workspace_grants.role,
         workspace_grants.status,
         workspace_grants.last_seen_at
  FROM contacts
  LEFT JOIN workspace_grants ON workspace_grants.contact_id = contacts.id
  WHERE contacts.zitadel_user_id = $1
  LIMIT 1
  ```

  Then:
  1. No row → 401 with code `not_provisioned`.
  2. Row exists but `workspace_grants` is NULL → 403 with code `no_grant`.
  3. `status = 'suspended'` → 401 with code `account_suspended`.
  4. `status = 'not_signed_in_yet'` → flip to `'active'` (idempotent
     UPDATE), attach context, proceed (decision F).
  5. Otherwise → attach `(contact_id, role)` to context; throttled-update
     `last_seen_at`; apply the policy map by role.

The policy map lives in `internal/authz/policy.go` as an explicit
`map[string]Level` (`LevelAuthenticated` or `LevelAdminOnly`). RPCs not
in the map are rejected by default.

**Alternatives considered:**

- *Roles in JWT via ZITADEL project roles*: rejected. JWT-TTL-bounded
  staleness window on every role change; requires roles scope on the
  publicconfig mutator; requires synchronous mirror writes to ZITADEL
  on every grant change.
- *ZITADEL token introspection per request*: ~50ms per RPC. Rejected.

### D4. Invite flow: server-generated password, shown once, password-change required

The team page invite handler:

1. Validates input (email, name, role, last-admin invariant).
2. Generates a 24-character cryptographically random password.
3. Calls `ZITADEL.AddHumanUser` with `passwordChangeRequired = true`,
   `isEmailVerified = true`. Captures `zitadel_user_id`.
4. In a single DB transaction, inserts the contact and the grant
   (`role = chosen`, `status = 'not_signed_in_yet'`).
5. Returns the password to the admin **once** in the RPC response. The UI
   shows it in a one-time modal with copy-to-clipboard.
6. On any failure after step 3, runs `ZITADEL.RemoveUser` in a `defer`
   block — same pattern as S15's `RemoveOrg` cleanup.

Note: there is **no** `ZITADEL.AddUserGrant` call. The grant is local
(decision D3).

**Alternatives considered:**

- *Real email invite from day 1*: requires SMTP. Rejected for MVP parity
  with the install flow.
- *Magic link with no password*: depends on email delivery. Rejected.

### D5. `slug` removal in two waves

**Wave 1** (Slice 2): remove `slug` from the proto, the handler, the sqlc
queries, the install wizard form, and the SPA. The DB column stays; inserts
use the column's default empty string. **Before** stopping the writes, a
separate prior step in Slice 0 widens the partial unique index to also
exclude empty strings (`WHERE archived_at IS NULL AND slug != ''`).

**Wave 2** (Slice 7): drop the column and the unique index.

**Conditional escape hatch**: if, before starting Slice 2, anyone discovers
a real architectural use of `slug`, the slice converts to "rename for
clarity" instead of "remove" — same two-wave structure, different end state.

### D6. No read-repair, no backfill — pre-v1 contract

Pre-v1, every schema or state change assumes the operator runs a fresh
install. There is no read-repair, no backfill, no migration data shim, no
mid-install crash recovery. If a workspace exists in a state the current
code can't handle, the recovery path is `mise run infra:reset` +
reinstall.

This is recorded as a hard rule in `repos/gospa/AGENTS.md` so future
agents (and we) don't accidentally re-add the pattern.

**Why:**

- Pre-v1: no production users, no persistent data we can't drop.
- Dev workflow: `mise run infra:reset` is daily.
- Backfill code carries permanent maintenance cost (testing, schema
  evolution, deprecation) for benefit that doesn't exist yet.
- Honest contract: "drop the DB and reinstall" is faster, more visible,
  and easier to reason about than "the app silently self-heals".

**Earlier draft of this design called for `teamrepair`** — a startup pass
that backfilled the MSP company + admin contact + admin grant on
workspaces installed before this change. That mirrored a pre-existing
`install.RepairAuthContract` pattern from the provisioning hardening cycle.

Both were removed in this slice:
- `internal/teamrepair/` (entire package, never committed) — gone
- `internal/install/repair.go` (existed) — gone
- `internal/install/repair_test.go` — gone
- `internal/zitadelcontract.DeriveRepair` — gone
- `RepairWorkspaceAuthContract` query in `db/queries/workspace.sql` — gone
- `workspace.zitadel_admin_user_id` column (added solely to support repair) — gone
- ZITADEL client methods `GetUserByID` and `ListOrgOwners` (added solely
  for repair recovery) — gone
- The mid-install crash-recovery block in `cmd/app/main.go` (flipped a
  `provisioning` workspace to `failed` after process restart) — gone
- All wiring from `cmd/app/main.go`

**When does this rule relax?** Only when v1 ships and there is a
production deploy with persistent data we cannot drop. Then the
discussion reopens — and the bar for re-introducing repair-style
patterns is "operator runs an explicit `gospa migrate` command", not
"app self-heals on every boot".

### D7. UI shell adopted from Claude Design handoff

The Claude Design handoff (2026-04-19) shipped a usable Linear-style shell
(sidebar with section headers, single-tenant workspace identity, top bar
with breadcrumbs + Cmd+K invocation, Cmd+K palette skeleton). We adopt the
shell as part of Slice 4.

Constraint: nav items beyond Settings render disabled with a "Soon" badge
until their respective features land. The Cmd+K palette renders but
registers only the verbs that exist.

### D8. Design system tokens

The design system tokens at `web/src/styles/{index,tokens,typography}.css`
shipped on 2026-04-19 ahead of this change to unblock component port. They
are the canonical source of color, typography, spacing, radii, shadows,
motion. Theme model: dark is default. Tailwind v4 `@theme` block aliases
the raw CSS vars. shadcn legacy aliases preserved so `Button` still works.

**Constraint:** No hex codes outside `tokens.css`. Components consume via
Tailwind utilities or `var(--token)` references.

## Risks / Trade-offs

- **[Suspend doesn't cut the ZITADEL session]** → Suspended members can
  still get fresh tokens from ZITADEL. Our middleware rejects them, but
  if Gospa shares ZITADEL with another future system they could use that
  system. Decision G accepts this for MVP.

- **[Out-of-band ZITADEL user creation]** → Middleware rejects with 401
  when the user authenticates but no contact row exists.

- **[Mid-invite kernel-kill]** → If the process dies between
  `AddHumanUser` succeeding and the DB transaction committing, a ZITADEL
  user exists with no contact row. On next login, the gate rejects.
  Pre-v1 the fix is `mise run infra:reset`; post-v1 will need a proper
  saga (Restate is the planned vehicle).

- **[Mid-install kernel-kill leaves workspace stuck in `provisioning`]**
  → Pre-v1 (per D6) the fix is `mise run infra:reset`. The
  previously-existing self-heal block was removed.

- **[Last-admin protection bypassed by direct DB edit]** → App-level only.

- **[Schema contains MSP-as-company surprise]** → New contributors may
  query `companies` and be surprised to see one extra row that does not
  represent a customer. Mitigation: column name `is_workspace_owner`
  is explicit; query examples always filter by it.

- **[Slug Wave 1 leaves a column with synthetic empty values]** → Slice 0
  widens the unique partial index to also `WHERE slug != ''`.

- **[Authz read on every request is one query per RPC]** → Trivial cost
  with proper indexes. We're adding one JOIN to a query the middleware
  was already going to do.

## Migration Plan

The change ships in seven slices.

1. **Slice 0 — Schema base.** Add columns and tables. Tighten the slug
   partial index. Materialise MSP company + admin contact + admin grant
   in the install transaction. Validation: tests, build, fresh install.
2. **Slice 1 — Authz foundation.** Add `internal/authz/policy.go`,
   `internal/authz/context.go`, `internal/authz/middleware.go`. Decorate
   all currently-private RPCs.
3. **Slice 2 — Slug Wave 1.** Drop `slug` from the proto, sqlc queries,
   handler, install wizard, SPA.
4. **Slice 3 — Team backend.** Add `proto/gospa/team/v1/team.proto`,
   `internal/team/` package, ZITADEL client extensions
   (`AddHumanUser`, `RemoveUser`).
5. **Slice 4 — Team UI.** Port the Claude Design components, add
   `web/src/routes/settings/team*`, `web/src/lib/command-registry.ts`.
6. **Slice 5 — Companies address fields + hide MSP.**
7. **Slice 6 — Contacts.**
8. **Slice 7 — Slug Wave 2.** Final migration drops the column.

**Rollback / recovery (per D6):**

- Pre-v1: `mise run infra:reset` is the canonical reset path. Each
  slice's individual rollback is "deploy the prior commit" (schema is
  additive in Slice 0; later slices only read existing columns or add
  new tables). Slice 7 is the only schema-destructive step; if it goes
  wrong, drop the DB and reinstall.
- Post-v1: this section will need real rollback strategy. Out of scope
  here.

**Push protocol** (per workspace memory): each slice that touches the
gospa submodule pushes to gospa's `main` first, then the workspace
bumps the submodule pointer. Never `--force`. No slice in this change
touches gofra.

## Open Questions

1. **`last_seen_at` update cadence implementation detail.** The middleware
   throttles updates to ~60s per `contact_id`. In-memory map (per-process)
   for MVP. Multi-replica revisits.
2. **Where does archive of the MSP company go?** It cannot be archived.
   Handler-level invariant.
3. **Does technician get to edit the MSP company itself via Settings?**
   Recommendation: no. `Settings > Workspace` is admin-only.
