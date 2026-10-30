# Spec Delta

## Purpose

Records, per stage instance, whether the stage's opening interview has been finished, so that the agent knows whether to ask the interview questions or to draft, and so that a new conversation in the same stage does not restart the interview.

## ADDED Requirements

### Requirement: Every stage instance has an interview status

Each stage instance SHALL carry an interview status that is `pending` when the stage instance is created and becomes `complete` only through `finishInterview`.
The status SHALL be returned with a conversation read and with a stage's conversation list.

#### Scenario: Status after initialization

- **WHEN** a project is initialized and a Maker lists the Ideation stage's conversations
- **THEN** the interview status is `pending`

### Requirement: A Maker finishes the interview of a started stage

The system SHALL provide `finishInterview`, accepted from a current Maker, taking `operation_id` and targeting a stage of the project that is `InProgress`.
It SHALL set the stage's interview status to `complete`, record the actor and time, reference the stage's open conversation if there is one, and leave `lifecycle_rev` and the stage's status unchanged.

#### Scenario: Finishing with an open conversation

- **WHEN** the Ideation stage has an open conversation and a Maker finishes its interview
- **THEN** the system responds `200` with status `complete`, the stage references that conversation, and the project's `lifecycle_rev` is unchanged

#### Scenario: Finishing without a conversation

- **WHEN** a stage has no conversation and a Maker finishes its interview
- **THEN** the status becomes `complete` with no conversation reference

#### Scenario: Stage not started

- **WHEN** a Maker finishes the interview of a stage that is `NotStarted`
- **THEN** the system responds `409` with code `STAGE_NOT_STARTED` and the stage as `current`, and the status stays `pending` with no receipt written

#### Scenario: Exact retry

- **WHEN** a Maker repeats a successful `finishInterview` with the same `operation_id`
- **THEN** the response is the same and no new receipt or obligation exists

#### Scenario: Reused operation_id for another stage

- **WHEN** a Maker repeats `finishInterview` with the same `operation_id` but targeting another stage
- **THEN** the system responds `409` with code `OPERATION_ID_REUSED` and nothing changes

### Requirement: Finishing a finished interview is a recorded no-op

`finishInterview` on a stage whose interview is already `complete` SHALL change nothing, SHALL record a no-op receipt for the new `operation_id`, and SHALL return the existing completion with outcome `already_complete`.

#### Scenario: Second finish under a new operation_id

- **WHEN** a stage's interview is complete and a Maker sends `finishInterview` with a new `operation_id`
- **THEN** the system responds `200` with outcome `already_complete`, the original actor, time and conversation reference are returned, and one new no-op receipt exists

### Requirement: A new conversation does not reset the interview

Opening or replacing a conversation on a stage SHALL NOT change the stage's interview status.

#### Scenario: Replacement after a finished interview

- **WHEN** a stage's interview is complete and a Maker opens a new conversation on it
- **THEN** the new conversation is open and the interview status is still `complete`
