# Proposal: add-conversations

## Why

The 2026-10-29 integration meeting moved the chat out of the AI module: the UI stops calling agents directly, the Workspace team owns conversation and message storage, and the agent becomes stateless over what Workspace holds.
Nothing in Workspace can record a conversation today, so the end-to-end chat flow has no durable handler to start from.
This change builds that handler's durable half: conversations bound to a stage, append-only messages, the stage-level interview fact, and the first delivery of named obligations after commit, which carries live hints to the UI.
The agent-facing half, turns and the AI-service contract, follows in the next change of the same stream so each one stays reviewable.

**Delivery stream:** Conversations, a new stream that `CLAUDE.md` does not list yet. This change adds it as stream 7: it runs after stream 1 and in parallel with stream 2, and stream 6 (`add-adk-generation`) requires it. It is the first of three sequential changes: `add-conversations`, then `add-state-package` (the context assembler, decided 2026-10-30), then `add-turns`.
It is applied after `add-item-section-writes` is archived, because it reuses `409 STAGE_NOT_STARTED`, the stage step of the guard order and the text rules from that change.
It does not depend on `use-platform-identity`; whichever of the two is applied second adapts the other's test helpers, and `use-platform-identity` already lists that task.

**Design doc sections implemented** (Product Workspace Design, Phase 1 v0.5 Review Edition):

- s2 Command boundary and authority (every conversation write is a receipted, authorized command)
- s5 Command receipts and Risk records and s7 Retry-safe commands and acknowledgements (receipts, no-op receipts, named obligations committed with the mutation)
- s9 Proposals, groups and amendment ("Conversational agents propose changes": the conversation is where that dialogue is recorded)
- s13 Initialization and completion ("Promotion makes source conversational threads read-only": the conversation close path that stream 3 will call)
- s16 Audit delivery and live UI recovery (one delivery adapter and reconciler protocol with owner, generation and expiry claims; hints that never replace state; reconnect and periodic reconciliation; the commit-then-publisher-crash case)
- s17 External API contract (additions to the frozen contract) and s18 Internal owner interfaces and transactions ("keep broker publication outside transactions")
- s21 Delivery sequence and decisions ("Conversation visibility/action matrix" is a Product + UX decision; the choices below are recorded for it)
- C08 (one outstanding turn serializes generation within a conversation; prepared here, enforced in `add-turns`)

## What Changes

- **Conversations.** A Maker opens a conversation on a stage that is `InProgress`. A stage has at most one open conversation; opening a new one closes the current one with reason `replaced`, in the same transaction. Closed conversations keep every message and stay readable. A conversation has a single writer, its creator; every Maker of the project can read it. Conversations of a stage are listed newest first with the open one marked.
- **Messages.** Append-only, with a per-conversation `seq` that starts at 1 and never has gaps. In this change only the writer posts, and only `text` messages (Markdown, 1 to 20,000 characters, same Unicode rules as item content). Posting to a closed conversation is refused with `409 CONVERSATION_CLOSED`, carrying the conversation's status and the stage's open conversation, if any, as `current`. History is read after a `seq`, which is also the catch-up cursor for the event stream.
- **Stage interview.** Each stage instance records whether its interview is `pending` or `complete`. `finishInterview` marks it complete with a receipt, references the stage's open conversation if there is one, leaves `lifecycle_rev` alone, and is a recorded no-op when repeated. It is refused on a stage that has not started with `409 STAGE_NOT_STARTED`. Reads of a conversation and of a stage's conversation list carry the status.
- **Obligation delivery.** Named obligations keep being committed with the receipt; this change delivers them. Each obligation gets a mutable delivery record with status, owner, generation, claim expiry and attempts. A deliverer runs after commit, claims the record, hands the obligation to the consumer registered for its kind, and marks it delivered. A reconciler re-drives queued and expired claims with bounded backoff. Kinds without a consumer stay queued and visible. Delivery never changes a command's response.
- **UI hints and the event stream.** Conversation, message and interview commands name `hint.*` obligations. Their consumer publishes each hint through PostgreSQL `NOTIFY`, atomically with the delivered mark. `GET /projects/{id}/events` serves them as server-sent events to current Makers of the project. A hint carries identities, `seq` and status only, never content. The subscription is authorized when it opens and again on every delivery and at a bounded interval, and ends when the session or membership is no longer valid. There is no replay: a client catches up with the message and tree reads.
- **Spec change to the closed error set** (called out as `CLAUDE.md` requires; its list is updated): `409 CONVERSATION_CLOSED`, a write to a conversation that is closed.
- **Decision records:** `docs/decisions/0010-conversations-in-workspace.md` (ownership, one open conversation per stage, single writer) and `0011-obligation-delivery.md` (delivery records, deliverer, reconciler, `NOTIFY` hints); `0002-guard-order.md` gains the stage-instance and conversation lock steps.
- **`CLAUDE.md`:** stream 7 in the delivery order, the new error code, and a storage rule for obligation delivery records.

