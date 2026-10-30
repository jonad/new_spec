# Spec Delta

## Purpose

Records what is said in a conversation as an append-only sequence, so that history is complete, ordered and never rewritten, and so that readers and the event stream can catch up from any point.

## ADDED Requirements

### Requirement: Messages are appended with a gapless sequence

Every message SHALL belong to one conversation and carry a `seq` that starts at 1 for the conversation's first message and increases by exactly 1 per message.
The conversation's `last_seq` SHALL equal the `seq` of its latest message.

#### Scenario: Three messages in a row

- **WHEN** the writer posts three messages to a new conversation
- **THEN** they carry `seq` 1, 2 and 3, and the conversation's `last_seq` is 3

#### Scenario: Concurrent posts on one conversation

- **WHEN** the writer posts two messages concurrently on separate connections
- **THEN** both succeed, their `seq` values are consecutive with no gap or duplicate, and `last_seq` equals the higher one

### Requirement: The writer posts text messages

The system SHALL provide `postMessage`, accepted from the conversation's writer, taking `operation_id`, `kind` `text` and `text`.
It SHALL append the message with role `user` and return its identity and `seq`.

#### Scenario: Writer posts

- **WHEN** the writer posts a `text` message to their open conversation
- **THEN** the system responds `201` with the message identity, `seq` and the time it was recorded

#### Scenario: Exact retry

- **WHEN** the writer repeats a successful `postMessage` with the same `operation_id` and text
- **THEN** the response carries the same message identity and `seq`, and the conversation has no additional message, receipt or obligation

#### Scenario: Reused operation_id with different text

- **WHEN** the writer repeats `postMessage` with the same `operation_id` but different text
- **THEN** the system responds `409` with code `OPERATION_ID_REUSED`, and `last_seq` and the message count are unchanged

#### Scenario: Retry after a lost response

- **WHEN** a `postMessage` commits but its response is lost, and the writer retries with the same `operation_id` and text
- **THEN** the system returns the committed message and appends nothing

### Requirement: Message text rules

`text` SHALL be valid Unicode with no lone surrogates, SHALL contain no NUL and no other control character except line feed, carriage return and tab, SHALL be 1 to 20,000 characters long and SHALL contain at least one character that is not whitespace.
Text that breaks a rule SHALL be rejected with `422` and nothing written.

#### Scenario: Blank text

- **WHEN** the writer posts text consisting only of spaces and line breaks
- **THEN** the system responds `422` and the conversation is unchanged

#### Scenario: Control character in text

- **WHEN** the writer posts text containing NUL or U+0007
- **THEN** the system responds `422` and the conversation is unchanged

#### Scenario: Multi-line Markdown is accepted

- **WHEN** the writer posts text with several paragraphs, a list and a code block
- **THEN** the message is appended with the text stored exactly as sent

#### Scenario: Text too long

- **WHEN** the writer posts text of 20,001 characters
- **THEN** the system responds `422` and the conversation is unchanged

### Requirement: Writes to a closed conversation are refused

`postMessage` on a conversation whose status is `closed` SHALL be refused with `409 CONVERSATION_CLOSED`, with the conversation's status and close reason and the stage's open conversation identity, if any, as `current`, and SHALL write nothing.

#### Scenario: Posting after a replacement

- **WHEN** a conversation was replaced and its writer posts to it
- **THEN** the system responds `409` with code `CONVERSATION_CLOSED`, `current` naming the replacing conversation, and the closed conversation's message count and `last_seq` are unchanged

#### Scenario: Close commits before the post checks it

- **WHEN** a `postMessage` is in flight and a replacement of its conversation commits before the post's transaction checks the status
- **THEN** the post responds `409 CONVERSATION_CLOSED` and nothing is written

### Requirement: History is read after a sequence number

The system SHALL return a conversation's messages to a current Maker of the project in `seq` order, starting after a given `seq` (default 0), up to a page limit, with each message's identity, `seq`, role, kind, text, author and time, and whether more messages follow.

#### Scenario: Catch-up read

- **WHEN** a conversation has messages 1 to 5 and a Maker reads after `seq` 3
- **THEN** messages 4 and 5 are returned in order and no more are indicated

#### Scenario: Paging

- **WHEN** a conversation has more messages than the page limit and a Maker reads from the start
- **THEN** one page is returned in order, more is indicated, and reading after the last returned `seq` continues without gap or repeat

#### Scenario: Read in one snapshot

- **WHEN** a message is appended while a history read is executing
- **THEN** the read returns a consistent prefix of the history and never a message whose predecessor by `seq` is missing
