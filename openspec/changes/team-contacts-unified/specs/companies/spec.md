## ADDED Requirements

### Requirement: Workspace company representation

The system SHALL represent the MSP itself as a `companies` row with
`is_workspace_owner = TRUE` and `zitadel_org_id` equal to
`workspace.zitadel_org_id`. Exactly one such row SHALL exist after
installation completes, enforced by a partial unique index.

#### Scenario: Fresh install creates the workspace company

- **WHEN** `/install` completes and `workspace.install_state` becomes
  `ready`
- **THEN** exactly one `companies` row SHALL exist with
  `is_workspace_owner = TRUE`

#### Scenario: List companies excludes the workspace company

- **WHEN** any caller invokes `CompaniesService.ListCompanies`
- **THEN** the response SHALL NOT include the row where
  `is_workspace_owner = TRUE`

#### Scenario: Archive of the workspace company is rejected

- **WHEN** any caller invokes `CompaniesService.ArchiveCompany` with
  the `id` of the workspace company
- **THEN** the system SHALL respond `409 FailedPrecondition` with code
  `workspace_company_archive_forbidden`

#### Scenario: Generic UpdateCompany on the workspace company is rejected

- **WHEN** any caller invokes `CompaniesService.UpdateCompany` with the
  `id` of the workspace company
- **THEN** the system SHALL respond `409 FailedPrecondition` with code
  `workspace_company_update_via_settings_endpoint` so the operator uses
  the dedicated `UpdateWorkspaceCompany` RPC instead

### Requirement: Company address fields

The system SHALL persist optional address fields on every `companies`
row: `address_line1`, `address_line2`, `city`, `region`,
`postal_code`, `country`, `timezone`. The fields SHALL be present in
both `Company` (read) and `CreateCompany` / `UpdateCompany` (write)
RPCs.

#### Scenario: Create a company with full address

- **WHEN** an admin creates a company with all address fields populated
- **THEN** the response SHALL echo every address value

#### Scenario: Create a company with no address

- **WHEN** an admin creates a company with only a `name`
- **THEN** the company SHALL be created and address fields SHALL be
  empty strings (or null for optional fields like `address_line2`)

### Requirement: Update an operational company

The system SHALL allow `admin` and `technician` to edit a non-workspace
company's name and address fields via `CompaniesService.UpdateCompany`.

#### Scenario: Technician updates a company name

- **WHEN** a `technician` calls `UpdateCompany` with a valid `id` and a
  new `name`
- **THEN** the system SHALL persist the new name and return the
  updated company

#### Scenario: Admin updates a company's address

- **WHEN** an `admin` calls `UpdateCompany` with new address fields
- **THEN** the system SHALL persist all provided fields and return the
  updated company

### Requirement: Update the workspace company

The system SHALL provide a dedicated admin-only RPC
`CompaniesService.UpdateWorkspaceCompany` (or equivalent service) that
edits the MSP company. The operation SHALL also propagate the name
change to the ZITADEL org.

#### Scenario: Admin renames the workspace company

- **WHEN** an `admin` calls `UpdateWorkspaceCompany` with a new `name`
- **THEN** the system SHALL update the local row and SHALL update the
  ZITADEL organization name

#### Scenario: Technician calls UpdateWorkspaceCompany

- **WHEN** a `technician` calls `UpdateWorkspaceCompany`
- **THEN** the authz middleware SHALL respond `403 PermissionDenied`
  with code `admin_required`

### Requirement: Role-based authorization on Companies RPCs

The system SHALL enforce role-based authz on `CompaniesService` RPCs
via the central policy map. By default `Create`, `Update`, and `List`
are `authenticated`; `Archive` and `UpdateWorkspaceCompany` are
`admin_only`.

#### Scenario: Technician creates a company

- **WHEN** a `technician` calls `CompaniesService.CreateCompany`
- **THEN** the call SHALL be allowed and the company created

#### Scenario: Technician archives a company

- **WHEN** a `technician` calls `CompaniesService.ArchiveCompany`
- **THEN** the authz middleware SHALL respond `403 PermissionDenied`
  with code `admin_required`

## REMOVED Requirements

### Requirement: Company slug

**Reason**: `slug` was added at MVP-bootstrap time as a "human-friendly
identifier" but no code path consumes it — there is no slug-based
routing, no public URL pattern, no operator workflow that types it.
Confirmed in code search before this change. Keeping a required, unique
field that nothing reads is a tax on every form, every test, every
seed, and every data import.

**Migration**: Two waves.

- *Wave 1* (Slice 2): the `slug` field is removed from the
  `Company` message, from `CreateCompanyRequest`, from the
  `companies` sqlc INSERT, from the install wizard form, and from
  the SPA. The DB column stays and is written as the empty string by
  the next deploy. Slice 0 (the prior schema slice) widens the unique
  partial index to also exclude empty strings (`WHERE archived_at IS
  NULL AND slug != ''`) so concurrent inserts do not collide on `''`.
- *Wave 2* (Slice 7, scheduled at least one week after Wave 1):
  the column and the index are dropped. `sqlc` is regenerated.

**Conditional escape hatch**: If, between this change being approved
and Wave 1 starting, an architectural use of `slug` is discovered (a
partner integration, a documented routing requirement, an operator
workflow, a public-API contract), the slice converts to "rename for
clarity" with the same two-wave structure instead of "remove" — the
end state changes but the migration shape and rollback story do not.
The current investigation found no such use; this clause exists so the
default outcome (removal) does not block discovery.
