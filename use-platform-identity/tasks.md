# Tasks

Work top to bottom; `make verify` must pass before a task is checked.
Groups are ordered so the old stand-ins keep working until the group that replaces them, and every group leaves the suite green.

## 1. Platform contract stand-in

- [ ] 1.1 Write `docker/postgres/platform-standin.sql`: role `platform_owner`, schema `platform` owned by it, stand-in tables `sessions`, `members`, `principals`, the seven contract functions from design.md Decision 2 (all `SECURITY DEFINER`, `search_path` fixed, database-time expiry) and the owner-only administration functions from Decision 7, every statement idempotent. Verify by loading the file twice into a fresh database with `psql` and checking the second load keeps a session created after the first.
- [ ] 1.2 Load the stand-in in tests: `tests/conftest.py` executes the file as superuser into the template database after `init.sql`. Add `tests/support/platform.py` with `create_session`, `end_session`, `grant`, `revoke`, `provision`, `deprovision` over the owner engine and a `platform` fixture. Verify with a test that creates a session through the helper and resolves it with `platform.resolve_session` as `product_app`.
- [ ] 1.3 Add `tests/storage/test_platform_contract.py`: the pinned signature list compared against `pg_proc` (name, identity arguments, result type, volatility, security mode), a failure message naming each drifted or missing function, and the `product_app` grant tests from the platform-owner-interfaces spec (cannot select any `platform` table, cannot execute an administration function). Verify the pinned test passes and that renaming one function in a scratch copy of the SQL makes it fail by name.
- [ ] 1.4 Add the migrations-untouched test: snapshot the `platform` schema's objects and definitions, run `alembic upgrade head` and `downgrade base`, assert the snapshot is identical after each step. Verify it passes with the current migrations.
- [ ] 1.5 Add stand-in behaviour tests: session lifecycle (resolve, end, resolve returns nothing), expiry in the past by database time returns nothing, `hold_session` and `hold_member` hold `FOR SHARE` so that an `end_session` or `revoke` on another connection waits until the holding transaction ends. Verify with two real connections, following `tests/support/concurrency.py`.
- [ ] 1.6 Mount the file in `docker-compose.yml` beside `init.sql` and add a `standin` Make target that runs `docker compose exec postgres psql` on the mounted file; `make setup` calls it after the service is healthy. Verify `make setup` on an existing volume reports the schema loaded and `make setup` after `docker compose down -v` works too.
- [ ] 1.7 Draft `docs/decisions/0009-platform-owner-interfaces.md` with the contract table, the function-not-view rationale, the `SECURITY DEFINER` and `search_path` requirement and the stand-in loading rules, marked "Proposed, awaiting Platform confirmation". Verify its table lists exactly the signatures pinned in 1.3.

## 2. Session identity at the boundary

- [ ] 2.1 Add `session_id` to `Principal` in `domain/identity.py` and update `tests/domain` accordingly. Verify `make verify` passes with `tests/support/principals.py` giving every principal a placeholder session id for now.
- [ ] 2.2 Replace `api/tokens.py` with `api/sessions.py`: `resolve_session(token, engine)` runs `platform.resolve_session` on a short autocommit connection and returns a `Principal`; missing, empty, unknown, ended or expired tokens raise `AuthenticationError`. Make `verify_caller` async, keep it inside `WorkspaceRoute` before the body is read, switch the OpenAPI scheme to an API key header `X-Session-Token`, and map a database failure during resolution to the existing `503`. Verify with `tests/api/test_sessions.py` (replacing `test_tokens.py`) covering every actor-authorization boundary scenario, including the malformed-body-still-401 case and the bearer-header-ignored case.
- [ ] 2.3 Switch API test support to sessions: `tests/api/support.py` drops the JWT helpers; `user_headers` and `system_headers` become async helpers on the `platform` fixture that create a session (and provision the principal for system callers) and return the `X-Session-Token` header. Update `tests/api/test_app.py`, `test_projects.py` and `test_artifacts.py` to the new helpers. Verify `make verify` passes with the stand-in sessions and the still-existing `system_principals` and `project_members` tables.
- [ ] 2.4 Remove `PyJWT` from `pyproject.toml`, the `TokenSettings` and `PRODUCT_JWT_*` variables from `config.py` and `.env.example`, and the `tokens` field from `Settings`. Verify `uv sync` and `make verify` pass and `uv.lock` no longer lists `pyjwt`.
- [ ] 2.5 Replace `token_cli.py` with `session_cli.py` and `make token` with `make session KIND=user|system SUB=<actor> [MINUTES=60]`, connecting as the owner role through `PRODUCT_MIGRATION_DATABASE_URL`, provisioning the principal for `KIND=system`, printing the token. Verify with `tests/api/test_session_cli.py` (replacing `test_token_cli.py`): the printed token resolves, a system session is provisioned, a bad actor id is rejected.
- [ ] 2.6 Update `README.md` where it mints tokens: the quick start, "Try the API" headers (`X-Session-Token`) and the Make targets table (`session` replaces `token`). Verify the documented `make session` command runs as written against docker-compose.

## 3. initializeWorkspace without initial Makers

