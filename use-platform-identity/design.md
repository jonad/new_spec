# Design: use-platform-identity

## Context

See proposal.md for motivation.
What shapes the approach:

- `init-workspace-core` built two seams for exactly this replacement (decision 0003): `api/tokens.py` is the only module that knows the token format, called by the `WorkspaceRoute` route class before the body is read; `storage/authority.py` defines the `PlatformAuthority` protocol, whose methods take the caller's open transaction and hold authority rows `FOR SHARE` on writes.
- Both seams are backed by stand-ins in Workspace's own schema: HS256 JWTs signed with `PRODUCT_JWT_SECRET`, and the `system_principals` and `project_members` tables with admin methods (`provision`, `deprovision`, `grant`, `revoke`) that tests and `make token` use.
- Platform's sessions, users, memberships and system principals live in the same PostgreSQL as Workspace. The Go gateway forwards `X-Session-Token` without authenticating it.
- The design doc (s3, s18) requires owner interfaces, not shared tables: another module never manipulates an owner's collections, owner helpers accept the caller's transaction, and an HTTP precheck cannot replace transaction participation.
- Tests run on a throwaway PostgreSQL container: `docker/postgres/init.sql` creates the cluster roles, migrations run once into a template database, and each test gets a copy. There is no Platform code to run locally, so the test suite needs the contract itself, not Platform.
- Mutable tables may be dropped by an ordinary migration; only append-only tables need a decision record.

## Goals / Non-Goals

**Goals:**

- Authenticate every request from Platform's session facts, keeping the fixed `401` before the body is read.
- Order every write against concurrent sign-out, revocation and deprovisioning, with no cache and no application-clock comparisons.
- Confine Platform's concrete names to one SQL file and one Python module, so a rename by Platform is a two-file change.
- Make the contract testable in this repository without Platform's code.
- Remove the Workspace-owned identity stand-ins entirely.

**Non-Goals:**

- Project spine ownership, stage configuration, the AI service's turn identity, audience rules and the gateway itself (proposal, Out of Scope).
- Performance tuning of session resolution beyond indexed lookups; the capacity plan (doc s20) measures it later.

## Decisions

### 1. Owner interfaces are `SECURITY DEFINER` functions, not views or tables

The contract is a set of functions in schema `platform`, each `SECURITY DEFINER` and owned by Platform's role, with `EXECUTE` granted to `product_app`.
`product_app` holds `USAGE` on the schema and nothing else.

Why functions:

- A write must hold the authority row `FOR SHARE` until its transaction ends (decision 0002). PostgreSQL requires `UPDATE` privilege to lock rows, including through a simple view, and `UPDATE` on an auto-updatable view would let Workspace change memberships. A `SECURITY DEFINER` function runs with its owner's privileges, locks the row inside the caller's transaction and exposes nothing else.
- Platform keeps its storage private: token hashing, column layout and indexes can change without touching Workspace. A function body is Platform's; its signature is the contract.

Alternatives considered:

- **Views.** Rejected for the locking privilege problem above, and because a view still fixes column names across teams.
- **Direct `SELECT` on Platform's tables.** Rejected: ownership becomes a convention instead of a grant, and the design doc forbids it.
- **An HTTP introspection endpoint on Platform.** Rejected: identity could be checked, but membership could not be ordered against the write (doc s18), and decision 0003 already refused a remote authority check.

Every contract function sets `search_path` to `pg_catalog, pg_temp` and names its tables schema-qualified, the standard precaution for `SECURITY DEFINER`.
The stand-in does this and decision 0009 asks Platform to do the same.

### 2. The contract

Signatures are assumptions for Platform to confirm or rename (proposal, Assumptions).
Time comparisons use the database clock, `now()` in the function, never a value the application passes in.

