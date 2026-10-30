# Design: add-conversations

## Context

See proposal.md for motivation.
What shapes the approach:

- Every write already runs through `commands/pipeline.py`: guards in the fixed order, receipt lookup, mutation, receipt and obligations in one short transaction, bounded retry. Obligations are rows in the append-only `ledger_obligations` table with `event_id`, `event_kind`, `payload` and `payload_sha256`. Nothing delivers them yet; the pipeline's step 7 reads "after commit: nothing yet".
- Reads follow `read_artifact_tree`: one `REPEATABLE READ READ ONLY` transaction, membership checked in the same snapshot, unknown and forbidden targets both `403`.
- `add-item-section-writes` adds the stage step to the guard order (stage row held while checking editability), `409 STAGE_NOT_STARTED`, and the text rules in `domain/text.py` that item titles and content use.
- `stage_instances` is a mutable table with `status`, `iteration_no` and `assigned_to`; `projects` carries `lifecycle_rev`, which this change must not touch.
- The route class verifies the caller before the body is read; a streaming response must authorize before its headers are sent, because a stream cannot change its status afterwards.
- psycopg 3 supports `LISTEN` on a dedicated async connection and `NOTIFY` inside a transaction, where the notification is sent only if that transaction commits.

## Goals / Non-Goals

**Goals:**

- A durable, ordered, authorized record of each conversation that the agent can rebuild its context from.
- One open conversation per stage, enforced by the database, with replacement in one transaction.
- Delivery after commit that cannot be lost and never delays or fails a command's response.
- Live hints to the UI that are safe to lose, because clients reconcile with reads.
- Lock order that cannot deadlock against the item write paths.

**Non-Goals:**

- Turns, assistant messages and the AI-service contract (`add-turns`).
- Operational hardening of delivery: metrics, dashboards, dead-letter handling (stream 5).
- Any fan-out transport beyond PostgreSQL `NOTIFY`; Redis is not needed at the planned load.

## Decisions

### 1. Data model

| Table | Kind | Columns and constraints |
| --- | --- | --- |
| `conversations` | mutable | `conversation_id` (UUID v7), `project_id`, `stage_instance_id`, `created_by`, `status` (`open`/`closed`), `close_reason` (`replaced`, later `promoted`), `replaced_by` (conversation id, nullable), `last_seq` (default 0), `opened_at`, `closed_at`. Partial unique index on `stage_instance_id` where `status = 'open'`. Index on (`stage_instance_id`, `opened_at` desc). |
| `messages` | append-only | `message_id` (UUID v7), `conversation_id`, `seq`, `role` (`user` in this change), `kind` (`text`), `text`, `actor_id`, `created_at`. Unique (`conversation_id`, `seq`). `reject_mutation` triggers and insert-only grants like the other append-only tables. |
| `obligation_deliveries` | mutable | `obligation_id` (primary key and foreign key to `ledger_obligations`), `status` (`queued`/`claimed`/`delivered`), `owner`, `generation` (default 0), `claim_expires_at`, `attempts` (default 0), `delivered_at`, `last_error`. Index on (`status`, `claim_expires_at`). |
| `stage_instances` | mutable, extended | `interview_status` (`pending`/`complete`, default `pending`), `interview_completed_by`, `interview_completed_at`, `interview_conversation_id` (nullable). |

Why interview columns on the stage instance rather than a table: it is one fact per stage instance, read with every stage read, and stream 3's manifest can copy it as it copies the rest of the stage row.

Why `obligation_deliveries` is separate from `ledger_obligations`: the obligation is evidence and stays immutable; delivery state changes. `CLAUDE.md`'s storage rules already list "delivery claims (owner, generation, expiry)" as a transactional-core record.

The `ledger_obligations` name predates hints. Renaming an append-only table needs a decision record and buys nothing now; the obligation record keeps its name and `event_kind` carries a namespace: `hint.*` for UI hints, `workspace.*` for ledger events.

### 2. Guard order and transaction boundaries

Decision 0002 gains two steps and is updated; same-kind rows are still locked in ascending id order.

