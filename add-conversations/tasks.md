# Tasks

Work top to bottom; `make verify` must pass before a task is checked.
Each group leaves the suite green and lands its own tests and documentation.

## 1. Schema and domain

- [ ] 1.1 Add the Alembic revision (next free number) creating `conversations`, `messages` and `obligation_deliveries` and adding the four interview columns to `stage_instances`, with the partial unique index on open conversations, the unique (`conversation_id`, `seq`), the `reject_mutation` triggers and insert-only grants on `messages`, and `SELECT, INSERT, UPDATE` grants on the mutable tables. Mirror it in `storage/tables.py`. Verify `tests/storage/test_migrations.py` passes with the new tables in its lists and a downgrade removes them.
- [ ] 1.2 Extend `tests/storage/test_immutability.py` to `messages`: the application role and the owner role can neither UPDATE nor DELETE a message, and inserts succeed. Verify the tests pass.
- [ ] 1.3 Add `domain/conversation.py` (Conversation, Message, ConversationStatus, CloseReason, InterviewStatus) and `is_message_text` in `domain/text.py` with unit tests for every text rule in the messages spec (blank, NUL, U+0007, tab and line breaks accepted, 20,000 accepted, 20,001 refused, lone surrogate refused). Verify `tests/domain` passes.
- [ ] 1.4 Add `CONVERSATION_CLOSED` to `ConflictCode` and to the error contract list in `CLAUDE.md`, and update `tests/domain/test_errors_and_outcomes.py` for the new member. Verify `make verify` passes.
- [ ] 1.5 Draft `docs/decisions/0010-conversations-in-workspace.md` with ownership, one open conversation per stage, single writer, `CONVERSATION_CLOSED` and the interview columns. Verify it cites the proposal's deviations and design Decisions 1 to 3.

## 2. Conversations

- [ ] 2.1 Implement `commands/create_conversation.py` and `storage/conversations.py`: stage lock, `STAGE_NOT_STARTED`, replacement of the open conversation in the same transaction, receipt references and the two hint obligations. Verify with `tests/commands/test_create_conversation.py`: first open, replacement with reason and `replaced_by`, `STAGE_NOT_STARTED` with nothing written, exact replay, `OPERATION_ID_REUSED` on another stage, and the no-op-free outcome references.
- [ ] 2.2 Add the concurrency test: two opens on one stage on separate connections leave exactly one open conversation and both callers succeed; a third connection's post to the replaced conversation sees `CONVERSATION_CLOSED` after commit. Verify with `tests/commands/test_create_conversation_concurrency.py` using `tests/support/concurrency.py`.
- [ ] 2.3 Implement the reads in `commands/read_conversations.py`: list for a stage (newest first, open marked, interview status), read one (with stage facts), with the snapshot pattern and fixed `403` for unknown and forbidden. Verify with `tests/commands/test_read_conversations.py`, including a non-member and an unknown id returning byte-identical `403` bodies.
- [ ] 2.4 Add `api/conversations.py` with `POST` and `GET /projects/{id}/stages/{code}/conversations` and `GET /conversations/{id}`, request and response models, OpenAPI descriptions and the `409` responses. Verify with `tests/api/test_conversations.py` covering status codes, bodies and the `422` for unknown fields.
- [ ] 2.5 Update `docs/decisions/0002-guard-order.md` with the stage-instance and conversation lock steps and the no-cycle argument against item writes. Verify the document's sequences match the implemented commands.

## 3. Messages

- [ ] 3.1 Implement `commands/post_message.py`: conversation lock, writer check (`AccessDeniedError` for other Makers and system callers), `CONVERSATION_CLOSED` with `current` carrying status, reason and the stage's open conversation, `seq` allocation, `last_seq` update, receipt and the `hint.message.appended` obligation. Verify with `tests/commands/test_post_message.py`: `seq` 1, 2, 3, exact replay returns the same message, `OPERATION_ID_REUSED` on different text, retry after a lost response appends nothing, `403` for `bob` and for a system principal with nothing written, and `CONVERSATION_CLOSED` after a replacement with counts unchanged.
- [ ] 3.2 Add the concurrency tests: two posts on one conversation give consecutive `seq` values with no gap or duplicate; a replacement committing before an in-flight post's status check makes the post fail with `CONVERSATION_CLOSED` and nothing written. Verify with `tests/commands/test_post_message_concurrency.py`.
- [ ] 3.3 Implement the history read with `after_seq`, `limit` (default 200, max 500) and `has_more` in one snapshot. Verify with `tests/commands/test_read_messages.py`: catch-up after `seq` 3, paging without gap or repeat, and a message appended during the read never producing a gap.
- [ ] 3.4 Add `GET` and `POST /conversations/{id}/messages` to `api/conversations.py` with the text validators (`422` before any lookup for blank, control characters, over-length and lone surrogates). Verify with `tests/api/test_messages.py`, including that a `422` body never echoes the text.
- [ ] 3.5 Add the revoked-writer test: the writer's membership is revoked and committed, and the post returns `403` with nothing written. Verify it passes with the stand-in revocation helper in use at apply time.

## 4. Stage interview

- [ ] 4.1 Implement `commands/finish_interview.py`: stage lock, `STAGE_NOT_STARTED`, completion with actor, time and the open conversation reference, the no-op path with outcome `already_complete`, receipt and the `hint.interview.finished` obligation, and `lifecycle_rev` untouched. Verify with `tests/commands/test_finish_interview.py` covering every stage-interview spec scenario, including replay, `OPERATION_ID_REUSED` on another stage and the `lifecycle_rev` assertion.
- [ ] 4.2 Add `POST /projects/{id}/stages/{code}/finish-interview` to `api/stages.py` (new router) with its models and responses. Verify with `tests/api/test_stages.py`: `200` with `complete`, `200` with `already_complete`, `409 STAGE_NOT_STARTED`, `403` for a system caller.
- [ ] 4.3 Assert the status flows through the reads: the conversation list and the conversation read show `pending` after initialization and `complete` after finishing, and a replacement after finishing keeps `complete`. Verify by extending `tests/commands/test_read_conversations.py`.