| Function | Volatility | Result | Semantics |
| --- | --- | --- | --- |
| `platform.resolve_session(token text)` | `STABLE` | `SETOF (session_id uuid, actor_id text, actor_kind text, issued_at timestamptz, expires_at timestamptz)` | Zero rows unless `token` names a session that is not ended and not expired. `actor_kind` is `user` or `system`. |
| `platform.session_valid(session_id uuid)` | `STABLE` | `boolean` | True if the session exists, is not ended and is not expired. No lock; for snapshot reads. |
| `platform.hold_session(session_id uuid)` | `VOLATILE` | `boolean` | Locks the session row `FOR SHARE` for the rest of the caller's transaction, then returns what `session_valid` would. False when the row is missing. |
| `platform.member_role(project_id uuid, actor_id text)` | `STABLE` | `text` | The actor's current role in the project, or `NULL`. No lock. |
| `platform.hold_member(project_id uuid, actor_id text)` | `VOLATILE` | `text` | Locks the membership row `FOR SHARE`, then returns the role or `NULL`. |
| `platform.principal_active(actor_id text)` | `STABLE` | `boolean` | True if a provisioned system principal. No lock. |
| `platform.hold_principal(actor_id text)` | `VOLATILE` | `boolean` | Locks the principal row `FOR SHARE`, then returns whether it is provisioned. False when missing. |

`hold_*` functions are what writes call; the `STABLE` twins are what reads call inside their `REPEATABLE READ READ ONLY` snapshot, where locking is neither allowed nor needed.
A row lock taken inside a function belongs to the calling transaction and lasts until it ends, which is the ordering the actor-authorization spec requires: a sign-out or revocation is an `UPDATE` or `DELETE` on that row and waits for the write, or committed first and is seen by it.

The pinned-contract test reads `pg_proc` for the `platform` schema and compares name, identity arguments, result type, `provolatile` and `prosecdef` against a list in the test.
Decision 0009 carries the same table so Platform reviews one document.

### 3. Identity is resolved at the boundary and checked again in the transaction

Two checks, for two different properties:

1. **Boundary.** `WorkspaceRoute` calls the (now async) `verify_caller` before the body is read. It runs `platform.resolve_session(token)` on a short autocommit connection and builds the `Principal`. This keeps the property that a bad token is always the fixed `401`, whatever the body holds, and that an invalid body is never parsed for an unauthenticated caller.
2. **Transaction.** `PlatformAuthority.require_member` and `require_system` call `platform.hold_session(session_id)` first, then `hold_member` or `hold_principal`. Reads call `session_valid` and `member_role` in their snapshot. A session that fails here raises a new domain error, `SessionInvalidError`, mapped to the same fixed `401` body as the boundary failure. Authority failures stay `AccessDeniedError`, the fixed `403`.

`401` for a lost session and `403` for lost authority: the first means "sign in again", the second means "you are signed in but may not do this". Both bodies disclose nothing, as the actor-authorization spec requires.
The design doc is silent on the status for a session that ends mid-request; this is the recorded choice.

A database failure during boundary resolution maps to `503` with `Retry-After`, like any other database failure; the route never falls back to trusting the header.

### 4. Guard order and transaction boundaries

Decision 0002's authority step (step 2) becomes two sub-steps, and `docs/decisions/0002-guard-order.md` is updated:

1. Artifact rows `FOR UPDATE` (unchanged; none for `initializeWorkspace`).
2. Authority, through the owner interface: (a) `platform.hold_session` for the caller's session; (b) `platform.hold_principal` for system commands or `platform.hold_member` for Maker commands.
3. Receipt lookup.
4. Counters.
5. Lineage root (stream 2).
6. Inserts.

Write path in this change, `initializeWorkspace`: one short transaction: `hold_session`, `hold_principal`, receipt lookup, `projects` row `FOR SHARE`, inserts of project, stages, artifacts, receipt and obligation, commit.
Memberships are no longer inserted.

Read path, `GET /artifacts/{id}`: one `REPEATABLE READ READ ONLY` transaction: `session_valid`, artifact row, `member_role`, then the two content queries, all from one snapshot (unchanged shape).