1. Artifact rows `FOR UPDATE` (none in this change).
2. Authority rows (session, principal or membership) through `PlatformAuthority`.
3. Receipt lookup.
4. **Stage instance row `FOR UPDATE`** when the command changes the stage's conversation slot or its interview; `FOR SHARE` when it only needs the stage's status (as `add-item-section-writes` does for editability).
5. **Conversation row `FOR UPDATE`** when the command appends to or closes a conversation.
6. Counters, lineage root (unchanged, unused here).
7. Inserts and updates, receipt, obligations and their delivery records.

Lock order never reverses: `createConversation` takes the stage (step 4) then the old open conversation (step 5); `postMessage` takes only the conversation; `finishInterview` takes only the stage. Item writes take the artifact then the stage `FOR SHARE`, never a conversation. There is no cycle.

Write paths, each one short `READ COMMITTED` transaction opened by `run_command`:

- **`createConversation`.** Membership `FOR SHARE`; receipt lookup; stage instance `FOR UPDATE`; status must be `InProgress`, else `409 STAGE_NOT_STARTED` with the stage as `current`; select the open conversation of the stage `FOR UPDATE` if any and update it to `closed`, reason `replaced`, `replaced_by` the new id, `closed_at now()`; insert the new conversation; receipt with outcome references `{conversation_id, replaced_conversation_id}`; obligations `hint.conversation.closed` (if any) and `hint.conversation.opened`. A race between two opens on one stage is serialized by the stage lock; the partial unique index is the backstop, and a violation of it retries the whole transaction (`retry_on_unique`), which then sees and replaces the winner.
- **`postMessage`.** Membership `FOR SHARE`; receipt lookup; conversation `FOR UPDATE`; `created_by` must equal the caller, else `AccessDeniedError`; status must be `open`, else `409 CONVERSATION_CLOSED` with `current = {status, close_reason, open_conversation_id}`, where the open conversation is found by the stage's partial index in the same transaction; insert the message with `seq = last_seq + 1`; update `last_seq`; receipt with `{message_id, seq}`; obligation `hint.message.appended`. The conversation lock serializes `seq` allocation; the unique (`conversation_id`, `seq`) is the backstop.
- **`finishInterview`.** Membership `FOR SHARE`; receipt lookup; stage instance `FOR UPDATE`; status must be `InProgress`, else `409 STAGE_NOT_STARTED`; if `interview_status` is already `complete`, a no-op receipt with outcome `already_complete` and the existing completion as references; else update the four interview columns, with `interview_conversation_id` set from the stage's open conversation if any; receipt; obligation `hint.interview.finished`. `lifecycle_rev` is not read or written.

Fingerprints: `createConversation` `{stage_instance_id}`; `postMessage` `{conversation_id, kind, text}`; `finishInterview` `{stage_instance_id}`. No preconditions; appending to a chat is never a conflict with another reader's view.

Project scope for `/conversations/{id}` routes: a pre-read resolves the conversation's `project_id` before `run_command`, exactly as `add-item-section-writes` resolves an item's artifact; unknown or malformed ids end there with `403`, and the transaction re-reads the row under lock.

### 3. `CONVERSATION_CLOSED` is a new code

Posting to a closed conversation is a stale-state conflict: the client did not see the replacement. The closed set has no code that says so, `STAGE_CHANGED` would mislead, and `403` would hide the one fact the UI needs, the open conversation to switch to. `ConflictCode` gains `CONVERSATION_CLOSED`; `CLAUDE.md`'s list is updated, as the proposal calls out.

### 4. Reads

All reads use the tree-read pattern: one `REPEATABLE READ READ ONLY` transaction, membership in the same snapshot, fixed `403` for forbidden and unknown.

- `GET /projects/{id}/stages/{code}/conversations`: the stage (`stage_instance_id`, `stage_code`, `status`, `interview_status`), `open_conversation_id`, and conversations newest first.
- `GET /conversations/{id}`: the conversation plus its stage's identity, code, status and interview status.
- `GET /conversations/{id}/messages?after_seq=0&limit=200`: messages with `seq > after_seq` in order, `limit` capped at 500, and `has_more`. The snapshot guarantees a gapless prefix.