- [ ] 3.1 Remove `initial_makers` from `InitializeWorkspaceRequest`, the fingerprint payload, the audit obligation payload and the `grant` call in `commands/initialize_workspace.py`, and from `InitializeWorkspaceBody` in `api/projects.py`. Verify with the updated `tests/commands/test_initialize_workspace.py`, `test_pipeline.py` and `tests/api/test_projects.py`: successful initialization, exact replay, `OPERATION_ID_REUSED` on a different name, the no-op repeat, and `422` for a body that still carries `initial_makers`.
- [ ] 3.2 Update the fingerprint golden vector and `tests/domain/test_fingerprint.py` for the new payload. Verify the pinned bytes change only for `initializeWorkspace`.
- [ ] 3.3 Update the README initialization step to the new body and note that membership is granted separately. Verify the documented `curl` returns `201`.

## 4. Authority through the contract

- [ ] 4.1 Add `SessionInvalidError` to `domain/errors.py` and map it in `api/errors.py` to the fixed `401` body. Verify with a unit test that the body is byte-identical to the boundary `401`.
- [ ] 4.2 Reimplement `PostgresPlatformAuthority` over the contract functions: `require_system` calls `hold_session` then `hold_principal`; `require_member` calls `hold_session` then `hold_member` for writes and `session_valid` then `member_role` for `for_share=False`; `grant`, `provision`, `deprovision` and `revoke` are removed from the class and `grant` from the protocol. Verify with `tests/storage/test_authority.py` rewritten over `tests/support/platform.py`: admitted, refused, revoked, deprovisioned, unknown, wrong kind, snapshot read without locking.
- [ ] 4.3 Add the concurrency tests for sessions and authority with two real connections: revocation waits for an in-flight member check, deprovisioning waits for an in-flight system check, session end waits for an in-flight write, and a session end committed first makes the write fail. Verify both orderings produce the outcomes the actor-authorization spec states.
- [ ] 4.4 Move command-level tests to real sessions: delete `tests/support/principals.py`; `tests/commands/*` obtain principals from `create_session`. Add the effect-time tests from the spec: session ends before the write checks it (`401`, nothing created), replay after the session ended (`401`, nothing disclosed, no row change), read after the session ended (`401`), Platform grants and revokes a Maker (`200` then `403`). Verify `make verify` passes.
- [ ] 4.5 Add `make member PROJECT=<uuid> SUB=<actor>` through `session_cli.py` (or a sibling `member` subcommand) calling `platform.standin_grant` as the owner role, and add its test. Update the README walkthrough to grant `alice` with it before reading. Verify the documented walkthrough reaches the `200` read and the `403` non-member read.
- [ ] 4.6 Update `docs/decisions/0002-guard-order.md`: the authority step holds the session, then the principal or membership, through the owner interface. Verify the document's `initializeWorkspace` sequence matches the implemented one.

## 5. Drop the Workspace stand-in tables

- [ ] 5.1 Add the next Alembic revision dropping `project_members` and `system_principals` after revoking their grants, with a downgrade that recreates them empty. Remove both tables from `storage/tables.py` and from the migration tests' table lists. Verify `tests/storage/test_migrations.py` and `test_immutability.py` pass, and `make migrate` upgrades a docker-compose database that was at the previous head.
- [ ] 5.2 Search the code and tests for any remaining reference to the dropped tables, `mint_token`, `verify_bearer`, `PRODUCT_JWT` or `token_cli`, and remove it. Verify with `grep` returning nothing outside `openspec/changes/archive` and the decision records.

## 6. Documentation and records

- [ ] 6.1 Rewrite the remaining README sections: Prerequisites and Quick start (`make setup` loads the stand-in), Configuration (no JWT variables), Tests (the stand-in is loaded into the template database), Database roles (`platform_owner` and what `product_app` may execute). Verify every command in the README runs as written.
- [ ] 6.2 Update `CLAUDE.md`: add `use-platform-identity` as the third change of stream 1 in the delivery order, and add a storage rule that Platform's data lives in the same PostgreSQL and is reached only through the `platform` schema functions, never by Workspace migrations or direct table access. Verify the file reads consistently with the proposal.
- [ ] 6.3 Mark `docs/decisions/0003-actor-identity-stand-in.md` as superseded by 0009 and annotate its "What Platform must provide" list with what this change delivered and what remains (project creation path, project spine). Finalize 0009 with the `401` versus `403` choice and the open questions for Platform. Verify both documents link to each other.
- [ ] 6.4 Edit the Purpose line of `openspec/specs/workspace-initialization/spec.md` (created when `init-workspace-core` was archived) to drop "initial Makers". Verify `openspec validate --specs` passes.

## 7. Integration

- [ ] 7.1 Run `make verify` from a clean checkout and `docker compose down -v`, then `make setup`, `make migrate` and the full README walkthrough: `make session` for `svc-platform` and `alice`, initialize, `make member`, read as `alice` (`200`), read as `bob` (`403`), end `alice`'s session through `psql` and read again (`401`). Verify every status matches.
- [ ] 7.2 Run `openspec validate use-platform-identity --strict` and confirm every task above is checked. Verify the output reports no issues.

## Workflow follow-up

- Send `docs/decisions/0009-platform-owner-interfaces.md` to Platform for confirmation of the schema name, header name, function signatures, `SECURITY DEFINER` ownership and the session-row update question; a rename changes `docker/postgres/platform-standin.sql`, `storage/authority.py` and the pinned test.
- Archive the change after review; the `actor-authorization` and `workspace-initialization` deltas then merge into the main specs.
