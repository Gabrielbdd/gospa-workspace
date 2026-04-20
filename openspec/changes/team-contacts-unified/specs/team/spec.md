## ADDED Requirements

### Requirement: List team members

The system SHALL list all team members of the workspace — every contact
at the MSP company that has a `workspace_grants` row — including their
role and current status.

#### Scenario: Admin lists the team

- **WHEN** an `admin` calls `TeamService.ListMembers`
- **THEN** the response SHALL include every team member with fields
  `contact_id`, `full_name`, `email`, `role`, `status`,
  `last_seen_at` (nullable), `created_at`

#### Scenario: Technician lists the team

- **WHEN** a `technician` calls `TeamService.ListMembers`
- **THEN** the response SHALL include the same fields as for an admin
  (read access is `authenticated`, not `admin_only`, so technicians can
  see who their teammates are)

#### Scenario: Suspended members appear with explicit status

- **WHEN** the team contains a suspended member
- **THEN** that member SHALL appear in the list with `status = suspended`

#### Scenario: Pending members appear with explicit status

- **WHEN** the team contains a member whose grant is `not_signed_in_yet`
- **THEN** that member SHALL appear in the list with
  `status = not_signed_in_yet`

### Requirement: Invite a new team member

The system SHALL allow an admin to invite a new team member by supplying
their full name, email, and role. On success the system SHALL create the
ZITADEL human user with a server-generated temporary password, insert
the contact row at the MSP company, and insert the workspace grant —
all atomically from the operator's perspective. The temporary password
MUST be returned in the RPC response exactly once and never retrievable
afterwards. **No ZITADEL `UserGrant` is created**; the role is recorded
locally in `workspace_grants` only.

#### Scenario: Admin invites a technician with a unique email

- **WHEN** an `admin` calls `TeamService.InviteMember` with a fresh email,
  a name, and `role = technician`
- **THEN** the system SHALL create the ZITADEL user with
  `passwordChangeRequired = true` and `isEmailVerified = true`
- **AND** SHALL insert the contact at the MSP company and the workspace
  grant with `role = technician`, `status = 'not_signed_in_yet'` in one
  DB transaction
- **AND** SHALL return `{ contact_id, temporary_password }` in the
  response, where the password is the only opportunity to obtain it

#### Scenario: Email already exists at the workspace company

- **WHEN** an `admin` calls `InviteMember` with an email that matches an
  existing non-archived contact at the MSP company
- **THEN** the system SHALL respond `409 AlreadyExists` with code
  `email_taken` before any ZITADEL call
- **AND** no ZITADEL user SHALL be created

#### Scenario: ZITADEL user creation succeeds but DB transaction fails

- **WHEN** `AddHumanUser` returns success and the subsequent DB INSERT
  fails (e.g., concurrent invite collided)
- **THEN** the system SHALL call `ZITADEL.RemoveUser` for the just-created
  user before returning the error
- **AND** SHALL respond with the original DB error wrapped as
  `Internal`
- **AND** SHALL log the cleanup outcome at WARN if `RemoveUser` itself
  fails, including the orphan `zitadel_user_id`

#### Scenario: Technician attempts to invite

- **WHEN** a `technician` calls `InviteMember`
- **THEN** the authz middleware SHALL respond `403 PermissionDenied`
  before the handler runs

### Requirement: Change a member's role

The system SHALL allow an admin to change another member's role between
`admin` and `technician`. The change SHALL update only the local
`workspace_grants.role`; **no ZITADEL Management API call is made**.
The new role SHALL take effect on the affected member's next
authenticated request (no JWT-TTL staleness — see authz spec).

#### Scenario: Admin promotes a technician to admin

- **WHEN** an `admin` calls `TeamService.ChangeRole` with another
  member's `contact_id` and `role = admin`
- **THEN** the system SHALL update `workspace_grants.role = 'admin'` and
  SHALL return success
- **AND** the affected member's next authenticated request SHALL be
  evaluated as an admin

#### Scenario: Admin demotes another admin while not being the last admin

- **WHEN** an `admin` calls `ChangeRole` with another admin's
  `contact_id` and `role = technician`, and there is at least one
  remaining active admin (the caller)
- **THEN** the system SHALL allow the change