### 5. Delivery after commit

`storage/receipts.insert_obligations` also inserts one `obligation_deliveries` row per obligation with status `queued`, in the command's transaction.

`delivery/deliverer.py`:

- `claim(tx, obligation_id, owner, ttl)`: `UPDATE obligation_deliveries SET status='claimed', owner=:owner, generation=generation+1, claim_expires_at=now()+:ttl, attempts=attempts+1 WHERE obligation_id=:id AND (status='queued' OR (status='claimed' AND claim_expires_at < now())) RETURNING generation`. No row means someone else holds a live claim.
- `deliver(obligation, consumer)`: the consumer runs outside any Workspace transaction unless it is the hint consumer, which needs the mark transaction (Decision 6).
- `mark_delivered(tx, obligation_id, owner, generation)`: compare-and-set on `owner` and `generation`; no row means the claim expired and was taken over, and the late deliverer logs and stops.
- `release(tx, obligation_id, owner, generation, error)`: back to `queued` with `last_error`.

Consumers are registered by kind prefix in a registry on the app: `hint.` to the `NOTIFY` publisher. A kind with no consumer is skipped: the record stays `queued` with zero attempts.

Trigger: `run_command` returns the receipt id in `CommandResult`; the API layer schedules `deliver_for_receipt(receipt_id)` as a tracked background task on the app after the response is built. The response never waits for delivery. Tests `await app.state.deliveries.drain()` to observe it.

Reconciler: `delivery/reconciler.py` with `reconcile_once(engine, owner, now)` selecting records that are `queued` or expired-claimed, oldest first, limited per pass, each with backoff `min(2 ** attempts, 60)` seconds since its last attempt; the lifespan starts a loop every `PRODUCT_RECONCILE_INTERVAL_SECONDS` (default 5; tests set 0 to disable the loop and call `reconcile_once`). Claim ttl is 30 seconds.

Lag: `delivery/lag.py` counts `queued` and expired-claimed records per kind with the oldest age; `make lag` prints it. An HTTP surface for operations waits for stream 5.

