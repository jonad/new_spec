# Design: add-state-package

## Context

See proposal.md for motivation.
What shapes the approach:

- `read_artifact_tree` is the read pattern: one `REPEATABLE READ READ ONLY` transaction holds the membership check (`require_member(for_share=False)`) and both content queries; unknown and forbidden both raise `AccessDeniedError`, which the API turns into the one fixed `403`; identifiers are parsed before any query. `domain/tree.merge_tree` orders sections and items by their UUID v7, which is creation order, and refuses a pointer whose version the snapshot did not return.
- `domain/fingerprint.canonical_json` (decision 0004) already defines the canonical form: sorted keys, compact separators, UTF-8, integers only, strings exactly as given.
- `add-conversations` adds `conversations` (`status`, `close_reason`, `last_seq`), append-only `messages` (`seq`, `role`, `kind`, `text`) and `interview_status` on `stage_instances`, and addresses stages as `/projects/{id}/stages/{code}`.
- Caller-supplied identifiers may be any RFC 9562 UUID, so test seeds can fix identifiers and make a rendering reproducible byte for byte.
- `add-turns` will call the same assembler at claim time and store the canonical rendering as an evidence object; the AI service will read it under a live claim. Nothing in this change stores a package.

## Goals / Non-Goals

**Goals:**

- One read, one snapshot, one rendering: the package a Maker sees is the package the agent will get.
- Deterministic bytes, so hashes mean something across processes, deployments and time.
- A document shape `add-turns` can store unchanged as evidence and `add-proposal-apply-gate` can pin to.
- No new write path, lock, table or error code.

**Non-Goals:**

- Storing packages, serving them to the AI service, the stage configuration pin, transcript windowing, and diffing packages (see proposal, Out of scope).

## Decisions

### 1. Assembled per turn, never stored per conversation

The assembler is a pure function of a stage, an optional conversation and a snapshot.
Nothing is written at conversation open, and nothing is cached between turns.

Why: Workspace state moves between turns (hand edits, applied proposals, the interview finishing, membership changes), so a package taken at conversation open would pin later proposals to stale revisions and hide the interview flip; the doc freezes a run's inputs at worker start (s10), which is per turn; C20 wants the actual context of each run; and the parallel section sub-agents of one turn must fan out from one snapshot.

Alternatives considered: assemble at conversation open (rejected: stale pins turn every later apply into `ITEM_CHANGED`); a stored package updated by deltas from hints (rejected: hints never replace state, for the agent as much as for the UI); a cached package invalidated by revisions (rejected: a cache is a second source of truth for the one document that must be exact).

Durable record: `docs/decisions/0012-state-package.md`.

### 2. Package model and schema version 1

`domain/state_package.py` holds frozen dataclasses:

- `StatePackage(schema_version=1, project, stage, artifacts, audience, conversation)`;
- `ProjectFacts(project_id, name, lifecycle_state, current_stage, lifecycle_rev)`;
- `StageFacts(stage_instance_id, stage_code, status, iteration_no, interview_status)`;
- `artifacts`: a tuple of the existing `ArtifactTree` nodes, one per artifact of the stage, ordered by `artifact_id`;
- `Audience(scope="project")`;
- `ConversationFacts(conversation_id, status, close_reason, last_seq, messages)` or `None`, with `MessageFacts(seq, role, kind, text)` in `seq` order.

`to_document(package)` is the one place the JSON shape is defined; the API response model mirrors it and a test asserts they agree, as the tables-versus-migrations test does for the schema.

Field policy: no timestamps and no actor identities anywhere in the package.
Why: determinism (two assemblies of equal content are equal), audience minimization (the package travels to the AI service and becomes evidence, and generation needs neither who nor when), and the single writer of a conversation is already implied by the `user` role.

Omitted doc contents and the bump rule: labeled historical manifests (stream 3), assessments (stream 5), eligible tentative proposals (stream 2) and the stage configuration pin (a later change) each add fields and increment `schema_version`; consumers check the version before reading fields.

### 3. Canonical rendering and the response envelope

`render_canonical(package)` is `canonical_json(to_document(package))`; `package_sha256` is the hex sha256 of those bytes.
One canonical form in the codebase: the fingerprint's.

The response is `{"package": <document>, "sha256": "<hex>"}`.
`schema_version` lives inside the package and is hashed; the envelope has no version of its own.
Why an envelope rather than the raw canonical bytes with a hash header: it is self-describing, the client verifies by re-rendering, a header would need exposure through the gateway's fixed CORS allowlist, and `add-turns` stores `render_canonical` output as the evidence object, so the stored bytes and the served document agree by construction.
The HTTP serializer may emit different whitespace or key order from the canonical bytes; equality is defined on the canonical rendering, which the client recomputes, and the API test does exactly that.

### 4. The read: one snapshot, fixed query order

`commands/assemble_state_package.py` exposes `assemble_state_package(uow, authority, principal, project_id, stage_code, conversation_id=None) -> StatePackage`.

1. Parse `project_id` and, when given, `conversation_id`; a malformed identifier raises `AccessDeniedError` before any query.
2. One `REPEATABLE READ READ ONLY` transaction through `uow.run`:
   1. `authority.require_member(tx, project_id, principal, for_share=False)`, first, because the route is project-scoped and nothing about the project is revealed before membership holds (the section listing does the same);
   2. the project row;
   3. the stage by `(project_id, stage_code)`; none raises `AccessDeniedError`;
   4. the stage's artifacts ordered by `artifact_id`;
   5. `tree_queries.sections_and_pointers` and `tree_queries.versions`, extended to take a collection of artifact ids so the stage's artifacts need one query each, then `merge_tree` per artifact;
   6. when requested, the conversation by id where `stage_instance_id` matches the stage; none raises `AccessDeniedError`; then its messages ordered by `seq`.

