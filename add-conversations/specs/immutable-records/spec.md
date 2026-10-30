# Spec Delta

## MODIFIED Requirements

### Requirement: Append-only record types

The following records SHALL be append-only: `ItemVersion`s, section definitions, command receipts, ledger obligations and messages.
Once inserted, their contents SHALL NOT change and they SHALL NOT be removed.

#### Scenario: Insert is allowed

- **WHEN** the application inserts a new `ItemVersion`
- **THEN** the insert succeeds

#### Scenario: Message cannot be edited or removed

- **WHEN** the application role, and separately the schema-owning role, attempts to UPDATE or DELETE a message
- **THEN** the database rejects the statement and the message is unchanged
