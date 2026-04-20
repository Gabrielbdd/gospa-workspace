## ADDED Requirements

### Requirement: Contact directory primitive

The system SHALL maintain a `contacts` table where each row represents
a person associated with exactly one company. Each row SHALL carry a
required `full_name` and optional `job_title`, `email`, `phone`,
`mobile`, `notes`. Identity (`zitadel_user_id`) SHALL be nullable so
that contacts who do not log in remain valid.

#### Scenario: Insert a minimal contact

- **WHEN** a caller inserts a contact with only `company_id` and
  `full_name`
- **THEN** the row SHALL be valid and persisted

#### Scenario: Insert a contact with all optional fields

- **WHEN** a caller inserts a contact with all optional fields populated
  (job title, email, phone, mobile, notes)
- **THEN** the row SHALL be persisted with each value preserved as
  provided

#### Scenario: Insert two contacts with the same email at different companies

- **WHEN** two contacts are inserted at different companies, both with
  the same email
- **THEN** both inserts SHALL succeed (uniqueness is per company, not
  global)

#### Scenario: Insert two contacts with the same email at the same company

- **WHEN** a second contact is inserted at the same company with an
  email matching an existing non-archived contact at that company
- **THEN** the insert SHALL fail with a unique-constraint violation
  surfaced as `409 AlreadyExists`

### Requirement: List contacts of a company

The system SHALL list contacts belonging to a given company,
optionally filtered by archive status. Archived contacts SHALL be
excluded by default.

#### Scenario: Admin lists active contacts

- **WHEN** an `admin` calls `ContactsService.ListContacts` with a
  `company_id`
- **THEN** the response SHALL include every contact at that company
  whose `archived_at IS NULL`

#### Scenario: Technician lists contacts

- **WHEN** a `technician` calls `ListContacts` with a `company_id`
- **THEN** the response SHALL be identical to the admin response (read
  access is `authenticated`)

### Requirement: Create, update, archive a contact

The system SHALL allow an admin or technician to create, update, and
archive contacts inside a company. Archive SHALL be a soft delete that
preserves the row for historical attribution.

#### Scenario: Technician creates a contact

- **WHEN** a `technician` calls `ContactsService.CreateContact` with a
  valid `company_id` and `full_name`
- **THEN** the contact SHALL be inserted and returned

#### Scenario: Admin updates a contact's job title

- **WHEN** an `admin` calls `ContactsService.UpdateContact` with a
  contact's `id` and a new `job_title`
- **THEN** the row SHALL be updated and the new value returned

#### Scenario: Admin archives a contact

- **WHEN** an `admin` calls `ContactsService.ArchiveContact` with a
  contact's `id`
- **THEN** `archived_at` SHALL be set to the current timestamp and the
  contact SHALL no longer appear in default `ListContacts` responses

#### Scenario: Update or archive of a workspace-grant-bearing contact

- **WHEN** an admin calls `ArchiveContact` with the `id` of a contact
  that has a `workspace_grants` row
- **THEN** the operation SHALL be rejected with `409 FailedPrecondition`
  and code `team_member_archive_via_team_endpoint` so that team-member
  removal goes through `TeamService.SuspendMember` instead

### Requirement: No commercial fields on contacts

The system SHALL not expose sales, billing, or scoring fields on the
contact surface in this slice. The contact response SHALL contain only
the agreed minimal operational fields.

#### Scenario: Contact detail is loaded

- **WHEN** a caller reads a contact via `GetContact` or `ListContacts`
- **THEN** the response SHALL contain only `id`, `company_id`,
  `full_name`, `job_title`, `email`, `phone`, `mobile`, `notes`,
  `created_at`, `archived_at`
- **AND** SHALL NOT contain lead status, billing flags, sales-owner
  fields, customer-portal preferences, or commercial scoring

### Requirement: Identity-source columns persisted but not exposed

The system SHALL persist `identity_source` (default `manual`) and
`external_id` (nullable) on every contact row, so that future
sync code (Microsoft 365, SCIM) can populate and reconcile externally
managed contacts without a schema migration. These columns SHALL NOT
be surfaced in the V1 RPC responses.

#### Scenario: Manual contact creation defaults the source

- **WHEN** a contact is created via `CreateContact`
- **THEN** the persisted row SHALL have `identity_source = 'manual'`
  and `external_id IS NULL`

#### Scenario: Reading a contact does not surface identity source

- **WHEN** any list, get, or update RPC returns a contact in V1
- **THEN** the response message SHALL NOT contain `identity_source`
  or `external_id` fields; the columns are storage-only preparation
  for a future sync slice
