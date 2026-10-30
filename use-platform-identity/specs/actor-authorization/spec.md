# Spec Delta

## ADDED Requirements

### Requirement: Callers are identified by a Platform session token

Every request SHALL carry a Platform-issued session token in the `X-Session-Token` header.
The server SHALL resolve that token through Platform's owner interface in the shared database before it reads the request body, and the actor and caller kind (`user` or `system`) SHALL come only from the resolved session.
A request whose token is missing, unknown, ended or expired SHALL be rejected with `401`, with no detail about any resource and without the body being read.

#### Scenario: Missing token

- **WHEN** a request arrives without an `X-Session-Token` header
- **THEN** the system responds `401` with the fixed body and performs no lookup or mutation, whatever the body holds

#### Scenario: Unknown token

- **WHEN** a request carries a token that names no session
- **THEN** the system responds `401` with the fixed body

#### Scenario: Expired session

- **WHEN** a request carries the token of a session whose expiry has passed by the database clock
- **THEN** the system responds `401` with the fixed body

#### Scenario: Ended session

- **WHEN** Platform ends a session and a request then carries its token
- **THEN** the system responds `401` with the fixed body

#### Scenario: Bearer tokens are no longer accepted

- **WHEN** a request carries only an `Authorization: Bearer` header
- **THEN** the system responds `401` with the fixed body

### Requirement: Session validity is checked at the moment of the effect

For a write, the caller's session SHALL be checked again inside the mutation's transaction, ordered against a concurrent end of that session, so an end that commits first prevents the write and an end that starts later waits for it.
For a read, the session SHALL be evaluated in the same snapshot as the data returned.
A session that is not valid at that moment SHALL be refused with `401`, and nothing SHALL be mutated or disclosed.

#### Scenario: Session ends before the write checks it

- **WHEN** a session is ended and committed while a write under that session is in flight but before the write's transaction checks the session
- **THEN** the write responds `401` and nothing is created or changed

#### Scenario: Session end waits for an in-flight write

- **WHEN** a write's transaction has checked the session and Platform then ends that session before the write commits
- **THEN** the write commits and the end takes effect afterwards

#### Scenario: Replay after the session ended

- **WHEN** a caller completes a command, the session is ended, and the same token retries with the same `operation_id` and inputs
- **THEN** the system responds `401`, the original outcome is not disclosed and no row changes

#### Scenario: Read after the session ended

- **WHEN** a session is ended and committed and a read under that session then runs
- **THEN** the system responds `401`

### Requirement: Membership and provisioning facts belong to Platform

Whether an actor is a current project member or a provisioned system principal SHALL be read from Platform's current facts through the owner interface, inside the transaction or snapshot of the effect.
Workspace SHALL keep no membership or principal records of its own.

#### Scenario: Platform grants a Maker

- **WHEN** Platform records `alice` as a Maker of a project and `alice` reads one of its artifacts
- **THEN** the read succeeds

#### Scenario: Platform revokes a Maker

- **WHEN** Platform removes `alice`'s membership and commits, and `alice` then reads an artifact of that project
- **THEN** the system responds `403`

#### Scenario: Platform provisions a system principal

- **WHEN** Platform provisions `svc-platform` with a session and it sends `initializeWorkspace`
- **THEN** the command is accepted

## REMOVED Requirements

### Requirement: Callers are identified by a verified bearer token

**Reason**: Platform issues opaque session tokens and verifies them in the shared database; Workspace no longer verifies signed tokens of its own.
**Migration**: Send the Platform session token in `X-Session-Token` instead of `Authorization: Bearer`. Locally, `make session` replaces `make token`.
