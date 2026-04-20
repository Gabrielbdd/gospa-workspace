## Why

Gospa today has a workspace install that mints a ZITADEL admin user, but **the
product has no concept of "team" inside the database**: the admin lives only in
ZITADEL, technician role does not exist, and every authenticated request is
treated identically by the backend — the auth gate validates the JWT signature
but no handler reads any role information. Without that foundation:

- An MSP cannot grow past a single admin.
- "Who is using my workspace?" is unanswerable from inside the product.
- Any feature added now (companies, contacts, tickets, billing) will be built
  on top of an authz vacuum and will need to be retrofitted later.

This change closes that gap with a unified directory primitive (`contacts`) and
an orthogonal authorization primitive (`workspace_grants`), then layers the UI
features the operator actually sees: Team management, simplified Companies
without `slug`, and a minimal Contacts surface inside each company.

The unification (the MSP itself is a `companies` row; team members are
`contacts` of that row with grants) is deliberate — it aligns with how Linear,
Slack, and Notion model identity in apps with external IdPs, and it removes the
polymorphic "requester is either a team member or a contact" burden that
legacy PSAs (ConnectWise, Autotask, Atera) carry forever once it lands. The
schema cost in MVP is one extra flag column.

The authorization model deliberately keeps **all role information in Gospa's
database** (`workspace_grants.role`) and uses ZITADEL only for authentication
(login, password, MFA, future passkeys/federation). No project roles, no user
grants, no roles claim in the JWT. This eliminates the role-staleness window
that JWT-embedded roles would impose, removes a class of mirror-sync bugs
between ZITADEL and Postgres, and reduces ZITADEL coupling to "identity, full
stop". The trade-off (one extra DB join per request, fully in scope of an
existing query) is trivial.

**Pre-v1 contract** (per `repos/gospa/AGENTS.md`): every schema change in this
slice assumes the operator runs a fresh install. There is no read-repair, no
backfill, no migration data shim. If a workspace was installed under an earlier
schema, the recovery path is `mise run infra:reset` + reinstall — not
self-healing code.

## What Changes

- **NEW** `contacts` table — one row per human in the system (MSP staff,
  client-side contacts). Identity (`zitadel_user_id`) is nullable. Carries
  `identity_source` and `external_id` columns from day one so future Microsoft
  365 / SCIM sync can populate them without a schema migration.
- **NEW** `workspace_grants` table — one row per team member, with
  `role IN ('admin','technician')`, a `status` enum
  (`active`, `not_signed_in_yet`, `suspended`), and `last_seen_at` for
  observability and "who's been idle" lookups. Orthogonal to contact-ness.
- **MODIFIED** `companies` — adds `is_workspace_owner BOOL` and address
  fields (`address_line1`, `city`, `region`, `postal_code`, `country`,
  `timezone`). The MSP itself becomes a `companies` row created during
  `/install`.
- **MODIFIED** `/install` orchestrator — after `SetUpOrg`, materializes the
  MSP company + admin contact + admin grant in the same transaction that
  flips `install_state` to `ready`. **No read-repair for legacy installs.**
  **No** ZITADEL project roles are created; **no** ZITADEL user grants are
  issued — authz lives in Postgres.
- **NEW** authz middleware in `internal/authz/` — reads
  `runtime/auth.User.ID` (the JWT subject set by Gofra's existing auth
  layer), looks up the matching contact + grant in one query, attaches
  `(contact_id, role)` to context, and enforces a single declarative
  per-RPC policy map. Three terminal cases:
  - No contact for this `zitadel_user_id` → 401 (decision B).
  - Grant `status = 'suspended'` → 401 with code `account_suspended`.
  - Grant `status = 'not_signed_in_yet'` → flip to `'active'` in the same
    transaction, attach context, proceed (decision F).
- **NEW** Team capability — `Settings > Team` page with list + invite
  (server-generated temporary password, displayed once, `passwordChangeRequired`
  set on the ZITADEL user) + role change + suspend / reactivate. Keyboard-first
  (`/`, `j/k`, `enter`, `esc`, `n`, `r`, `s`, `space`). Verbs are registered in
  a `commandRegistry` so the future Cmd+K palette consumes them with zero
  rework.
- **MODIFIED** Companies — UI gains the address fields. The MSP company is
  hidden from `ListCompanies`. Authz: technicians can create/edit; archive is
  admin-only.
