# Tasks

Work top to bottom; `make verify` must pass before a task is checked.
Every group lands its own tests and documentation.
Apply after `add-conversations` is archived.

## 1. Package model and canonical rendering

- [ ] 1.1 Add `domain/state_package.py`: the frozen dataclasses from design Decision 2, `to_document`, `render_canonical` (through `canonical_json`) and `package_sha256`. Verify with `tests/domain/test_state_package.py`: `schema_version` 1 and the exact key set at every level, artifacts, sections, items and messages in order, canonical bytes with sorted keys and no insignificant whitespace, the hash equal to the sha256 of those bytes, and two packages built from equal facts rendering to identical bytes; `make verify` passes.
- [ ] 1.2 Add the field-policy test to the same file: walk a rendered document and assert every string value is the project's `name`, a section's `title`, an item's `title` or `content`, a message's `text`, a UUID, or a member of the closed enumerations, and that no key is a timestamp or an actor identity. Verify it fails when a `created_at` is smuggled into the document in the test and passes otherwise.
- [ ] 1.3 Write `docs/decisions/0012-state-package.md` covering per-turn assembly, the content-only field policy, schema version 1 and the bump rule, the canonical rendering and envelope, provenance by structure, fixed-identifier golden tests, and the divergences in design Decision 10. Verify it names the alternatives rejected in design Decision 1.

## 2. Snapshot queries

- [ ] 2.1 Add `storage/state_package_queries.py` with `project_facts`, `stage_by_code` and `artifacts_of_stage`, and extend `tree_queries.sections_and_pointers` and `tree_queries.versions` to take a collection of artifact ids with the single-artifact callers delegating. Verify with `tests/storage/test_state_package_queries.py` against a real database: two artifacts in one stage, an empty section, a retired item absent from the pointers, an unknown stage code returning nothing; the existing tree tests pass unchanged.
- [ ] 2.2 Add `conversation_of_stage` and `messages_in_order` for the tables `add-conversations` created. Verify with tests: a conversation of another stage returns nothing, a closed conversation is returned with its reason, messages come back in `seq` order with `role`, `kind` and `text`.

## 3. The read

- [ ] 3.1 Add `commands/assemble_state_package.py` per design Decision 4. Verify with `tests/commands/test_assemble_state_package.py`: a freshly initialized stage, sections and items, a retired item hidden, an empty section listed, interview status `complete` after `finishInterview`, a `NotStarted` stage readable, an open and a closed conversation, a null conversation when none is named; unknown project, unknown stage code, conversation of another stage, malformed identifiers, non-member and revoked member all raising `AccessDeniedError`; and no mutation (row counts, `content_rev`, `membership_rev`, `lifecycle_rev`, `interview_status` and `last_seq` unchanged).
- [ ] 3.2 Add `tests/commands/test_assemble_state_package_snapshot.py` with two real connections: an item write committing mid-assembly is never half-visible, and a message appended mid-assembly leaves the transcript a consistent prefix with a matching `last_seq`. Verify both pass under `make verify`.
- [ ] 3.3 Add the determinism tests: two assemblies with no write between them render identically; an item update changes the hash; two stages with the same content written by different actors at different times render to the same hash. Verify they pass.

## 4. API

- [ ] 4.1 Add `api/state_packages.py` with the route from design Decision 5, the response models mirroring `to_document` with `extra="forbid"`, router registration and regenerated OpenAPI. Verify with `tests/api/test_state_packages.py`: `200` with the documented shape, the `conversation_id` query including the transcript, and the client recomputing the sha256 from the canonical rendering of `package` and matching `sha256`; the OpenAPI document lists the route.
- [ ] 4.2 Add the mirror test: `to_document` output validates into the response model and dumps back unchanged, and an unknown key is rejected. Verify it passes.
- [ ] 4.3 Add the authorization tests: non-member, unknown project, unknown stage code, malformed project id, malformed and foreign `conversation_id`, and a revoked member all get `403` byte-identical to the existing fixed body, and no receipt is written by a read. Verify they pass.
- [ ] 4.4 Add the cross-process test: two application instances on two engines return byte-identical packages and the same `sha256` for one stage. Verify it passes.

## 5. Golden tests

- [ ] 5.1 Let the seed helpers accept fixed identifiers with generated ones as the default: `seed_artifact`, `seed_section`, `seed_item`, `seed_revision` and the conversation and message seeds `add-conversations` added. Verify the existing tests pass unchanged.
- [ ] 5.2 Add `tests/support/golden.py` building the golden project with constant identifiers in creation order: one stage with two sections, three items of which one retired, the interview complete, and one open conversation with three messages. Record `tests/fixtures/state_packages/ideation.json` (the canonical bytes) and `ideation.sha256`, and add `tests/test_state_package_golden.py` comparing bytes and hash, with `PRODUCT_UPDATE_GOLDEN=1` rewriting the fixture. Verify the test passes, and that changing one item's content in the golden project fails it with a diff that names the item.
- [ ] 5.3 Document the golden procedure in the README: how to regenerate, and that a changed fixture is reviewed as a content change. Verify the documented command regenerates an identical fixture.

## 6. Documentation and integration

- [ ] 6.1 Add the endpoint to the README walkthrough with a `curl` example and a short hash verification snippet. Verify the commands run against `make run` and the snippet prints a matching digest.
- [ ] 6.2 Update `CLAUDE.md`: the stream 7 line lists `add-conversations`, `add-state-package` and `add-turns`; the testing expectations name state packages among the golden-tested documents; the conventions add "state package" as the doc's name, with no synonym. Verify the diff touches only those lines.
- [ ] 6.3 Run `make verify` from a clean checkout and `openspec validate add-state-package --strict`, and confirm every task above is checked. Verify both report no issues.

## Workflow follow-up

- Propose `add-turns`, the third change of the Conversations stream: the claim calls `assemble_state_package`, stores `render_canonical` output as an evidence object (key, version id, sha256 on the turn), and the AI service reads it under a live claim.
- `add-proposal-apply-gate`: a proposal records the `sha256` of the package it was generated from.
- Archive the change after review.
