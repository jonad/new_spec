# Spec Delta

## ADDED Requirements

### Requirement: Conversation writes require the conversation's writer

`postMessage` SHALL be accepted only from the `user` caller who opened the conversation and who is a current Maker of its project, checked inside the write's own transaction.
Any other caller SHALL receive `403` with the fixed body and nothing SHALL be written.

#### Scenario: Another Maker posts

- **WHEN** `bob`, a current Maker who did not open the conversation, posts a message to it
- **THEN** the system responds `403` and no message, receipt or obligation is written

#### Scenario: System caller posts

- **WHEN** a provisioned system principal posts a message to a conversation
- **THEN** the system responds `403` and nothing is written

#### Scenario: Writer loses membership before the post

- **WHEN** the writer's membership is revoked and committed, and the writer then posts
- **THEN** the system responds `403` and nothing is written

### Requirement: Opening a conversation and finishing an interview require Maker membership

`createConversation` and `finishInterview` SHALL be accepted only from a current Maker of the project, checked inside the write's own transaction.
System callers and non-members SHALL receive `403` with the fixed body.

#### Scenario: Non-member opens a conversation

- **WHEN** an authenticated user who is not a Maker of the project opens a conversation on one of its stages
- **THEN** the system responds `403` and nothing is written

#### Scenario: System caller finishes an interview

- **WHEN** a provisioned system principal sends `finishInterview`
- **THEN** the system responds `403` and the interview status is unchanged