Reads by system principals do not exist yet; when the AI service reads the state package, `require_system` gains the same `for_share=False` form.

### 5. `Principal` carries the session id; fingerprints and receipts do not

`Principal` gains `session_id`.
It is the handle the in-transaction check needs.
It never enters the canonical fingerprint or the receipt (decision 0004: token facts are transport details), so a retry from a new session by the same actor still replays, and the fingerprint of every existing command is unchanged except for `initializeWorkspace`'s payload (Decision 6).

### 6. `initializeWorkspace` without `initial_makers`

The request body loses `initial_makers`; because every request model forbids unknown fields, a client still sending it gets `422`, which is the migration signal.
The fingerprint payload becomes `{"name": ...}` and the audit obligation payload drops the Maker list.
`grant` leaves `PlatformAuthority`, so the command has no path to write membership.

Platform's creation path is expected to grant Makers in its own transaction before or after calling `initializeWorkspace`; the order does not matter to Workspace because reads check membership at read time.
Locally, `make member` does the grant.

### 7. The stand-in is one idempotent SQL file outside Alembic

`docker/postgres/platform-standin.sql` creates, if missing: the role `platform_owner` (no login), schema `platform` owned by it, stand-in tables `platform.sessions`, `platform.members` and `platform.principals`, the seven contract functions, and owner-only administration functions:

- `platform.standin_create_session(actor_id text, actor_kind text, lifetime interval) RETURNS text`: inserts a session and returns the plaintext token; the table stores `sha256` of the token. The token is two `gen_random_uuid()` values with hyphens removed, which needs no extension.
- `platform.standin_end_session(session_id uuid)`, `platform.standin_grant(project_id uuid, actor_id text, role text)`, `platform.standin_revoke(project_id uuid, actor_id text)`, `platform.standin_provision(actor_id text)`, `platform.standin_deprovision(actor_id text)`.

`EXECUTE` on the contract functions goes to `product_app`; `EXECUTE` on the administration functions goes to `product_owner` only, so tests use their existing owner engine and the application role provably cannot administer identity.
Every statement is `CREATE ... IF NOT EXISTS` or `CREATE OR REPLACE`, so loading twice keeps data.

Loading:

- docker-compose mounts the file into `/docker-entrypoint-initdb.d/` next to `init.sql`, which covers a fresh volume. Because initdb scripts run only once, `make setup` also runs `docker compose exec postgres psql -f` on the mounted file after the service is healthy, through a `standin` target, so an existing volume gets it too.
- `tests/conftest.py` executes the file as superuser into the template database, right after `init.sql`.

Why not an Alembic migration: Platform owns these objects in production, and Workspace migrations must leave the `platform` schema alone (platform-owner-interfaces spec).
Why not Python: the file is the contract Platform reads; SQL is the language both teams share.

### 8. Drop the Workspace stand-in tables

A new Alembic revision drops `project_members` and `system_principals` and revokes their grants; `storage/tables.py` loses both tables and the migration tests' table lists are updated.
`downgrade` recreates the tables empty, which is acceptable for local databases.
The revision number is the next free one at apply time: `0004` if `add-item-section-writes` has landed with `0003`.

### 9. Test support

`tests/support/platform.py` wraps the administration functions over the owner engine: `create_session(actor_id, kind, *, expires_in=...) -> Session` (token plus `Principal`), `end_session`, `grant`, `revoke`, `provision`, `deprovision`.
A `platform` fixture gives each test these helpers bound to its fresh database.
`tests/api/support.py` keeps `user_headers` and `system_headers` as names but they become `await platform.user_headers("alice")` and `await platform.system_headers("svc-platform")`, creating the session and, for system callers, provisioning the principal, as `make token` did.
`tests/support/principals.py` is removed; command-level tests obtain real principals from `create_session`, because the in-transaction check needs a real session row.

