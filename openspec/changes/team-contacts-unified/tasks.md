## 0. Slice 0 — Schema base

- [x] 0.1 Write goose migration `00004_extend_companies_workspace_owner_and_address.sql` adding `is_workspace_owner BOOL NOT NULL DEFAULT FALSE`, `address_line1`, `address_line2`, `city`, `region`, `postal_code`, `country`, `timezone` (all TEXT, default `''`), plus a partial unique index `companies_one_workspace_owner ON companies (is_workspace_owner) WHERE is_workspace_owner = TRUE`.
- [x] 0.2 In the same migration as 0.1, widen the existing `companies_slug_active_unique` partial index to also exclude empty strings (`WHERE archived_at IS NULL AND slug != ''`). This MUST land before Slice 2 stops writing meaningful slug values, so that concurrent inserts of `''` do not collide.
- [x] 0.3 Write goose migration `00005_create_contacts.sql` creating `contacts` per the design (id UUID PK, company_id FK, full_name TEXT NOT NULL, job_title TEXT NULL, email TEXT NULL, phone TEXT NULL, mobile TEXT NULL, notes TEXT NULL, zitadel_user_id TEXT NULL, identity_source TEXT NOT NULL DEFAULT 'manual', external_id TEXT NULL, created_at TIMESTAMPTZ NOT NULL DEFAULT now(), archived_at TIMESTAMPTZ NULL) plus indexes: `UNIQUE(company_id, lower(email)) WHERE email IS NOT NULL AND archived_at IS NULL`, `INDEX(company_id) WHERE archived_at IS NULL`, `UNIQUE INDEX(zitadel_user_id) WHERE zitadel_user_id IS NOT NULL`.
- [x] 0.4 Write goose migration `00006_create_workspace_grants.sql` creating the `workspace_role` enum (`'admin','technician'`), the `grant_status` enum (`'active','not_signed_in_yet','suspended'`), and `workspace_grants(contact_id UUID PK FK, role workspace_role NOT NULL, status grant_status NOT NULL DEFAULT 'active', last_seen_at TIMESTAMPTZ NULL, granted_at TIMESTAMPTZ NOT NULL DEFAULT now(), granted_by_contact_id UUID FK NULL)`.
- [x] 0.5 Add sqlc queries in `db/queries/`: extend `companies.sql` (workspace-company filter, address fields, workspace-company getter); new `contacts.sql` (CreateContact only — query-by-zitadel-user-id removed because no read-repair); new `workspace_grants.sql` will land in Slice 1 + Slice 3 (the authz middleware needs `ResolveTeamCallerByZitadelUserID`, the team handler needs CRUD).
- [x] 0.6 Run `mise run gen:sql` and commit the regenerated `db/sqlc/*.go`.
- [x] 0.7 Update the install orchestrator to write the three new constructs (MSP company, admin contact, admin grant with `status='active'`) inside the existing post-SetUpOrg DB transaction. **Do not** call any ZITADEL Management API for project roles or user grants.
- [x] 0.8 Remove the existing `install.RepairAuthContract` package and its wiring (per H rule: no read-repair before v1). Remove `zitadelcontract.DeriveRepair`, the `RepairWorkspaceAuthContract` query, and the mid-install crash-recovery block from `cmd/app/main.go`.
- [x] 0.9 Unit tests: orchestrator post-SetUpOrg materialisation (success + each step failing).
- [x] 0.10 Run `mise run test` and `mise run build`. Manual: `mise run infra:reset` + `mise run migrate` + `mise run dev`. Confirm via `psql` that the new tables exist and only fresh-install path is supported.

## 1. Slice 1 — Authz foundation

