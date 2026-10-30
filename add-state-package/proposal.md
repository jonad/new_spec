# Proposal: add-state-package

## Why

On 2026-10-30 the teams agreed that Workspace owns the context assembler: Workspace supplies the facts a generation runs on, the AI service supplies the instructions.
Today those facts are spread over the artifact tree read, the section list, the stage reads and the message history, and nothing fixes them as one snapshot, renders them the same way twice, or lets anyone say afterwards what the agent saw.
This change adds the state package: the design doc's `getStatePackage` (s11), extended with the conversation transcript because conversations now live in Workspace, assembled as one canonical, hashed document from one database snapshot.
It is assembled per turn and never stored per conversation, because Workspace state moves between turns (hand edits, applied proposals, the interview finishing, membership changes) and the doc freezes a run's inputs at worker start.

**Delivery stream:** Conversations (stream 7), the second of three sequential changes: `add-conversations`, `add-state-package`, `add-turns`.
It is applied after `add-conversations` is archived, because the package carries the stage's interview status and a conversation's transcript, and through it after `add-item-section-writes`.
It does not depend on `use-platform-identity`.

**Design doc sections implemented** (Product Workspace Design, Phase 1 v0.5 Review Edition):

- s4 Content records and immutable evidence (the package carries current `ItemVersion`s by exact identity and revision)
- s10 AI handoff, reservation and crashes ("AI freezes the actual delivered inputs at worker start": the package is those inputs on the Workspace side)
- s11 State packages, dependencies and audience (`getStatePackage` supplies current items, section bindings, exact versions, membership tokens and the intended audience)
- s14 Promotion and consistent reads (the two-query snapshot tree; the package reuses it)
- s21 Delivery sequence and decisions, shared stream 2 "Context, contracts and agent quality" (state packages)
- C20 "Can we reproduce exactly what the AI did?" (the actual context of a run becomes a Workspace fact with a stable hash)

## What Changes

- **State package read.** `GET /projects/{id}/stages/{code}/state-package` returns the stage's state package to a current Maker of the project, from one `REPEATABLE READ` snapshot, as the tree read does. An optional `conversation_id` names one of the stage's conversations, whose complete transcript is then included. Reads do not depend on stage status and never mutate.
- **Package content, schema version 1.** The project (identity, name, lifecycle state, current stage, `lifecycle_rev`); the stage (instance identity, code, status, iteration, interview status); every artifact of the stage with `content_rev` and `membership_rev`, its sections (identity, title, source, `ref_prefix`) and their active items with the current `ItemVersion` (identity, revision, title, content), all in creation order; the intended audience (project scope); and the conversation (identity, status, close reason, `last_seq`, every message with `seq`, role, kind and text) or null. Retired items are absent, empty sections are present, nothing is windowed or summarized.
- **Canonical rendering and hash.** The response carries the package and the sha256 of its canonical rendering: UTF-8 JSON with sorted keys, no insignificant whitespace, integers only and strings exactly as stored. The same state renders to the same bytes in any process; any change of content, membership or revision changes the hash. `add-turns` records that rendering as evidence per turn and `add-proposal-apply-gate` can pin a proposal to its hash.
- **Facts only, with fixed provenance.** The package holds no timestamps, no actor identities and no instructions. Every string a user wrote or a model produced appears verbatim in the one position the schema fixes for it (item title and content, message text), so the AI service can fence it as untrusted data by structure rather than by scanning.
- **Golden tests.** A fixture project seeded with fixed identifiers renders to a recorded canonical document and hash under `tests/fixtures/state_packages/`; a content or schema change shows up as a diff of that file in review, as `CLAUDE.md` requires for generated documents.
- **Decision record:** `docs/decisions/0012-state-package.md` (per-turn assembly, content-only package, schema version 1 and its extension path, structural provenance).
- **`CLAUDE.md` and README:** the stream 7 line lists three changes, the testing expectations name state packages among the golden-tested documents, and the README documents the endpoint.
- No new tables, migrations, commands, receipts, obligations or `409` codes: this change is a read.

**Extensions and deviations from the design doc** (each recorded in decision 0012, not silently chosen):

- s11 says "AI adds history, skills and Knowledge inputs to the complete snapshot". History is now a Workspace fact (decision 0010), so the transcript is part of the package; skills and Knowledge inputs stay on the AI side.
- s11 lists labeled historical manifests, assessments and eligible tentative proposals among the package's contents. Those records do not exist before streams 3, 5 and 2; schema version 1 omits them and the version increments when each lands.
- The stage's pinned configuration (section templates, interview question set, agent configuration) is Platform-owned configuration that Workspace does not hold yet; the package carries the sections as they exist, which are the materialized template, and the configuration pin joins the package under a later schema version.
- Per-turn assembly rather than a package stored at conversation open is a product decision taken on 2026-10-30; the doc's wording (inputs frozen at worker start) supports it.

## Capabilities

### New Capabilities

- `state-package`: the assembled, canonical, hashed set of Workspace facts for a stage and optionally one of its conversations, read from one snapshot by current Makers.

### Modified Capabilities

None.
Authorization for the read follows the existing read rule (current membership checked in the same snapshot, one fixed `403` for unknown and forbidden), stated inside the new capability as the section listing does.

## Impact

- **Code:** `domain/state_package.py` (package model, canonical rendering, hash), `storage/state_package_queries.py` (stage, artifacts, conversation and transcript queries in one snapshot, reusing the tree queries), `commands/assemble_state_package.py` (the read), `api/state_packages.py` with its response models, OpenAPI regenerated.
- **Tests:** command tests with two real connections for the snapshot guarantee, API tests for authorization and the fixed `403`, golden tests with fixed-identifier seeds, determinism tests across processes.
- **Documentation:** decision 0012, README endpoint section, `CLAUDE.md` delivery order and testing expectations.
- **Consumers:** `add-turns` (the claim assembles a package and records it as evidence), the AI service through `add-turns` (reads it under a live turn claim), `add-adk-generation` (generates from it), the UI (shows what the agent will see), `add-proposal-apply-gate` (a proposal pins the package hash it was generated from).
- **No** schema migration, no new error code, no new command.

**Out of scope** (deferred, with where it goes):

- Persisting a package as an evidence object with key, version id and sha256: `add-turns`, with the first row that references one.
- Reads by the AI service: `add-turns`, authorized by a live turn claim.
- The stage configuration pin (templates, question set, agent configuration): a later change, bumping the schema version.
- Manifests, assessments and tentative proposals in the package: streams 3, 5 and 2.
- Windowing, summarization or paging of the transcript: out of Phase 1; a package is one snapshot.
- Skills, prompts and the fencing itself: the AI service.
- Comparing two packages or sending deltas: the AI service, using the hash and the per-artifact revisions the package carries.