Concurrency tests keep their shape: two connections, one holding the authority row inside a write, the other ending the session or revoking, asserting that the second waits and that the order of commits decides the outcome.

### 10. Local tooling

`src/product_workspace/api/session_cli.py` replaces `token_cli.py`: `make session KIND=user|system SUB=<actor> [MINUTES=60]` connects with `PRODUCT_MIGRATION_DATABASE_URL` (the owner role), provisions a system principal when `KIND=system`, creates a session and prints the token.
`make member PROJECT=<uuid> SUB=<actor>` grants a Maker.
`PyJWT` and the `PRODUCT_JWT_*` settings are removed; `config.py` keeps only the database URL.
The OpenAPI security scheme becomes an API key header named `X-Session-Token`.

### 11. Decision records

- `docs/decisions/0009-platform-owner-interfaces.md` (new): the contract table from Decision 2, the function-not-view rationale, the `SECURITY DEFINER` and `search_path` requirement, the `401` versus `403` choice, and the stand-in loading rules.
- `docs/decisions/0003-actor-identity-stand-in.md`: status becomes "Superseded by 0009"; its "What Platform must provide" list is annotated with what was delivered and what remains (project creation path, project spine).
- `docs/decisions/0002-guard-order.md`: the authority step gains the session hold.

Numbers `0005` to `0008` are reserved by `add-item-section-writes`.

### 12. Ambiguities and divergences from the design doc

- Doc s18 describes "Platform / Knowledge owner guards(tx, ...)" semantically; this change freezes concrete SQL functions as that guard. Recorded in 0009 for Platform's sign-off.
- The doc names no status for a session that ends mid-request; `401` is chosen (Decision 3).
- Doc s3 makes the Project record Platform-owned; the `projects` table stays in Workspace for now (proposal, Out of Scope), so `initializeWorkspace` still creates it.
- `add-item-section-writes` adds a delta to `actor-authorization` for content writes; its scenarios remain valid, with "revoked" now meaning revoked by Platform through the stand-in administration.

## Risks / Trade-offs

- [Platform renames or reshapes a function] → The pinned-contract test fails on the first run with the real objects; the fix is in one SQL file and one module.
- [Platform updates the session row on ordinary activity] → `FOR SHARE` holds would contend with those updates. Asked in Open Questions; fallback is holding a separate per-session revocation row, which changes only `hold_session`'s body on Platform's side.
- [Two session lookups per write] → Both are primary-key reads; acceptable until the capacity plan measures otherwise.
- [`SECURITY DEFINER` without a fixed `search_path`] → The stand-in sets it; 0009 requires it of Platform.
- [Platform's data leaves the shared database later] → `PlatformAuthority` stays the seam, but the ordering guarantee cannot survive a remote call; that would need a new decision, as 0003 already states.
- [initdb scripts run only on a fresh volume] → `make setup` reloads the idempotent file every time.
- [Deprecating `make token` breaks muscle memory] → The README walkthrough is rewritten and the Make target list documents `session` and `member`.

## Migration Plan

1. Apply after `add-item-section-writes` is archived (proposal). Tasks are ordered so that `make verify` passes after each group.
2. Local databases: `make setup` loads the stand-in into an existing volume; `make migrate` applies the drop revision.
3. Clients: send `X-Session-Token`; stop sending `initial_makers`. The Go gateway forwards the header unchanged.
4. Rollback: downgrade the drop revision (tables return empty) and revert the code; the stand-in schema can stay, it is inert without callers.

## Open Questions

- Does Platform update a session row on ordinary activity, such as a last-seen timestamp? If so, `hold_session` should lock a stable per-session row instead; the spec and tasks do not change.
- Do service sessions expire? The contract tolerates `NULL` expiry only if Platform says so; the stand-in sets an expiry for both kinds.
- Exact header name: `X-Session-Token` is assumed; a different spelling changes one constant and the OpenAPI scheme name.