- [x] 1.1 Add `internal/authz/policy.go` with the explicit method-to-level map (`map[string]Level` where Level is `LevelAuthenticated` or `LevelAdminOnly`). Seed with every existing private RPC.
- [x] 1.2 Add `internal/authz/context.go` exporting `ContactID(ctx)` / `Role(ctx)` accessors. Unit test the helpers.
- [x] 1.3 (Already done in Slice 0.) `db/queries/workspace_grants.sql` already contains `ResolveTeamCallerByZitadelUserID`, `ActivatePendingGrant`, `TouchLastSeen`.
- [x] 1.4 Add `internal/authz/middleware.go`. The middleware reads `runtime/auth.User` from context, resolves `(contact_id, role, status)` in one query, maps to terminal cases (`401 not_provisioned`, `403 no_grant`, `401 account_suspended`), flips `not_signed_in_yet → active` idempotently, throttles `last_seen_at` (~60s per contact_id, in-memory), and enforces the policy map (`403 admin_required`, `403 policy_undefined`).
- [x] 1.5 Wire the middleware into `cmd/app/main.go` after `authgate.Middleware`. Public RPC list (install, public config) bypass authz exactly as they bypass authgate via the shared `publicProcedures` matcher.
- [x] 1.6 (Decision: requireReady stays as defense-in-depth.) Companies handler does not need policy entries inside the handler — `internal/authz/policy.go` is the single source. requireReady is layered behind the gate's pre-install fail-closed.
- [x] 1.7 Companies handler tests left untouched — handler-internal concerns (Zitadel.AddOrganization, row persistence) don't change. Authz-layer access matrix is covered by the 11 new cases in `internal/authz/middleware_test.go`.
- [ ] 1.8 Manual: complete `mise run infra:reset` + install via UI; confirm `ListCompanies` still works as the install admin. **Pending operator smoke** — backend/test path validated; live SPA path not yet exercised in this slice (no Slice 4 UI yet).
- [x] 1.9 Run `mise run test` and `mise run build`. All green; 11 new authz tests pass.

## 2. Slice 2 — Slug Wave 1 (remove from app surface)

- [ ] 2.1 Edit `proto/gospa/companies/v1/companies.proto`: remove `slug` from `Company` and `CreateCompanyRequest`.
- [ ] 2.2 Edit `proto/gospa/install/v1/install.proto`: remove `workspace_slug`.
- [ ] 2.3 Run `mise run gen:proto`.
- [ ] 2.4 Edit `db/queries/companies.sql` and `db/queries/workspace.sql`: drop `slug` from INSERT/UPDATE column lists. INSERTs implicitly use the column default (empty string).
- [ ] 2.5 Run `mise run gen:sql`.
- [ ] 2.6 Edit `internal/companies/handler.go`: drop slug validation, drop slug from `toProtoCompany`, drop slug from `CreateCompanyParams`.
- [ ] 2.7 Edit `internal/install/handler.go`, `internal/install/orchestrator.go`, `internal/install/params.go`: drop slug references.
- [ ] 2.8 Edit `web/src/routes/install.tsx`, `web/src/lib/install-client.ts`, `web/src/lib/companies-client.ts`: drop slug from forms and types.
- [ ] 2.9 Update tests in `internal/companies/handler_test.go` and `internal/install/*_test.go` to remove slug expectations.
- [ ] 2.10 Update `docs/operations.md` and `docs/examples/deploy/kubernetes/README.md` to remove operator-facing slug mentions.
- [ ] 2.11 Run `mise run test`, `mise run build`, `docker build -t gospa:dev .`. Manual: `mise run infra:reset` + complete a fresh install and confirm no UI field asks for slug.
- [ ] 2.12 Add a one-line note in `docs/operations.md` that the slug column remains in the database temporarily and is scheduled for removal in Slice 7.

## 3. Slice 3 — Team backend