Alternatives considered: deliver inline before responding (rejected: a slow consumer would slow every write, and `add-turns` adds an HTTP consumer); a transactional outbox table per kind (rejected: one record type with a kind namespace is the doc's model and what stream 5 extends); PostgreSQL `SKIP LOCKED` queues (not needed at one process; the claim columns give the same guarantee across processes and survive crashes).

### 6. Hints over `NOTIFY`, atomically with the delivered mark

The hint consumer opens one short transaction: `mark_delivered` then `NOTIFY product_hints, :payload`, then commit. Both happen or neither, so a hint is published exactly when its record says delivered. The payload is the hint JSON (event id, project id, operation id, kind, entity ids, seq or status), well under the 8000-byte limit.

`delivery/hint_bus.py` keeps one `LISTEN` connection per process and a set of subscriber queues keyed by project. On connection loss it reconnects and pushes a `resync` event to every subscriber, since notifications during the gap are lost. Each subscriber queue is bounded; on overflow the subscriber gets `resync` instead of the dropped hints. `resync` tells the client to refetch, which is what it does on reconnect anyway.

Hint kinds: `conversation.opened` (conversation, stage instance), `conversation.closed` (conversation, stage instance, reason, replaced_by), `message.appended` (conversation, seq), `interview.finished` (stage instance). The SSE `event:` name is the kind without the `hint.` prefix; `id:` is the event id; `data:` is the JSON.

### 7. The event stream endpoint

`GET /projects/{id}/events` is a `StreamingResponse` with `text/event-stream`. Authorization runs before the response starts: session at the route boundary, membership in a snapshot read; a failure is the ordinary `403` before any header is sent. The generator then sends `ready`, subscribes to the bus for the project, and loops: before forwarding a hint it rechecks session and membership with one short snapshot read; while idle it sends a heartbeat comment every 15 seconds and rechecks every 60 seconds; a failed recheck ends the stream. `Last-Event-ID` is read and ignored, by design (doc s16: no invented event history).

Membership recheck uses `PlatformAuthority.require_member(for_share=False)`; after `use-platform-identity`, the session check is `platform.session_valid`. Before it, the session is the JWT's expiry, so the idle recheck covers expiry.

### 8. Message text rule

`domain/text.py` gains `is_message_text(value)`: valid Unicode, no NUL, no C0 or C1 control other than LF, CR and TAB, length 1 to 20,000, at least one non-whitespace character. The Pydantic body uses a pattern plus a validator, so `422` happens before any lookup, as item content does.

### 9. Test strategy

- Command tests run `run_command` directly with two real connections for the races: two opens on one stage, two posts on one conversation, a replacement committing before an in-flight post's check.
- Delivery tests call `deliver_for_receipt` and `reconcile_once` directly, with a recording consumer; the crash case runs a command with the background trigger disabled, asserts the record is `queued`, then reconciles.
- Event stream tests use httpx streaming against the ASGI app with a real database: subscribe, post, assert `ready` then the hint; revoke membership and assert the stream ends within the bounded interval (test settings shrink the intervals).
- Immutability tests extend the existing ones to `messages`.

### 10. Ambiguities and divergences from the design doc

- Ownership of conversations and the stream fan-out diverge from s16, as the proposal records; decision 0010 holds the rationale.
- s16 asks hints to carry entity/version discriminators for multi-item batches; conversation hints have one entity each, so the rule is trivially met. Item hints come with stream 2.
- The doc gives no status for a write to a closed thread (s13 only says promotion makes threads read-only); `CONVERSATION_CLOSED` is the recorded choice and stream 3 reuses it with reason `promoted`.
- The interview is a product addition with no doc section; its fact lives on the stage instance so that stream 3's manifest captures it.

### 11. Decision records

- `docs/decisions/0010-conversations-in-workspace.md` (new): ownership, one open conversation per stage with replacement, single writer, `CONVERSATION_CLOSED`, interview on the stage instance.
- `docs/decisions/0011-obligation-delivery.md` (new): delivery records, claim semantics, background trigger plus reconciler, `NOTIFY` atomic with the mark, `resync` on loss, and what stream 5 adds.
- `docs/decisions/0002-guard-order.md`: steps 4 and 5 above.

## Risks / Trade-offs

- [Stage `FOR UPDATE` contends with item writes' `FOR SHARE`] → Conversation opens and interview finishes are rare and short; item writes wait milliseconds.
- [`NOTIFY` is lost when no process listens or during a `LISTEN` reconnect] → The record is still `delivered`, which is correct: hints are hints; `resync` on reconnect and the client's periodic reconcile cover the gap.
- [A claim stuck for its ttl after a crash] → Thirty seconds of delay at worst before the reconciler re-drives; the client's catch-up reads do not depend on it.
- [Background delivery task lost on process shutdown] → Tasks are tracked and awaited at shutdown with a timeout; anything unfinished is re-driven by the reconciler of the next process.
- [Subscriber overflow under bursts] → Bounded queues and `resync` keep memory flat; the client refetches once.
- [Reusing `ledger_obligations` for hints] → Naming only; the kind namespace keeps consumers apart, and a rename can come with stream 5 under its own decision record.

## Migration Plan

1. Apply after `add-item-section-writes` is archived; `use-platform-identity` may land before or after.
2. One Alembic revision (next free number) adds the three tables and the four columns; downgrade drops them. Existing stage instances get `interview_status = 'pending'` by default.
3. Deploy: the reconciler starts with the app; no data backfill.
4. Rollback: downgrade the revision and remove the routers; receipts for the new commands remain as history.

## Open Questions

- Heartbeat (15 s), idle recheck (60 s), claim ttl (30 s) and page limit (200, max 500) are starting values; load testing in stream 5 may change them without touching the specs.
- Whether hints should carry the acting `actor_id` for a "who is typing" affordance; left out until UX asks, since it changes no behaviour.
- The lag surface stays a `make` target until stream 5 decides the operations API.