## 5. Obligation delivery

- [ ] 5.1 Make `storage/receipts.insert_obligations` insert a `queued` delivery record per obligation in the same transaction, and return the receipt id in `CommandResult`. Verify with `tests/storage/test_receipts.py`: a record per obligation after commit, none after a failure before commit, and the obligation row still immutable.
- [ ] 5.2 Implement `delivery/deliverer.py` (`claim`, `mark_delivered`, `release`, `deliver_for_receipt`) and `delivery/registry.py` (consumers by kind prefix). Verify with `tests/delivery/test_deliverer.py` using a recording consumer: delivered once with one attempt, an expired claim's late mark is refused and the current claimant's mark wins, a failing consumer returns the record to `queued` with the error and attempt count, and a kind without a consumer stays `queued` with zero attempts.
- [ ] 5.3 Implement `delivery/reconciler.py` (`reconcile_once` with backoff and a per-pass limit, and the lifespan loop controlled by `PRODUCT_RECONCILE_INTERVAL_SECONDS`) and the tracked background trigger in the API layer after each command. Verify with `tests/delivery/test_reconciler.py`: a command run with the trigger disabled leaves `queued`, one pass delivers it exactly once; a record with an expired claim is re-driven; backoff delays a recently failed record; and `tests/api/test_app.py` shows the response of a command does not wait for or fail on a failing consumer.
- [ ] 5.4 Add `delivery/lag.py` and `make lag` printing queued and expired-claimed counts per kind with the oldest age, plus `PRODUCT_RECONCILE_INTERVAL_SECONDS` and `PRODUCT_HINT_CHANNEL` in `config.py`, `.env.example` and the README configuration section. Verify `make lag` lists the `workspace.initialized` obligation of a freshly initialized local project.
- [ ] 5.5 Draft `docs/decisions/0011-obligation-delivery.md` with the claim semantics, the background trigger plus reconciler, and what stream 5 adds. Verify it matches design Decision 5.

## 6. Hints and the event stream

- [ ] 6.1 Implement the hint consumer in `delivery/hints.py`: payload from the obligation, `mark_delivered` and `NOTIFY` in one transaction, registered for the `hint.` prefix. Verify with `tests/delivery/test_hints.py` using a real `LISTEN` connection: the notification arrives after the mark commits, carries identities and `seq` and no text, and nothing is notified when the mark fails.
- [ ] 6.2 Implement `delivery/hint_bus.py`: one `LISTEN` connection per process, per-project subscriber queues with a bound, `resync` on overflow and on reconnect. Verify with `tests/delivery/test_hint_bus.py`: a subscriber receives hints for its project only, an overflowed subscriber receives `resync`, and a dropped connection reconnects and sends `resync`.
- [ ] 6.3 Implement `GET /projects/{id}/events` in `api/events.py`: authorization before headers, `ready`, hints as `id`/`event`/`data` records, heartbeat comments, the per-delivery and idle rechecks that end the stream, and `Last-Event-ID` ignored. Verify with `tests/api/test_events.py` using httpx streaming against the ASGI app and shortened intervals: `ready` then `message.appended` after a post, no hint from another project, `403` without a stream for a non-member, the stream ending after a revocation, and the stream starting with `ready` despite a `Last-Event-ID` header.
- [ ] 6.4 Add the commit-without-publish test: run a command with the trigger disabled, read messages after the last `seq` and see the message, then reconcile and see the hint arrive. Verify with `tests/api/test_events_recovery.py`.
- [ ] 6.5 Document the event stream in the README (how to subscribe with `curl -N`, the event names, the `resync` rule and the catch-up read) and describe the endpoint in OpenAPI. Verify the documented `curl` shows `ready` and a hint when a message is posted in another terminal.

## 7. Documentation

- [ ] 7.1 Extend the README API walkthrough: open a conversation, post a message, read history after a `seq`, list the stage's conversations, finish the interview, open a second conversation and show the first is `closed` with `replaced`. Verify every command runs as written against docker-compose.
- [ ] 7.2 Update `CLAUDE.md`: stream 7 "Conversations" in the delivery order with its three changes (`add-conversations`, `add-state-package`, `add-turns`), the `CONVERSATION_CLOSED` code, and a storage rule naming `obligation_deliveries` as the delivery-claim record. Verify the file reads consistently with the proposal.
- [ ] 7.3 Finalize decisions 0010 and 0011 and cross-link them from 0002. Verify each links to the others and to the change.

## 8. Integration

- [ ] 8.1 Run `make verify` from a clean checkout with `docker compose down -v`, `make setup`, `make migrate`, then the README walkthrough end to end including the event stream in a second terminal and `make lag` showing an empty hint backlog. Verify every status and event matches the README.
- [ ] 8.2 Run `openspec validate add-conversations --strict` and confirm every task above is checked. Verify the output reports no issues.

## Workflow follow-up

- Propose `add-turns`, the third change of the Conversations stream, after `add-state-package`: turns, the AI-service claim contract, assistant messages and the `turn.run` obligation on top of this delivery mechanism.
- Archive the change after review; the `actor-authorization` and `immutable-records` deltas then merge into the main specs.