- **NEW** Contacts capability — CRUD inside a company. Fields: `full_name`,
  `job_title?`, `email?`, `phone?`, `mobile?`, `notes?`. No login, no sync, no
  commercial fields.
- **REMOVED — staged** `slug`. Confirmed it has no architectural use (no
  routing key, no public lookup). Two-wave removal: backend/UI first, then
  drop the column. **BREAKING** for any external API caller of
  `CompaniesService.CreateCompany` (none known today). **Conditional escape
  hatch**: if, between this change being approved and Wave 1 starting, an
  architectural use of `slug` is discovered (a partner integration, a
  documented routing requirement), the slice converts to "rename for clarity"
  with the same two-wave structure instead of "remove".
- **POSTPONED, NOT DROPPED** Sites. No `sites` table in this change. Companies
  hold their own address. When a multi-site customer or feature requires it,
  introduce `sites` then with a localized migration that extracts the company
  address into a default site.
- **NEW** `web/src/styles/{index,tokens,typography}.css` — the design system
  copied from the Claude Design handoff and structured as Tailwind v4
  `@theme` map + raw token vars + semantic typography. Geist + dark-first +
  Vercel-inspired neutrals. Already shipped (out-of-band on 2026-04-19) so
  the frontend slices can start consuming tokens immediately. Documented in
  `design.md` D8.
- **REMOVED — pre-existing** `install.RepairAuthContract`,
  `zitadelcontract.DeriveRepair`, the workspace `RepairWorkspaceAuthContract`
  query, the mid-install crash recovery block in `cmd/app/main.go`. These
  were over-engineering for pre-v1 (no production users to protect from
  schema drift). Documented in the new `AGENTS.md` rule.

## Capabilities

### New Capabilities

- `authz`: A single per-RPC policy map enforced by middleware that reads the
  caller's role from the local `workspace_grants` table (NOT from the JWT).
  Replaces the current "any authenticated request passes" default.
- `team`: Members of the workspace — list, invite (with temporary password),
  change role, suspend, reactivate. Backed by `contacts` + `workspace_grants`.
- `contacts`: Directory of people associated with companies. CRUD inside the
  context of a company, no login, no sync.

### Modified Capabilities

- `companies`: Adds address fields, adds the `is_workspace_owner` flag, hides
  the workspace company from operator-facing lists, and removes `slug` in two
  waves. Authz becomes role-aware.
- `install`: Adds the materialization step that creates the MSP company + admin
  contact + admin grant in the install transaction. Removes the legacy
  `RepairAuthContract` startup pass.

## Impact

### Code

- `repos/gospa/proto/gospa/companies/v1/companies.proto` — drops `slug`, adds
  address fields on `Company`, adds `UpdateCompany` and
  `UpdateWorkspaceCompany`.
- `repos/gospa/proto/gospa/contacts/v1/contacts.proto` — new.
- `repos/gospa/proto/gospa/team/v1/team.proto` — new.
- `repos/gospa/proto/gospa/install/v1/install.proto` — drops `workspace_slug`.
- `repos/gospa/internal/companies/handler.go` — slug removal, authz, hide
  workspace company, `UpdateCompany`, `UpdateWorkspaceCompany`.
- `repos/gospa/internal/contacts/` — new package.
- `repos/gospa/internal/team/` — new package.
- `repos/gospa/internal/authz/` — new package (policy + middleware + context).
  Lookup-based, NOT JWT-claim-based.
- `repos/gospa/internal/install/orchestrator.go` — materialization step.
- `repos/gospa/internal/install/repair.go` — **DELETED**.
- `repos/gospa/internal/zitadelcontract/contract.go` — `DeriveRepair`
  **DELETED**; `DeriveFresh` is the only remaining derivation.
- `repos/gospa/internal/zitadel/client.go` — adds **only**: `AddHumanUser`
  (with `passwordChangeRequired`, `isEmailVerified`); `RemoveUser`
  (for invite cleanup).
- `repos/gospa/cmd/app/main.go` — removes the mid-install crash-recovery
  block, the `install.RepairAuthContract(...)` call. Adds explicit
  pre-v1-no-repair comment.
- `repos/gospa/db/migrations/` — three new migrations (companies extension +
  slug index tightening; contacts; workspace_grants; final slug drop in
  Wave 2 = four total; the previously-proposed `00004_add_workspace_admin_user_id`
  is removed and the rest renumbered).
