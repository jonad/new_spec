# Spec Delta

## Purpose

Defines how Workspace reaches Platform-owned identity and authority facts in the shared database: only through Platform's contract functions, with grants that forbid anything else, migrations that never touch Platform's schema, a pinned contract that the tests check, and a local stand-in that implements it.

## ADDED Requirements

### Requirement: Platform facts are reached only through contract functions

Workspace SHALL obtain session, membership and system-principal facts only by executing the contract functions in the `platform` schema.
The application role SHALL hold `USAGE` on that schema and `EXECUTE` on the contract functions, and SHALL hold no privilege on any table, view or other function in it.

#### Scenario: Application role executes a contract function

- **WHEN** the application role calls `platform.resolve_session` with the token of a session the stand-in issued
- **THEN** the session's identity, actor id, caller kind, issue time and expiry are returned

#### Scenario: Application role cannot read Platform tables

- **WHEN** the application role selects from any table in the `platform` schema
- **THEN** the database refuses with a permission error

#### Scenario: Application role cannot administer the stand-in

- **WHEN** the application role calls a stand-in administration function, such as creating a session or granting a membership
- **THEN** the database refuses with a permission error and nothing changes

### Requirement: Workspace migrations never touch the platform schema

No Workspace migration SHALL create, alter, drop or grant on any object in the `platform` schema.

#### Scenario: Upgrade and downgrade leave the schema untouched

- **WHEN** the stand-in is loaded and the migrations are upgraded to head and then downgraded to base
- **THEN** the objects in the `platform` schema and their definitions are identical before and after each step

### Requirement: The contract is pinned and checked

The repository SHALL record the contract as the exact signature of each function (name, argument types, result type, volatility and security mode), and the test suite SHALL fail when the loaded `platform` schema differs from that record.

#### Scenario: Stand-in matches the record

- **WHEN** the stand-in is loaded into a test database
- **THEN** the pinned-contract test passes

#### Scenario: Drift is detected

- **WHEN** a contract function in the loaded schema has a different signature, result type, volatility or security mode than the record, or is missing
- **THEN** the pinned-contract test fails and names the function

### Requirement: A local stand-in implements the contract

The database used by docker-compose and by the test suite SHALL load a stand-in that implements every contract function with the contract's semantics, plus administration functions to create and end sessions, grant and revoke membership, and provision and deprovision system principals, executable by the schema-owning role only.

#### Scenario: Stand-in session lifecycle

- **WHEN** the owner role creates a session for a user, Workspace resolves its token, and the owner role then ends the session
- **THEN** the first resolution returns the session and a resolution after the end returns nothing

#### Scenario: Expiry is judged by database time

- **WHEN** a session is created whose expiry lies in the past according to the database clock
- **THEN** resolving its token returns nothing

#### Scenario: Loading twice is harmless

- **WHEN** the stand-in file is loaded into a database that already holds it
- **THEN** loading succeeds and existing sessions, memberships and principals are kept
