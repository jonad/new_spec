# Spec Delta

## Purpose

Tells connected clients that something changed, fast, without ever being the source of truth: hints carry identities and sequence numbers, and clients reconcile with reads.

## ADDED Requirements

### Requirement: Hints carry identity, never content

A hint SHALL carry an event identity, the project, the originating `operation_id`, a kind, and the identities and `seq` or status of the affected entities.
A hint SHALL NOT carry message text or any other content.

#### Scenario: Message hint

- **WHEN** a message is appended
- **THEN** a hint of kind `message.appended` is published with the conversation identity and the message `seq`, and without the text

#### Scenario: Conversation hints

- **WHEN** a conversation replaces another
- **THEN** hints `conversation.closed` for the old one and `conversation.opened` for the new one are published with their identities and the stage instance

#### Scenario: Interview hint

- **WHEN** an interview is finished
- **THEN** a hint `interview.finished` is published with the stage instance identity

### Requirement: Hints are published only after commit

A hint SHALL be published only after the transaction that produced it has committed, from its recorded obligation, and the publication SHALL be atomic with the obligation's delivered mark.

#### Scenario: Commit, then publish

- **WHEN** a command commits and its hint is delivered
- **THEN** a subscriber receives the hint after the command's committed rows are readable

#### Scenario: Commit without publish

- **WHEN** a command commits and the process dies before its hint is published
- **THEN** a client that reads messages after its last `seq` sees the message, and the reconciler publishes the hint later

### Requirement: Makers subscribe to a project's events

The system SHALL serve `GET /projects/{id}/events` as a server-sent event stream to a current Maker of the project, sending a `ready` event first, then every hint for the project as it is published, and a heartbeat at a fixed interval.
The stream SHALL NOT replay earlier hints; a client catches up with reads.

#### Scenario: Subscribe and receive

- **WHEN** a Maker subscribes and the writer then posts a message
- **THEN** the subscriber receives `ready` and then the `message.appended` hint with that `seq`

#### Scenario: Other projects are not delivered

- **WHEN** a Maker subscribes to project A and a message is appended in project B
- **THEN** the subscriber receives no hint for project B

#### Scenario: No replay

- **WHEN** a client subscribes with a `Last-Event-ID` header
- **THEN** the stream starts with `ready` and sends no earlier hint

### Requirement: Subscriptions are authorized at open and on delivery

Opening a stream SHALL require a valid session and current membership, evaluated like a read.
Before each hint is delivered, and at a bounded interval while idle, the session and membership SHALL be checked again, and the stream SHALL end when either is no longer valid.

#### Scenario: Non-member subscribes

- **WHEN** an authenticated user who is not a Maker of the project subscribes
- **THEN** the system responds `403` with the fixed body and no stream is opened

#### Scenario: Membership revoked while subscribed

- **WHEN** a subscribed Maker's membership is revoked and committed
- **THEN** the stream ends within the bounded interval and no later hint is delivered

#### Scenario: Session ends while subscribed

- **WHEN** a subscribed Maker's session ends
- **THEN** the stream ends within the bounded interval