- `repos/gospa/web/src/styles/` — design system entry already shipped.
- `repos/gospa/web/src/components/` — port of the Claude Design component
  bundle (Avatar, RoleBadge, StatusBadge, Kebab, MenuItem, Toast,
  InviteSlideover, OneTimePasswordModal, SearchBar, MiniStats, KbdLegend,
  MemberRow, TableHeader, Shell, Sidebar, TopBar, CommandPalette
  skeleton).
- `repos/gospa/web/src/routes/settings/team*` — new.
- `repos/gospa/web/src/routes/settings/workspace.tsx` — edits the MSP company.
- `repos/gospa/web/src/routes/companies/*` — gains address UI; drops slug.
- `repos/gospa/web/src/lib/command-registry.ts` — new (foundation for future
  Cmd+K).

### Cross-repo

- `repos/gofra` — **none**. The Gospa-only authz model means Gospa never
  needs to read JWT claims beyond `sub`, which `runtime/auth.User.ID`
  already exposes.

### Operational

- **No upgrade path from older schemas.** Pre-v1, every schema change
  assumes a fresh install. Workspaces installed under earlier schemas
  must be reset (`mise run infra:reset` + reinstall). This is the
  documented contract — see `repos/gospa/AGENTS.md`.
- `/install` token requirement, gate fail-closed semantics, PAT hot-reload,
  and S15 cleanup pattern are unchanged.
- The blueprint at `repos/gospa/docs/blueprint/index.md` will gain a new
  Apêndice section "MVP cuts (2026-04)" documenting what this change
  defers (sites, ABAC, email invite, 7-role model, OIDC introspection)
  and what signal would re-open each. The blueprint vision itself stays
  intact; the deferrals are pragmatic, not architectural.

### Decisions accepted (kept across review cycles)

- **A. Role staleness — RESOLVED, NOT a trade-off any more.** Under
  Gospa-only authz, role is read from the DB on every request. There is
  no JWT-embedded role and therefore no staleness window.
- **B. Out-of-band ZITADEL users**: middleware **rejects** with 401 +
  clear error code when an authenticated user has no matching `contacts`
  row. No auto-provisioning.
- **C. Orphan user cleanup**: invite handler runs `RemoveUser` on `defer`
  if the post-ZITADEL DB insert fails — same pattern as S15 (`RemoveOrg`).
- **D. Last-admin protection**: app-level invariant — cannot demote, suspend,
  or archive the only `admin`-grant contact. Surfaced as a clear error.
- **E. ~~Admin user id persisted~~ — DROPPED.** Originally added to
  support the team read-repair. Read-repair removed; column removed.
- **F. Lifecycle activation**: on the first successful authenticated request
  from an invited member, the middleware flips the grant `status` from
  `not_signed_in_yet` to `active`. Removes the need for a separate "accept
  invite" RPC.
- **G. Suspend is Gospa-only**: suspending a member sets
  `workspace_grants.status = 'suspended'`. The ZITADEL user remains active
  and able to authenticate; the Gospa middleware rejects them. A future
  `RemoveMember` action will additionally call `ZITADEL.DeactivateUser` for
  the "kill all access" case.
- **H. No repair before v1.** Recorded as a hard rule in
  `repos/gospa/AGENTS.md`. Every schema or state change assumes fresh
  install. Read-repair, backfill, mid-install recovery, and any other
  "the app self-heals" pattern are out of scope until v1 is shipped and
  there is real production data to protect.

### Out of scope (explicit, to prevent re-litigation)

- Sites.
- Client portal login (contacts that authenticate).
- ABAC / fine-grained permissions matrix.
- Bulk operations on team or contacts.
- MFA enforcement (operator-configurable in ZITADEL).
- Audit log / event log of role changes.
- Cmd+K palette UI (the registry foundation lands; the palette itself does not).
- SCIM / Entra ID federation (the schema columns `identity_source` and
  `external_id` are added now to avoid future migrations, but no sync code).
- ZITADEL project roles, user grants, and any token-introspection-based
  authorization. Gospa owns authz end-to-end.
- "Hard kill" of a team member's identity at ZITADEL — see decision G.
- **Read-repair, backfill, migration data shims, mid-install recovery** —
  see decision H.
