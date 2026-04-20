## ADDED Requirements

### Requirement: Materialize MSP company, admin contact, admin grant during install

The system SHALL, in the same transaction that flips `install_state`
to `ready`, insert the MSP `companies` row (with
`is_workspace_owner = TRUE`), the admin `contacts` row (with
`zitadel_user_id` from `SetUpOrg`), and the `workspace_grants` row
with `role = admin` and `status = active`. The system SHALL NOT
create any ZITADEL project roles or `UserGrants`; authorization is
recorded in the local `workspace_grants` row only.

#### Scenario: Successful install yields a complete bootstrap

- **WHEN** `/install` completes successfully
- **THEN** the database SHALL contain (a) the MSP company row, (b) the
  admin contact row linked to that company, (c) the admin grant row
  pointing to that contact with `role = 'admin'` and `status =
  'active'`

#### Scenario: Materialization step fails

- **WHEN** the materialization transaction fails (e.g., DB constraint
  unexpectedly trips)
- **THEN** the install orchestrator SHALL behave as if a post-SetUpOrg
  step failed (run the existing S15 cleanup: `RemoveOrg` on the
  workspace org, mark `install_state = failed`, record
  `install_error`)

#### Scenario: Install does not provision project roles in ZITADEL

- **WHEN** `/install` completes successfully
- **THEN** the ZITADEL project SHALL NOT have any `ProjectRole`
  definitions added by Gospa
- **AND** the admin user SHALL NOT have any ZITADEL `UserGrant` issued
  by Gospa (their existing `ORG_OWNER` grant from `SetUpOrg` is
  unchanged)

### Requirement: Install no longer requires a workspace slug

The install flow SHALL bootstrap the workspace without collecting or
persisting `workspace_slug`. The `workspace.slug` column remains in
the database temporarily (Wave 1 of slug removal); the install handler
SHALL NOT write any meaningful value to it.

#### Scenario: Operator completes install without slug

- **WHEN** the operator submits the install form
- **THEN** the request SHALL require `workspace_name`, timezone,
  currency, and initial admin data
- **AND** SHALL NOT require `workspace_slug`
- **AND** the install handler SHALL NOT validate, accept, or persist
  any meaningful slug value

### Requirement: No read-repair or backfill for legacy installs

The system SHALL NOT attempt to backfill, repair, or self-heal
workspaces that exist in a state the current schema or code does not
expect. Pre-v1, the contract is "fresh install only". Workspaces
installed under an earlier schema MUST be reset
(`mise run infra:reset` + reinstall) — there is no auto-recovery
path.

This requirement is the spec-level expression of the rule recorded in
`repos/gospa/AGENTS.md` ("No repair before v1") and decision H in the
proposal.

#### Scenario: Legacy workspace boots after upgrade

- **WHEN** the app boots against a workspace whose schema or state
  predates the current code (e.g., missing tables, missing rows the
  install bootstrap now writes)
- **THEN** the app SHALL NOT attempt to backfill the missing data
- **AND** the operator SHALL be expected to run
  `mise run infra:reset` and complete a fresh install instead

#### Scenario: Mid-install crash on a previous process

- **WHEN** a previous process exited mid-install and left
  `workspace.install_state = 'provisioning'`
- **THEN** the next process SHALL NOT auto-recover by flipping the
  state to `failed`
- **AND** the operator SHALL be expected to run
  `mise run infra:reset` and reinstall
