# Proposal: use-platform-identity

## Why

Platform now owns users, administration, configuration and authentication, issues the `X-Session-Token` that the browser and services present, and keeps its data in the same PostgreSQL as Workspace.
The Go gateway only routes, so Workspace must authenticate every caller itself against Platform's session and authority facts, as decision 0003 anticipated.
Workspace still runs on the stand-ins from `init-workspace-core`: HS256 bearer tokens minted locally, and its own `system_principals` and `project_members` tables.
Those stand-ins cannot talk to the real gateway or the AI service, so the end-to-end flow agreed on 2026-10-29 cannot start until they are replaced.

**Delivery stream:** 1 (Workspace core), third sequential change, applied after `add-item-section-writes` is archived.
That change is reviewed and queued, and this one touches only the identity and authority seams, so the queue order stays as planned.
`CLAUDE.md`'s delivery order gains this change.

**Design doc sections implemented** (Product Workspace Design, Phase 1 v0.5 Review Edition):

- s2 Command boundary and authority (trusted identity from server context; provisioned system authority; "a browser, model, run ID or database credential does not supply domain permission")
- s3 Data model and ownership ("Module ownership is a persistence contract": owners share a database but expose guarded interfaces, and another module never manipulates their collections)
- s18 Internal owner interfaces and transactions (Platform owner guards called with the caller's open transaction and enforceable commit ordering; "an HTTP precheck cannot replace transaction participation")
- s19 Required failure demonstrations ("current-authorized receipt replay after access loss")
- C06 (Workspace and Platform deliberately share one transaction-capable database in Phase 1)

## What Changes

- **Session tokens replace bearer JWTs.** Every request carries `X-Session-Token`. Workspace resolves it through a Platform-owned function in the shared database before the request body is read. Missing, unknown, ended or expired tokens are the fixed `401`, as today. Expiry is judged by database time, never by the application clock.
- **Session validity joins the in-transaction authority check.** A write holds the caller's session `FOR SHARE` next to its membership or system-principal row, so a sign-out that commits first is seen by the write, and a sign-out that starts later waits. A read evaluates the session in the same snapshot as the data. A session that is no longer valid at the moment of the effect is `401` with nothing mutated.
- **Platform's authority facts are read only through owner interfaces.** `PlatformAuthority` keeps its protocol and is reimplemented over functions in the `platform` schema. The Workspace application role holds `USAGE` on that schema and `EXECUTE` on the contract functions, and can neither read nor write Platform's tables. Workspace migrations never create or alter anything in that schema.
- **The contract is named as assumptions for Platform to confirm or rename:** `platform.resolve_session`, `platform.session_valid`, `platform.hold_session`, `platform.member_role`, `platform.hold_member`, `platform.principal_active` and `platform.hold_principal`, with signatures in design.md and decision 0009. A rename touches one SQL file and one Python module.
- **A local stand-in implements the same contract.** One SQL file, loaded by docker-compose and by the test bootstrap, creates the `platform` schema, stand-in tables owned by a stand-in role, the contract functions and owner-only administration functions for local sessions, membership and provisioning. It is not an Alembic migration. A test pins the contract's function signatures so drift is caught in this repository.
- **BREAKING: `initializeWorkspace` no longer takes `initial_makers`.** Membership is Platform's; the Platform creation path grants the Makers and then calls `initializeWorkspace` as a provisioned system principal. The command's fingerprint payload becomes the name alone. The `grant` method leaves `PlatformAuthority`.
- **BREAKING: the Workspace stand-in tables `system_principals` and `project_members` are dropped,** with their grants, in a new Alembic revision. They are mutable tables, so no immutability decision is needed.
- **Local tooling:** `make token` and `token_cli.py` are replaced by `make session` (prints a local `X-Session-Token`, provisioning a system principal when asked) and `make member` (grants a Maker on a project). `PRODUCT_JWT_*` settings disappear. The README walkthrough is rewritten for the new flow.
- **Decision records:** `docs/decisions/0009-platform-owner-interfaces.md` records the contract and the function-not-view choice; `0003-actor-identity-stand-in.md` is marked superseded by it; `0002-guard-order.md` gains the session hold in the authority step.
- **`CLAUDE.md`:** the delivery order lists this change, and the storage rules state that Platform's data is reached only through the `platform` schema interfaces.

**Assumptions recorded for Platform to confirm** (each isolated to the stand-in SQL and the authority module):

- The header is `X-Session-Token`, carrying an opaque token that Platform hashes and stores; Workspace never sees how.
- Service callers such as the AI service authenticate with the same header, through sessions Platform administration issues for provisioned system principals, so one mechanism serves both caller kinds.
- The schema is named `platform`, its functions are `SECURITY DEFINER`, and Platform grants `EXECUTE` to the Workspace application role.
- Platform does not update a session row on ordinary activity. If it does, the `FOR SHARE` hold would contend with those updates and the hold must target a stable row; see design.md open questions.

## Capabilities

### New Capabilities

- `platform-owner-interfaces`: how Workspace reaches Platform-owned identity and authority facts: only through the contract functions in the `platform` schema, with grants that forbid anything else, migrations that never touch that schema, a pinned contract that tests check, and a local stand-in that implements it.

### Modified Capabilities

- `actor-authorization`: callers are identified by a Platform session token in `X-Session-Token` resolved through the owner interface, instead of a verified bearer JWT; session validity is checked at the moment of the effect alongside membership and provisioning; ended and expired sessions are refused with `401` and nothing mutated.
- `workspace-initialization`: the command no longer takes or records initial Makers; membership is granted by Platform's creation path; atomicity, retry and `OPERATION_ID_REUSED` requirements drop the Maker inputs.

## Impact

- `src/product_workspace/api/tokens.py` becomes `api/sessions.py`: resolution of `X-Session-Token` through `platform.resolve_session`, keeping the route-class check before the body is read. `api/auth.py` and `api/routing.py` change only their imports and the OpenAPI security scheme (API key header instead of bearer).
- `src/product_workspace/domain/identity.py`: `Principal` gains `session_id`; token facts stay out of fingerprints and receipts.
- `src/product_workspace/storage/authority.py`: `PostgresPlatformAuthority` is reimplemented over the contract functions; `grant`, `provision`, `deprovision` and `revoke` are removed; the stand-in administration moves to SQL functions that only the owner role may run.
- `src/product_workspace/commands/initialize_workspace.py`, `api/projects.py` and their tests: `initial_makers` removed from the request, the fingerprint payload and the audit obligation.
- New Alembic revision dropping `system_principals` and `project_members`; `storage/tables.py` loses both tables; `docker/postgres/platform-standin.sql` added; `docker-compose.yml` mounts it; `tests/conftest.py` loads it into the template database.
- `src/product_workspace/config.py`, `.env.example`, `Makefile`, `README.md`, `CLAUDE.md`, `docs/decisions/0002`, `0003` and new `0009`.
- Test support (`tests/support/principals.py`, `tests/api/support.py`) issues sessions through the stand-in administration functions instead of minting JWTs; every test that used the bearer helpers moves to the session helpers.
- No new runtime dependencies; `PyJWT` is removed.
- Clients: the Go gateway forwards `X-Session-Token` unchanged; nothing else in the external API changes except `initializeWorkspace`'s body.

## Out of Scope

- The project spine stays in Workspace's `projects` table; moving it to Platform ownership (doc s3, B17) is a later decision.
- Platform's real project creation path and sign-in; locally, `make session` and `make member` stand in for them.
- Stage configuration (section templates, prefixes, question sets): it is read through the same `platform` schema, but its contract is defined by the change that consumes it.
- Turn and worker identity for the AI service (the turn claim), conversations, messages and the event stream: the conversations stream.
- Audience and source eligibility checks (doc s11): a later stream.
- Caching of session resolution: one indexed read per request is the design; a cache would reintroduce revocation lag.
- Any change to the closed `409` set or to the `403` and `401` bodies.