No locks and no writes.
The snapshot invariants extend: every pointer finds its version (`InconsistentSnapshotError` otherwise) and the conversation's `last_seq` equals the highest `seq` read, asserted the same way.

Why the conversation is cross-checked against the stage in the path: a package is a stage's facts; a transcript from another stage would be a different package, and the mismatch is treated as an unknown target, not a validation error, so the fixed `403` reveals nothing.

### 5. Route and response models

`api/state_packages.py`: a router with `WorkspaceRoute`, `GET /projects/{project_id}/stages/{stage_code}/state-package` with optional query `conversation_id`, `ERROR_RESPONSES` as every read.
`StatePackageResponse(package: StatePackageDocument, sha256: str)`, where `StatePackageDocument` and its nested models mirror `to_document` with `extra="forbid"` and are built from the document, never from the dataclasses directly, so the mirror test is the single guard against drift.
OpenAPI is regenerated.

Why the stage code is in the path and the conversation is a query parameter: stages are addressed by code within a project in the conversation reads, one read serves both shapes, and the path's stage cross-checks the conversation.

### 6. Golden tests with fixed identifiers

The seed helpers accept optional identifiers (generated by default); the golden project uses constants whose sort order is the intended creation order.
The fixture is `tests/fixtures/state_packages/ideation.json`, the canonical bytes exactly as rendered, with `ideation.sha256` beside it.
The test renders the golden project with its conversation, compares bytes and hash, and on failure shows a structural diff.
`PRODUCT_UPDATE_GOLDEN=1` rewrites the fixture; the diff is reviewed as a content change.

Why fixed identifiers rather than placeholder normalization: the recorded file is the real rendering, byte for byte, and the same bytes `add-turns` will store as evidence; a normalized rendering would be a second format with its own hash.

### 7. Provenance by structure

No per-field tags.
The schema fixes where human and model text lives: the project's `name`, a section's `title`, an item's `title` and `content`, a message's `text`; every other field is an identifier, a revision, a count or a value from a closed set.
A unit test walks the document and asserts every string value is in one of those positions, parses as a UUID, or belongs to the known enumerations.
Why: tags inside the content would themselves be data the model sees; the AI service fences by path, which is reliable because the schema is versioned.

### 8. Size and performance

Eight indexed queries per package: membership, project, stage, artifacts, sections and pointers, versions, conversation, messages.
No paging by design: a package is one snapshot.
The transcript is bounded by Phase 1 use (one open conversation per stage, messages of at most 20,000 characters); `add-turns` measures package sizes in its tests, and a cap would be a product decision under a new schema version.

### 9. Test strategy

- Domain: document shape and ordering, canonical bytes, hash, the field walk, and the document-versus-response-model mirror.
- Storage: the new queries against a real database with seeds: several artifacts, empty sections, retired items, a conversation of another stage, messages in order.
- Command: two real connections for the snapshot scenarios, following `test_read_artifact_tree_snapshot.py`; authorization paths all raise `AccessDeniedError`; no mutation checked with row counts, revisions and `last_seq`.
- API: shape, client-side hash verification, byte-identical `403`s, two app instances on two engines returning identical bytes.
- Golden: Decision 6.

### 10. Ambiguities and divergences from the design doc

- s11 has the AI call `getStatePackage`; here Makers read it, and the AI service reads it in `add-turns` under a claim. Same facts, two readers.
- s11 "membership tokens": the per-artifact `membership_rev` is the token; no separate token type is introduced.
- s11 "trusted intended audience, policy identity": the MVP has one audience, the project's members, recorded as `audience.scope = "project"`; a policy identity waits for Platform to define one.
- s11 contents that do not exist yet (manifests, assessments, tentative proposals) are omitted under schema version 1, with the bump rule in Decision 2.
- The doc has no transcript in the package; it is included because decision 0010 moved conversation history into Workspace.
- The interview is a product addition (`add-conversations`); its status is a stage fact and belongs in the package.

### 11. Decision records

- `docs/decisions/0012-state-package.md` (new): per-turn assembly, the content-only field policy, schema version 1 and the bump rule, canonical rendering and envelope, provenance by structure, fixed-identifier golden tests, and the divergences above.

## Risks / Trade-offs

- [Long transcripts make large packages] → No paging by design; sizes are measured in `add-turns`; a cap is a product decision under a new schema version.
- [Document and response model drift] → One source (`to_document`) and a mirror test that validates the document into the model and dumps it back unchanged.
- [Golden fixture churn on every content change] → One fixture, regenerated with a flag, reviewed as a content change; schema changes bump the version with the diff.
- [Extending `tree_queries` touches `getTree`] → The single-artifact functions delegate to the collection variants; the existing tree tests run unchanged.
- [Clients compare raw response bytes instead of canonical bytes] → The README and the API test show verification against the canonical rendering.

## Migration Plan

1. Apply after `add-conversations` is archived. No schema change.
2. Deploy: one new route. Rollback: remove the router; nothing references packages until `add-turns`.

## Open Questions

- Whether the UI will call the endpoint; if it does not, the endpoint still serves tests, operations and the `add-turns` walkthrough.
- Whether to expose the hash as an `ETag` for conditional reads; an addition that changes no behaviour here.
- The shape of the stage configuration pin, decided by the change that introduces it.
