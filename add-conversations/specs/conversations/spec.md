# Spec Delta

## Purpose

Holds the chat conversations of a stage: who opened one, which one is open, who may write to it, and what a closed conversation keeps, so the agent and the UI share one durable record of the dialogue.

## ADDED Requirements

### Requirement: A Maker opens a conversation on a started stage

The system SHALL provide `createConversation`, accepted from a current Maker, taking `operation_id` and targeting a stage of the project.
It SHALL create a conversation bound to the project, the stage instance and the caller as writer, with status `open`, and return its identity, status and `seq` 0.

#### Scenario: First conversation on the stage in progress

- **WHEN** a Maker opens a conversation on a project's Ideation stage, which is `InProgress` and has no conversation
- **THEN** the system responds `201` with the conversation's identity, status `open`, the caller as writer and `last_seq` 0

#### Scenario: Stage not started

- **WHEN** a Maker opens a conversation on the Requirements stage while it is `NotStarted`
- **THEN** the system responds `409` with code `STAGE_NOT_STARTED` and the stage as `current`, and no conversation, receipt or obligation is written

#### Scenario: Exact retry

- **WHEN** a Maker repeats a successful `createConversation` with the same `operation_id` and stage
- **THEN** the response carries the same conversation identity, and no new conversation, receipt or obligation exists

#### Scenario: Reused operation_id with a different stage

- **WHEN** a Maker repeats `createConversation` with the same `operation_id` but another stage
- **THEN** the system responds `409` with code `OPERATION_ID_REUSED` and nothing is written

### Requirement: A stage has at most one open conversation

Opening a conversation on a stage that already has an open one SHALL close that one with reason `replaced` and a reference to the new conversation, in the same transaction that opens the new one.
At no time SHALL a stage have two open conversations.

#### Scenario: Opening a new conversation replaces the current one

- **WHEN** a stage has an open conversation and a Maker opens another one
- **THEN** the earlier conversation is `closed` with reason `replaced` and references the new one, the new one is `open`, and both facts come from one committed receipt

#### Scenario: Two Makers open at once

- **WHEN** two Makers open a conversation on the same stage concurrently on separate connections
- **THEN** both commands succeed, exactly one conversation is open afterwards, and the other is `closed` with reason `replaced`

#### Scenario: Replacement by another Maker

- **WHEN** `alice` has the open conversation of a stage and `bob`, another Maker, opens one
- **THEN** `alice`'s conversation is closed with reason `replaced` and `bob`'s is open with `bob` as writer

### Requirement: A closed conversation keeps everything

Closing a conversation SHALL change only its status, reason, close time and replacement reference.
Its messages SHALL remain readable in full by every Maker of the project.

#### Scenario: Reading a replaced conversation

- **WHEN** a conversation with three messages is replaced and a Maker reads it and its history
- **THEN** the conversation is returned with status `closed`, reason `replaced`, `last_seq` 3, and all three messages are returned unchanged

### Requirement: Conversations of a stage are listed with the open one marked

The system SHALL list the conversations of a stage, newest first, with each conversation's status, writer, open and close times, close reason and `last_seq`, and SHALL identify the open conversation if there is one.
The list SHALL carry the stage's interview status.

#### Scenario: Listing after a replacement

- **WHEN** a stage has had two conversations, the second replacing the first, and a Maker lists them
- **THEN** the second is listed first and marked open, the first follows with status `closed` and reason `replaced`, and the stage's interview status is included

#### Scenario: Listing a stage without conversations

- **WHEN** a Maker lists the conversations of a stage that never had one
- **THEN** the system returns an empty list, no open conversation and the stage's interview status

### Requirement: Reading a conversation

The system SHALL return one conversation to a current Maker of its project with its identity, project, stage instance and code, writer, status, open and close times, close reason, replacement reference, `last_seq` and the stage's interview status.

#### Scenario: Non-member reads a conversation

- **WHEN** an authenticated user who is not a Maker of the project reads a conversation
- **THEN** the system responds `403` with the fixed body

#### Scenario: Unknown conversation

- **WHEN** a Maker reads a conversation identity that does not exist
- **THEN** the system responds `403` with the same fixed body as for a forbidden one