#### Scenario: Admin attempts to demote themselves while being the last admin

- **WHEN** an `admin` calls `ChangeRole` with their own `contact_id`
  and `role = technician`, and they are the only active admin
- **THEN** the system SHALL respond `409 FailedPrecondition` with code
  `last_admin` and SHALL NOT change anything

### Requirement: Suspend and reactivate a member (Gospa-only)

The system SHALL allow an admin to suspend a team member, which SHALL
set `workspace_grants.status = 'suspended'`. **The ZITADEL user
SHALL NOT be deactivated** — the member's identity remains valid for
authentication, but the Gospa middleware will reject every request
they make (per the authz spec). The system SHALL allow reactivation by
setting `workspace_grants.status = 'active'`.

#### Scenario: Admin suspends a technician

- **WHEN** an `admin` calls `TeamService.SuspendMember` with a
  technician's `contact_id`
- **THEN** the system SHALL set `workspace_grants.status = 'suspended'`
- **AND** SHALL NOT call `ZITADEL.DeactivateUser`
- **AND** SHALL return success

#### Scenario: Suspended member sends a request

- **WHEN** a suspended member sends a request with any access token
  (existing or freshly refreshed from ZITADEL)
- **THEN** the authz middleware SHALL reject it with code
  `account_suspended`

#### Scenario: Admin reactivates a suspended member

- **WHEN** an `admin` calls `TeamService.ReactivateMember` with a
  suspended member's `contact_id`
- **THEN** the system SHALL set `workspace_grants.status = 'active'`
- **AND** the member's subsequent requests SHALL succeed

#### Scenario: Admin attempts to suspend themselves while being the last admin

- **WHEN** an `admin` calls `SuspendMember` with their own `contact_id`
  and they are the only `active`-status admin
- **THEN** the system SHALL respond `409 FailedPrecondition` with code
  `last_admin`

### Requirement: Last-admin invariant

The system SHALL guarantee at all times that at least one team member
exists with `role = admin` and `status = active`. Any operation that
would violate this invariant MUST be rejected with
`409 FailedPrecondition` and code `last_admin`.

#### Scenario: Demoting the last admin

- **WHEN** the operation in flight would result in zero active admins
- **THEN** the system SHALL reject the operation with code `last_admin`

#### Scenario: Suspending the last admin

- **WHEN** the operation in flight would result in zero active admins
- **THEN** the system SHALL reject the operation with code `last_admin`

#### Scenario: Archiving the last admin's contact

- **WHEN** an archive operation in flight would result in zero active
  admins (the contact backing the only active admin grant)
- **THEN** the system SHALL reject the operation with code `last_admin`

### Requirement: Settings > Team UI

The system SHALL provide a `/settings/team` route that lists members,
supports keyboard navigation (`/`, `j/k`, `enter`, `esc`, `n`, `r`,
`s`, `space`), opens a slideover for invite, and shows a one-time
modal containing the temporary password for newly invited members. The
status label canonical form is **"Not signed in yet"** (used in badges,
filters, and stat labels — never abbreviated to "Invited").

#### Scenario: Admin opens the team page

- **WHEN** an admin navigates to `/settings/team`
- **THEN** the page SHALL render the member list with a focused row,
  `j` and `k` SHALL move focus, `enter` SHALL open the detail panel
- **AND** `n` SHALL open the invite slideover

#### Scenario: Admin invites successfully

- **WHEN** the admin submits the invite form with valid input
- **THEN** the UI SHALL display a one-time modal containing the
  temporary password with a copy button
- **AND** dismissing the modal SHALL refresh the member list to show
  the new member with status `Not signed in yet`

#### Scenario: Technician opens the team page

- **WHEN** a technician navigates to `/settings/team`
- **THEN** the page SHALL render the read-only member list (per the
  ListMembers requirement)
- **AND** the invite, change-role, and suspend actions SHALL not be
  visible

#### Scenario: Kebab menu does not offer hard delete

- **WHEN** the kebab menu opens on a team member row
- **THEN** the available actions SHALL be limited to: Open details,
  Change role, Send password reset, Suspend (or Reactivate when
  already suspended)
- **AND** there SHALL NOT be a "Remove from workspace" or "Delete"
  action; suspend is the only removal path