- [ ] 3.1 Create `proto/gospa/team/v1/team.proto` with `TeamService` (`ListMembers`, `InviteMember`, `ChangeRole`, `SuspendMember`, `ReactivateMember`) and message types (`TeamMember`, `MemberStatus` enum: `active`, `not_signed_in_yet`, `suspended`).
- [ ] 3.2 Run `mise run gen:proto`.
- [ ] 3.3 Add ZITADEL client methods in `internal/zitadel/client.go`: **only** `AddHumanUser` (with `passwordChangeRequired`, `isEmailVerified`); `RemoveUser` (for invite cleanup). **Do NOT add** `AddProjectRole`, `AddUserGrant`, `UpdateUserGrant`, `RemoveUserGrant`, `DeactivateUser`, `ReactivateUser`, `GetUserByID`, `ListOrgOwners` — none are used.
- [ ] 3.4 Extend `db/queries/workspace_grants.sql` with handler-side queries: `CountActiveAdmins`, `GetGrantByContactID`, `UpdateGrantStatus`, `UpdateGrantRole`. Run `mise run gen:sql`.
- [ ] 3.5 Create `internal/team/handler.go` with the five RPCs. Use a strong-random 24-char generator (`crypto/rand`) for temporary passwords.
- [ ] 3.6 Implement `InviteMember`: validate input → check email duplicate at MSP company → `AddHumanUser` → DB transaction (insert contact + grant with `role = chosen`, `status = 'not_signed_in_yet'`) → return temp password. `defer` block runs `RemoveUser` if any post-`AddHumanUser` step fails.
- [ ] 3.7 Implement `ChangeRole`: load contact + grant → run last-admin invariant → `UPDATE workspace_grants SET role = $new`. **No ZITADEL call.**
- [ ] 3.8 Implement `SuspendMember` / `ReactivateMember`: same shape, last-admin protection on suspend, flips `workspace_grants.status` in DB. **No ZITADEL deactivate calls.**
- [ ] 3.9 Implement `ListMembers`: `JOIN contacts ON workspace_grants.contact_id = contacts.id` filtered by MSP company.
- [ ] 3.10 Update `internal/authz/policy.go` with the four new RPC entries: `ListMembers` = authenticated, `InviteMember`/`ChangeRole`/`SuspendMember`/`ReactivateMember` = admin_only.
- [ ] 3.11 Wire the new handler into `cmd/app/main.go`.
- [ ] 3.12 Unit tests with a fake ZITADEL: invite-success, invite-DB-failure triggering `RemoveUser`, change-role updates DB only, last-admin protection, suspend flips status without ZITADEL call, suspended member rejected by middleware, technician calling admin RPC returns 403, first-login activation flips status.
- [ ] 3.13 Run `mise run test` and `mise run build`. Manual: `mise run infra:reset` + install + invite a technician via `buf curl`; log in via browser; verify activation flip.

## 4. Slice 4 — Team UI

- [x] 4.1 Design system tokens at `web/src/styles/{index,tokens,typography}.css` already shipped (2026-04-19).
- [ ] 4.2 Create `web/src/lib/command-registry.ts`.
- [ ] 4.3 Create `web/src/lib/team-client.ts` mirroring `companies-client.ts`.
- [ ] 4.4 Port the Claude Design components from `/tmp/claude-design-extract/gospa-psa/project/SettingsTeam.jsx` and `Shell.jsx` into TypeScript React 19 components under `web/src/components/`. **No inline hex codes** — every color/spacing must consume Tailwind utilities or `var(--token)` references.
- [ ] 4.5 Port the shell components: `Shell.tsx`, `Sidebar.tsx`, `TopBar.tsx`, `CommandPalette.tsx`, `SettingsRail.tsx`. Constraints: nav items beyond Settings render disabled with a `Soon` badge.
- [ ] 4.6 Adapt the InviteSlideover copy: remove "They'll get an email"; replace with "We'll generate a temporary password and show it to you once."
- [ ] 4.7 Implement `OneTimePasswordModal`: centered modal, large monospace password block + Copy + required checkbox + Done. No esc/clickaway.
- [ ] 4.8 Remove "Remove from workspace" from Kebab; suspend/reactivate is the only removal path.
- [ ] 4.9 Build `web/src/routes/settings/team.tsx` consuming `team-client.ts` via TanStack Query. Wire keyboard shortcuts via reusable hook `useTableKeyboardNav`.
- [ ] 4.10 Register the four primary verbs in `command-registry`.
- [ ] 4.11 For technicians: render the read-only list; hide invite/change-role/suspend affordances entirely.
- [ ] 4.12 Add minimal route: `/settings` redirects to `/settings/team`.
- [ ] 4.13 Manual: full path — log in as install admin, invite a technician, copy the temp password, log out, log in as the technician with the temp password, ZITADEL forces change.
- [ ] 4.14 Run `npm run build` in `web/`, then `mise run build`.

## 5. Slice 5 — Companies address fields + hide MSP

- [ ] 5.1 Edit `proto/gospa/companies/v1/companies.proto`: add address fields, `UpdateCompany`, `UpdateWorkspaceCompany`.
- [ ] 5.2 Run `mise run gen:proto`.
- [ ] 5.3 Add `db/queries/companies.sql` queries `UpdateCompany`, `UpdateWorkspaceCompany`. Run `mise run gen:sql`.
- [ ] 5.4 Edit `internal/companies/handler.go`: address fields, `UpdateCompany`, `UpdateWorkspaceCompany`, `ListCompanies` filters `is_workspace_owner = FALSE`, `ArchiveCompany` rejects on workspace company.
- [ ] 5.5 Update `internal/authz/policy.go`: `Update`, `Create`, `List` are `authenticated`; `Archive`, `UpdateWorkspaceCompany` are `admin_only`.
- [ ] 5.6 Edit `web/src/routes/companies/*`.
- [ ] 5.7 Add `web/src/routes/settings/workspace.tsx`.
- [ ] 5.8 Update `web/src/lib/companies-client.ts`.
- [ ] 5.9 Unit tests including workspace-company rejection paths and ZITADEL org-rename propagation.
- [ ] 5.10 Run `mise run test` and `mise run build`.