**Deliberate deviations from the design doc** (each recorded, not silently chosen):

- The doc places conversation state on the AI side and gives AI/UX the stream fan-out (s16). Here Workspace owns conversations, messages and the event stream, as agreed with the AI and Platform teams on 2026-10-29. Recorded in decision 0010.
- A single writer per conversation: multi-user threads are out of Phase 1 (`CLAUDE.md`). Other Makers read; a Maker who wants to chat opens a new conversation, which closes the current one.
- The stage interview does not exist in the doc. It is a product addition: a question set per stage whose answers fill the sections. This change records only the fact; the questions stay in stage configuration, outside this change.
- Delivery after commit was planned for stream 5 (`add-risk-and-ledger-delivery`). The mechanism lands here because hints need it now; stream 5 keeps the ledger and Risk consumers and the reconciler hardening.

## Capabilities

### New Capabilities

- `conversations`: opening a conversation on a stage, the one-open-per-stage rule and replacement, single writer, listing and reading, and what closing preserves.
- `messages`: the append-only message record, `seq` allocation, who may post, text rules, `CONVERSATION_CLOSED`, and reading history after a `seq`.
- `stage-interview`: the interview status of a stage instance, `finishInterview` and its no-op repeat, and where the status is read.
- `obligation-delivery`: delivery of named obligations after commit through claims, the reconciler, consumer registration by kind, visibility of lag, and isolation from command responses.
- `event-stream`: the hint contract, the server-sent events endpoint, its authorization at open and on delivery, catch-up by reads, and the commit-without-publish recovery.

### Modified Capabilities

- `actor-authorization`: conversation writes are accepted only from the conversation's writer; other Makers and system callers receive `403`.
- `immutable-records`: messages join the append-only record types.

## Impact

- New Alembic revision: `conversations`, `messages` (append-only, with the `reject_mutation` triggers and insert-only grants), `obligation_deliveries`, and interview columns on `stage_instances`.
- New HTTP API: `POST` and `GET /projects/{id}/stages/{code}/conversations`, `GET /conversations/{id}`, `GET` and `POST /conversations/{id}/messages`, `POST /projects/{id}/stages/{code}/finish-interview`, `GET /projects/{id}/events` (server-sent events).
- New modules under `commands`, `domain`, `storage`, `api` and the first real content of `delivery/`: the deliverer, the reconciler, the `NOTIFY` publisher and the consumer registry. The command pipeline gains a post-commit hook that triggers delivery for the receipt just committed.
- The app lifespan starts the reconciler task; `PRODUCT_RECONCILE_INTERVAL_SECONDS` and `PRODUCT_HINT_CHANNEL` join the settings.
- The closed `409` set grows by `CONVERSATION_CLOSED`.
- `docs/decisions/0002`, new `0010` and `0011`, `CLAUDE.md` and the README API walkthrough.
- No new runtime dependencies: server-sent events are a streaming response in FastAPI, and `NOTIFY`/`LISTEN` come with psycopg.

## Out of Scope

- Turns, the AI-service claim contract, assistant messages, `choice` and `question` message kinds, cancellation of a running turn on close, and the `turn.run` obligation: `add-turns`, the next change of this stream.
- The stage configuration that holds the interview question set and section templates.
- Closing a conversation on promotion (stream 3 calls the close path with reason `promoted`) and an explicit close command, which the flow does not need.
- Ledger and Risk consumers, delivery metrics and the reconciler's operational hardening (stream 5).
- Audience rules for conversations (doc s11): every Maker of the project reads every conversation in Phase 1.
- Attachments and images in messages, summarization and multi-user threads.
