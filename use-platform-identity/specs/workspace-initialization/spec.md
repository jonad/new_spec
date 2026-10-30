# Spec Delta

## MODIFIED Requirements

### Requirement: Initialize a project workspace

The system SHALL provide `initializeWorkspace`, accepted only from a system caller, taking `operation_id`, `project_id` (a UUID minted by the Platform creation path) and a non-empty `name`.
Project membership is not an input: Platform's creation path grants the Makers.
On success it SHALL return the project, stage and artifact identities and their initial state.

#### Scenario: Successful initialization

- **WHEN** a system caller sends `initializeWorkspace` with a new `project_id` and a name
- **THEN** the system responds `201` with the project identity, four stage instances and five artifacts, and outcome `initialized`

#### Scenario: Invalid input

- **WHEN** a system caller sends `initializeWorkspace` with an empty name, or a name containing a control character such as NUL or a line break
- **THEN** the system responds `422` and nothing is created

#### Scenario: Makers in the body are refused

- **WHEN** a system caller sends `initializeWorkspace` with an `initial_makers` field
- **THEN** the system responds `422` and nothing is created

### Requirement: Initialization is atomic

The project, stage instances, artifacts, receipt and `workspace.initialized` ledger obligation SHALL commit together or not at all.

#### Scenario: Failure leaves no partial workspace

- **WHEN** initialization fails after some rows are staged but before commit
- **THEN** no project, stage instance, artifact, receipt or ledger obligation for that `project_id` exists

#### Scenario: Audit obligation recorded

- **WHEN** initialization commits
- **THEN** exactly one `workspace.initialized` ledger obligation exists for the project, referencing the initialization receipt

### Requirement: Retried initialization returns the same identities

A retry with the same `operation_id` and the same meaningful inputs SHALL return the original identities and outcome, create nothing and write no new receipt.

#### Scenario: Exact retry

- **WHEN** a system caller repeats a successful `initializeWorkspace` with the same `operation_id`, `project_id` and name
- **THEN** the response carries the same stage and artifact identities as the first, and row counts are unchanged

### Requirement: Reused operation_id with different inputs is refused

A request reusing an existing initialization `operation_id` for the same `project_id` with different meaningful inputs SHALL be refused with `409 OPERATION_ID_REUSED`.

#### Scenario: Same operation_id, different name

- **WHEN** a system caller repeats `initializeWorkspace` with the same `operation_id` and `project_id` but a different name
- **THEN** the system responds `409` with `OPERATION_ID_REUSED`, the stored name is unchanged, `lifecycle_rev` is unchanged and no row is added

## REMOVED Requirements

### Requirement: Initial Makers become project members

**Reason**: Project membership is owned by Platform and is no longer written by Workspace.
**Migration**: Platform's creation path grants the Makers around its call to `initializeWorkspace`. Locally, `make member PROJECT=<id> SUB=<actor>` grants a Maker through the stand-in.