## 6. Slice 6 — Contacts

- [ ] 6.1 Create `proto/gospa/contacts/v1/contacts.proto`. The proto MUST NOT expose `identity_source` or `external_id`.
- [ ] 6.2 Run `mise run gen:proto`.
- [ ] 6.3 Add list/get/update/archive queries to `db/queries/contacts.sql`. Run `mise run gen:sql`.
- [ ] 6.4 Create `internal/contacts/handler.go` with company-scoped email uniqueness; `identity_source = 'manual'` default on insert.
- [ ] 6.5 Add to `internal/authz/policy.go`: all five RPCs are `authenticated`.
- [ ] 6.6 Reject `ArchiveContact` on a contact that has a `workspace_grants` row with code `team_member_archive_via_team_endpoint`.
- [ ] 6.7 Wire handler into `cmd/app/main.go`.
- [ ] 6.8 Unit tests covering CRUD, email uniqueness per company, cross-company duplicate allowed, archive of team-member contact rejected.
- [ ] 6.9 Add `web/src/routes/companies/$companyId.tsx` with a `Contacts` section.
- [ ] 6.10 Register contact verbs in `command-registry`.
- [ ] 6.11 Manual: open a company, add/edit/archive contacts.
- [ ] 6.12 Run `mise run test`, `mise run build`, `docker build`.

## 7. Slice 7 — Slug Wave 2 (drop schema)

- [ ] 7.1 Verify in production logs / metrics (post-v1) that no read of `slug` has occurred since Slice 2 deploy. Pre-v1: skip; just delete and `mise run infra:reset`.
- [ ] 7.2 Write goose migration `00007_drop_slug.sql`: `DROP INDEX companies_slug_active_unique`; `ALTER TABLE companies DROP COLUMN slug`; same for `workspace.slug`.
- [ ] 7.3 Re-run `mise run gen:sql`.
- [ ] 7.4 Search the entire repository for any residual `slug` references (use Grep over `repos/gospa`).
- [ ] 7.5 Run `mise run test`, `mise run build`, `docker build`. `mise run infra:reset` + fresh install.
- [ ] 7.6 Update `docs/operations.md` to remove the temporary note added in 2.12.

## 8. Cross-cutting validation & documentation

- [ ] 8.1 **Blueprint Apêndice C — MVP cuts (2026-04).** Add a new section to `repos/gospa/docs/blueprint/index.md` Apêndice C titled "MVP cuts (2026-04)" listing what this change postpones (sites, ABAC, email invite, 7-role model, OIDC introspection, **read-repair / backfill**) and what signal would re-open each.
- [ ] 8.2 **Update `repos/gospa/README.md`**: reflect the new state (Gospa-owned authz, Team page, Companies with address, Contacts, no read-repair).
- [ ] 8.3 **Update `repos/gospa/docs/stack.md`**: describe the Gospa-owned authz layer (`internal/authz`), `workspace_grants` as the role source of truth.
- [x] 8.4 **AGENTS.md rule.** Add a hard rule to `repos/gospa/AGENTS.md`: "No repair before v1" — no read-repair, no backfill, no migration data shims, no mid-install recovery code. Includes the rationale and the relaxation criterion (real production data after v1).
- [x] 8.5 **Hardening pause override note.** Update `docs/progress/provisioning-zitadel-hardening-plan.md` with a brief addendum: the "pause for real usage" recommendation has been consciously overridden, and the `RepairAuthContract` from S5/S7 has been removed for the same reason (over-engineering pre-v1).
- [ ] 8.6 Open a workspace-level progress file at `docs/progress/team-contacts-unified.md`. Update at the end of each slice.
- [ ] 8.7 Confirm cross-impact: `repos/gofra` impact is **none**.
- [ ] 8.8 Push protocol per slice that touches gospa: gospa first, then workspace bump. No `--force`.
- [ ] 8.9 After Slice 7 lands, archive this change with `openspec archive team-contacts-unified`.
