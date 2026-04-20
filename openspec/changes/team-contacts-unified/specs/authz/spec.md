## ADDED Requirements

### Requirement: Local team-grant resolution from JWT subject

The system SHALL identify the authenticated caller by querying the local
`contacts` table for the row whose `zitadel_user_id` equals the JWT `sub`
claim (already validated and exposed by Gofra's `runtime/auth` as
`User.ID`), then loading the matching `workspace_grants` row. The role
SHALL be read from `workspace_grants.role`. **No JWT claim other than
`sub` SHALL be consulted for authorization.**

#### Scenario: Active team member sends a request

- **WHEN** a JWT-validated request arrives, a contact exists with
  `zitadel_user_id = jwt.sub`, and the matching `workspace_grants.status
  = 'active'`
- **THEN** the middleware SHALL attach `(contact_id, role)` to the
  request context and SHALL proceed to the next handler

#### Scenario: No contact exists for the JWT subject

- **WHEN** a JWT-validated request arrives and no contact row has
  `zitadel_user_id = jwt.sub`
- **THEN** the middleware SHALL respond `401 Unauthenticated` with code
  `not_provisioned` and a Connect-shaped JSON body explaining that the
  user is not provisioned in Gospa, and MUST log the event with
  `zitadel_user_id` and the request method

#### Scenario: Contact exists but has no workspace grant

- **WHEN** a JWT-validated request arrives, a contact matches the
  subject, but no `workspace_grants` row references it (the contact has
  identity but no team grant — reserved for the future client-portal
  case)
- **THEN** the middleware SHALL respond `403 PermissionDenied` with code
  `no_grant`

#### Scenario: Workspace grant is suspended

- **WHEN** a JWT-validated request arrives, a contact matches the
  subject, and the matching `workspace_grants.status = 'suspended'`
- **THEN** the middleware SHALL respond `401 Unauthenticated` with code
  `account_suspended`

### Requirement: First successful authentication activates a pending grant

The system SHALL transition a `workspace_grants` row from
`not_signed_in_yet` to `active` on the first successful authenticated
request from the corresponding contact. The transition MUST be idempotent
and MUST NOT block the request beyond the duration of the activating
UPDATE.

#### Scenario: Invited member completes first successful request

- **WHEN** a JWT-validated request arrives from a contact whose
  `workspace_grants.status = 'not_signed_in_yet'`
- **THEN** the middleware SHALL allow the request to proceed
- **AND** SHALL persist `workspace_grants.status = 'active'` for that
  contact in the same context-resolution transaction

#### Scenario: Already-active member sends a request

- **WHEN** a JWT-validated request arrives from a contact whose
  `workspace_grants.status = 'active'`
- **THEN** the middleware SHALL NOT issue a status update write

### Requirement: Last-seen tracking on each authenticated request

The system SHALL update `workspace_grants.last_seen_at` on authenticated
requests, throttled to no more than one write per ~60 seconds per
`contact_id` so that high-frequency request patterns do not cause write
amplification.

#### Scenario: First request in a 60-second window

- **WHEN** an authenticated request arrives from a member whose
  `last_seen_at` is older than ~60 seconds (or NULL)
- **THEN** the middleware SHALL persist `last_seen_at = now()`

#### Scenario: Subsequent requests within the throttle window

- **WHEN** an authenticated request arrives from a member whose
  `last_seen_at` was updated less than ~60 seconds ago
- **THEN** the middleware SHALL NOT issue a `last_seen_at` write

### Requirement: Per-RPC declarative authorization policy

The system SHALL maintain a single authoritative policy map at
`internal/authz/policy.go` that lists every Connect RPC method and its
required role level (`admin_only` or `authenticated`). RPCs not present
in the map MUST be rejected by default.

#### Scenario: RPC requires admin and caller is admin

- **WHEN** a caller with `role = admin` invokes an RPC declared as
  `admin_only`
- **THEN** the middleware SHALL allow the call to reach the handler

#### Scenario: RPC requires admin and caller is technician

- **WHEN** a caller with `role = technician` invokes an RPC declared as
  `admin_only`
- **THEN** the middleware SHALL respond `403 PermissionDenied` with code
  `admin_required`

#### Scenario: RPC requires authenticated and caller is technician

- **WHEN** a caller with `role = technician` invokes an RPC declared as
  `authenticated`
- **THEN** the middleware SHALL allow the call to reach the handler

#### Scenario: RPC method is not in the policy map

- **WHEN** a caller invokes an RPC method that the policy map does not
  list
- **THEN** the middleware SHALL respond `403 PermissionDenied` with code
  `policy_undefined` and MUST log the event so the missing entry is
  visible

### Requirement: Role changes apply immediately, with no JWT staleness window

The system SHALL ensure that any change to `workspace_grants.role`
applies to the next authenticated request from that contact. There MUST
NOT be a JWT-TTL-bounded staleness window between the database write
and the new role taking effect.

#### Scenario: Admin demotes another admin to technician

- **WHEN** an admin updates another contact's `workspace_grants.role`
  from `admin` to `technician`
- **AND** that contact issues a new request immediately afterwards
  (with their existing access token)
- **THEN** the middleware SHALL apply the new role on that request

#### Scenario: Admin promotes a technician to admin

- **WHEN** an admin updates another contact's `workspace_grants.role`
  from `technician` to `admin`
- **AND** that contact issues a new request immediately afterwards
- **THEN** the middleware SHALL apply the new role on that request

### Requirement: Public RPC bypass preserved

The system SHALL preserve the existing list of public Connect procedures
(install RPCs, public config) so that authz never blocks pre-install
flows or unauthenticated discovery endpoints.

#### Scenario: Install RPC called pre-install

- **WHEN** an `InstallService.*` RPC is invoked before installation has
  completed
- **THEN** the authz middleware SHALL NOT run; the existing `authgate`
  pre-install behavior SHALL apply

#### Scenario: Install RPC called post-install

- **WHEN** an `InstallService.*` RPC is invoked after installation
- **THEN** the authz middleware SHALL NOT run on it; the install
  service remains unauthenticated by definition (it is the bootstrap
  surface)
