# Persistence Conventions — Customer Portal

## 1. Purpose

This document defines the stable rules for working with PostgreSQL and Redis in Customer Portal for all User Stories. Agents apply them during Technical Design, Implementation, Verification and Debugging.

The document details [Architecture](architecture.md), [Package Map](package-map.md), [Security Conventions](security-conventions.md) and [API Conventions](api-conventions.md), but does not change them. The conventions cannot weaken any Architecture Rule (AR) or Security Conventions rule. The Redis session model (index-first, compare-and-set, revocation, index cleanup) is defined in [Security Conventions §4](security-conventions.md#4-sessions-in-redis-and-revocation). It is not repeated here, only supplemented with implementation rules.

**In scope:**

- connections to PostgreSQL and Redis, pools, startup behavior;
- transactions, per-user serialization, row locks, the timeout model;
- schema and SQL conventions, constraints, identifiers, time;
- migration rules;
- translation of PostgreSQL errors and classification of unknown outcome;
- Redis conventions: client, keys, TTL, atomic operations, Lua, value representation;
- PostgreSQL ↔ Redis consistency;
- mandatory persistence testing scenarios.

**Out of scope:**

- the schema of specific tables, SQL queries, migrations and Lua scripts of specific Stories (Story Technical Design);
- timeout values, pool parameters, exact versions and images (Infrastructure Bootstrap);
- the Redis persistence and failover mechanism, TLS, the rollback policy in shared environments (Deployment conventions);
- test infrastructure and libraries (Testing Strategy);
- Go code.

**Statuses:**

- `APPROVED` — the decision is explicitly approved. Agents and people may rely on it.
- `OPEN` — the question is not resolved. Functionality that depends on it is not implemented until the question is closed.
- `DEFERRED` — the decision is deliberately deferred until the stated point. Until then, the agent does not rely on any option.

Items with the `OPEN` and `DEFERRED` statuses are collected in [section 13](#13-open-questions-deferred-decisions-and-dependencies).

**Mandatory values.** Parameter names, SQLSTATE, key names, formats and name templates in this document are mandatory. Examples marked with the word "for example" are illustrative.

**Relationship to other documents.** Terms are used in the meanings from the [Business Glossary](../product/business-glossary.md). If this document contradicts other documents, the agent must report the contradiction. The agent must not automatically prefer one of the documents.

Decisions approved on 2026-11-16.

## 2. Assumptions, Limits and Production Gate

### Environment

**Status:** `APPROVED`

- PostgreSQL and Redis are trusted stores ([Security Conventions §2](security-conventions.md#2-assumptions-and-limits-of-guarantees)).
- Major versions: **PostgreSQL 18**, **Redis 8.x**. Minor and patch versions and images are defined by Infrastructure Bootstrap.
- Minimum required capabilities: `gen_random_uuid()` in PostgreSQL; hash, sorted set and Lua scripts in Redis.
- For Development and Testing, a **standalone Redis** without durability guarantees is used.
- **Redis Cluster is not used.** Moving to Cluster requires a separate architectural review: multi-key operations and Lua scripts in Cluster require the keys to be in one hash slot.
- New technologies and alternatives to PostgreSQL and Redis are not introduced ([AR-012](architecture.md#8-architectural-invariants)).

### Production Gate for Redis

**Status:** `OPEN` until Deployment / production readiness.

The invariant that must hold in production:

> After a successful BlockUser, previously revoked sessions must not grant access again through a Redis restart, persistence recovery or failover.

- A standalone Redis without durability does not ensure this invariant and does not close the question ([Security Conventions §2](security-conventions.md#2-assumptions-and-limits-of-guarantees)).
- Production requires an approved mechanism that ensures the invariant, and a verification of this mechanism. This document does not choose the specific mechanism (persistence settings, replication, write acknowledgement, etc.).
- If the invariant cannot be guaranteed, this is a production readiness blocker or grounds for an explicit revision of [BR-007](../product/business-rules.md#br-007--blocked-user-access), the Security Conventions and the Architecture. Risk acceptance by the product owner is not an automatic permission to violate the current BR-007.
- The `maxmemory-policy=noeviction` requirement ([Security Conventions §4](security-conventions.md#redis-requirement)) is checked in Infrastructure Bootstrap or Deployment Verification. `PING` does not check it. The application does not change the Redis configuration.

## 3. PostgreSQL: Connection and Pool

**Status:** `APPROVED`

- One `pgxpool` per process. It is created and closed by `internal/platform/postgres`. The composition root passes the pool to the modules' PostgreSQL adapters ([Package Map §4](package-map.md#4-platform)).
- The following are configured:

| Parameter | Note |
|---|---|
| Maximum number of connections | |
| Minimum number of connections | |
| Maximum connection lifetime | |
| Maximum connection idle time | |
| Health check period | |
| Connection establishment timeout | |
| Limit on waiting for a free connection from the pool | [Section 5](#timeout-model) |
| `TimeZone=UTC` | A connection session parameter ([section 6](#time)) |
| `application_name` | A connection session parameter, for diagnostics |
| `statement_timeout` | A connection session parameter, server-side protection ([section 5](#timeout-model)) |
| Credentials | From the configuration, through environment variables ([Security Conventions §8](security-conventions.md#8-secrets)) |

- The values are defined by Infrastructure Bootstrap.
- The configuration must be valid and bounded. There is no unconditional prohibition of zero: the semantics of each parameter is checked. A value that means unbounded waiting where a limit is needed is invalid.
- TLS to PostgreSQL is defined by the Deployment conventions.
- **Startup.** The application checks the connection to PostgreSQL with a timeout. If PostgreSQL is unavailable, the application does not start (fail fast).
- A successful check at startup does not replace failure handling during operation ([section 8](#8-postgresql-errors-and-unknown-outcome)).
- Liveness and readiness endpoints are deferred until Deployment.

**Rationale:** one pool per process and creating connections only in Platform correspond to the Package Map. Fail fast is consistent with [Security Conventions §8](security-conventions.md#8-secrets).

## 4. Transactions

**Status:** `APPROVED`

### Transaction Port and Owner

- The transaction boundary is defined by the use case through the transaction port ([AR-010](architecture.md#8-architectural-invariants)).
- The port is a module `Transactor` in callback form (for example, `WithinTx(ctx, fn func(ctx context.Context) error) error`). It is declared by the module's `application` package and implemented by `adapters/outbound/postgres` of the same module ([Package Map §5](package-map.md#ports)). There are no cross-module transactions.
- The transaction is created, completed and cleaned up by the PostgreSQL adapter. `pgx.Tx` is passed through `context.Context`.
- Application does not know `pgx.Tx` and does not read the transaction from the context. Application only passes `ctx` on to the ports. This corresponds to the interpretation from [API Conventions §10](api-conventions.md#request-id): technical metadata in `context.Context` is used by Platform and adapters, and Application extracts nothing application-level from the context ([Security Conventions §7](security-conventions.md#authorization-and-who-is-acting)).
- The completion of the transaction is determined in the following order of priority:
  1. The callback returned an error or a panic occurred → `COMMIT` is not initiated, the adapter proceeds to rollback and cleanup.
  2. The callback returned `nil`, but `ctx` has already been canceled or its deadline passed before `COMMIT` started → `COMMIT` is not initiated, the adapter proceeds to rollback and cleanup.
  3. The callback returned `nil`, `ctx` is valid → the adapter initiates `COMMIT`.
  4. Cancellation or timeout occurred during an already started `COMMIT` → the [unknown outcome](#unknown-outcome-commit) rules apply.
- **Nested transactions are prohibited.** Calling `WithinTx` inside an active transaction is an error. Savepoints are not used.
- **Redis is not part of the transaction.** Redis calls inside the callback are performed independently of PostgreSQL. Their order relative to `COMMIT` is defined by [AR-011](architecture.md#8-architectural-invariants) or the use case's Consistency Analysis ([section 10](#10-postgresql--redis-consistency)).
- A general Unit of Work and new technologies are not introduced.

### Operations That Require a Transaction

- A repository operation that by contract requires an active transaction (for example, `SELECT … FOR UPDATE`) **does not silently fall back to the pool** if there is no transaction in the context. The absence of the required transaction is an explicit error.
- Regular non-transactional operations may be performed through the pool.
- The way transactional and non-transactional operations are distinguished is defined by Technical Design. The requirement not to perform a critical operation silently outside a transaction is mandatory.

### Cleanup, Rollback and Outcome

The Transactor distinguishes four different things:

| What | Guarantee |
|---|---|
| Cleanup obligation | The Transactor **always** attempts to perform rollback and releases the connection and resources on a callback error, panic and cancellation |
| Rollback success | **Not guaranteed** under any infrastructure failure. The `ROLLBACK` command itself may fail. A rollback error does not mask the original error and is not turned into success |
| PostgreSQL outcome | Determined according to [section 8](#unknown-outcome-commit). A `COMMIT` error is handled separately from a callback error, taking unknown outcome into account |
| Redis side effects | Independent of the PostgreSQL outcome. A PostgreSQL rollback does not roll them back |

**Rationale:** the callback form makes it impossible to forget the rollback, and passing the transaction in `ctx` does not change repository signatures and does not expose pgx to Application ([AR-002](architecture.md#8-architectural-invariants)). An explicit error when the transaction is absent prevents a critical operation from silently executing without a lock.

## 5. Per-User Serialization, Row Locks, Timeouts

### Isolation Level and Locking

**Status:** `APPROVED`

- The default isolation level for all transactions is `READ COMMITTED`.
- Access-state operations (Block User, login, refresh) take an exclusive lock on the user row via `SELECT … FOR UPDATE` in a transaction ([AR-011](architecture.md#8-architectural-invariants), [Architecture §7](architecture.md#access-state-operations)).
- `NOWAIT` and `SKIP LOCKED` are not used. Waiting for a lock is bounded by `lock_timeout` ([timeout model](#timeout-model)).
- `FOR NO KEY UPDATE` is not used for now. The decision may be revisited if locking needs to be optimized.
- **Deadlock.** Locking a single user row reduces the risk of deadlock, but does not exclude it if transactions also acquire other conflicting locks (for example, through additional SQL operations). Deadlock is classified according to [section 8](#sqlstate-classification).
- Concurrent scenarios are checked by integration tests ([section 11](#11-persistence-testing)).

Not defined here:

- how ChangePassword is serialized with login (Technical Design of US-005, [Security Conventions §7](security-conventions.md#login));
- how login confirms the freshness of the state checked under the lock and how rehashing is organized (Technical Design of US-002).

Both decisions must use the mechanism from this section or an explicitly approved extension of it.

**Rationale:** `READ COMMITTED` with an explicit `FOR UPDATE` directly embodies AR-011. `SERIALIZABLE` can lead to `40001` serialization_failure. An automatic retry after such an error is not mandatory, but without a separate retry policy the operation does not complete transparently for the client. For access-state operations, such a policy is complicated: Redis side effects are independent of the PostgreSQL rollback, and the Security Conventions restrict retries of such operations ([Security Conventions §10](security-conventions.md#10-failures-and-timeouts)).

### Timeout Model

**Status:** `APPROVED` — the model. Values — `DEFERRED` until Infrastructure Bootstrap.

Every operation with PostgreSQL and Redis has a finite deadline, even if the incoming `context.Context` has no deadline ([Security Conventions §10](security-conventions.md#timeouts)).

| Store | Limit |
|---|---|
| PostgreSQL | Waiting for a connection from the pool |
| | SQL execution: the `ctx` deadline and `statement_timeout` as server-side protection |
| | `lock_timeout` for waiting for a row lock |
| | The overall transaction deadline |
| Redis | `DialTimeout`, `ReadTimeout`, `WriteTimeout` |
| | The operation deadline |
| | `PoolTimeout` — waiting for a connection from the pool |

- **Consistency of values.** The values are agreed with each other taking into account latency, load, the overall HTTP request deadline ([API Conventions §11](api-conventions.md#11-http-server-timeout-and-limit-model)) and acceptable behavior on failure.
- Block User holds the row lock while it accesses Redis ([Architecture §7](architecture.md#access-state-operations)). The `lock_timeout` for login and refresh is **not necessarily** longer than this time. A shorter `lock_timeout` deliberately leads to a "temporarily unavailable" response. This is a trade-off in favor of availability, not a violation of BR-007: the transaction is rolled back without creating a session.

## 6. Schema and SQL Conventions

### Names

**Status:** `APPROVED`

- Tables, columns, constraints and indexes — `snake_case`.
- Tables are plural nouns (for example, `users`). Table names are unique within the project. There is no module prefix.
- The primary key is `id`. A reference is `<entity>_id`.
- Constraints and indexes have **explicit and stable names**. PostgreSQL's default automatic names are not used, because they depend on the form of the DDL. Template:

| Object | Template |
|---|---|
| Primary key | `<table>_pkey` |
| Unique | `<table>_<columns>_key` |
| Foreign key | `<table>_<columns>_fkey` |
| Check | `<table>_<columns>_check` |
| Index | `<table>_<columns>_idx` |

Constraint names are part of the contract: errors are translated by them ([section 8](#sqlstate-classification)) and they are written to logs ([Security Conventions §9](security-conventions.md#9-security-logging)). Renaming a constraint to which a translation is bound requires a simultaneous change of the translation.

### PostgreSQL Schema and Module Boundaries

**Status:** `APPROVED`

- All tables lie in one `public` schema.
- The owner of a table is determined by the modular architecture, the module's PostgreSQL adapter and the migration file name ([Package Map §8](package-map.md#migrations)).
- [AR-004](architecture.md#8-architectural-invariants) is mandatory: a module does not access another module's tables directly.
- Separate DB schemas and roles for modules are deferred.

### SQL

**Status:** `APPROVED`

- SQL is written manually in the module's PostgreSQL adapters and executed through pgx.
- Only parameterized queries (`$1`, `$2`, …). Values are never inserted into the query text.
- `SELECT *` is not used; columns are listed explicitly.
- Rows are read into adapter types and then translated into Domain entities. pgx types do not go beyond the adapter's boundaries ([AR-008](architecture.md#8-architectural-invariants)).
- ORMs, query builders and code generation are not used.

### NULL

**Status:** `APPROVED`

Columns are `NOT NULL` by default. `NULL` is allowed only for a value that is truly optional.

### Constraints

**Status:** `APPROVED`

- Database constraints are the last line of defense of data integrity. Business rules stay in Domain and Application ([Architecture §4](architecture.md#business-rule-placement)).
- PK, `NOT NULL`, `UNIQUE`, `CHECK` and FK are used where they correspond to approved invariants.
- **Uniqueness required by business rules is guaranteed by a `UNIQUE` constraint.** A preliminary check in Application does not replace it because of concurrent writes. It is acceptable only as an additional one. A violation is translated according to [section 8](#sqlstate-classification).
- For [BR-003](../product/business-rules.md#br-003--unique-email), the form of the constraint depends on the case and normalization of email, which remain open ([section 13](#open-questions)).
- Simple enumerations (for example, `role`, status) — `text` + `CHECK`. The PostgreSQL `ENUM` type is not used by default.
- `ON DELETE CASCADE` is not used without a separate decision. By default — `NO ACTION` / `RESTRICT`.
- FKs between tables of one module are allowed.
- The policy for FKs between tables of different modules is deferred until a second module appears and a separate architectural approval is given. There is no global prohibition, but AR-004 applies independently of it.

### Identifiers

**Status:** `APPROVED` for Account/User.

- The Account/User identifier is a **UUID v4**, column type `uuid`.
- It is generated by PostgreSQL via `gen_random_uuid()` during `INSERT`. The adapter obtains the value via `RETURNING id`.
- The text representation is the canonical lowercase UUID string. It is used in the API ([API Conventions §7](api-conventions.md#json-format)) and in Redis values (`user_id`, the key `user_sessions:<user_id>`).
- Domain and Application work with the opaque type `AccountID`. Its Go representation is defined by Technical Design.
- A new UUID library is not introduced without need.
- UUID v7 is not used.
- The decision does not automatically extend to the entities of future modules.
- `sid` and `jti` are separate values according to the Security Conventions ([section 9](#session-value-representation)).

### Time

**Status:** `APPROVED`

**Storage:**

- Points in time — `timestamptz`. `timestamp` without a time zone is not used.
- For calendar dates that are not points in time, `date` is allowed.
- Application works with time in UTC. Connection sessions have `TimeZone=UTC`.
- PostgreSQL stores microseconds. If a point in time is written and then compared, its precision is normalized to microseconds before writing.

**Time source — a controlled hybrid:**

1. Business and security-sensitive points in time, in particular the expiration times of JWTs and sessions, are set by Application through a controlled time source.
2. Values that must be consistent between JWT, PostgreSQL and Redis are taken from one computed point in time, not generated independently in each adapter.
3. For technical timestamps that do not take part in cross-component comparisons for business or security, `DEFAULT now()` is allowed. `now()` in PostgreSQL returns the time of the **start of the transaction**, not the moment of writing.
4. There is no general rule "every time column is set by Application".
5. A separate clock port for each module is not created without need. The minimum contract of the time source is defined where it is actually needed.

Time in Redis sessions — [section 9](#session-value-representation).

## 7. Migrations

**Status:** `APPROVED`

### Execution

- Migrations are executed by the golang-migrate CLI as a separate step before the application starts.
- The application does not run migrations.
- The way the CLI is integrated is defined by Infrastructure Bootstrap.

### Files

- One `migrations/` directory, one sequence ([Package Map §8](package-map.md#migrations)).
- Sequential numbering, 6 digits. The name states the owner module (for example, `000001_account_create_users.up.sql`).
- A migration that has already been applied or merged into the main branch is not edited. A correction is made with a new migration.
- Migrations do not contain secrets.
- Migrations do not create administrator accounts until P6 Administrator Provisioning is closed ([Epic Map](../product/epic-map.md#open-product-dependencies)).

### Transactionality

- Migrations are transactional where PostgreSQL allows it.
- Not every migration is atomic. `CREATE INDEX CONCURRENTLY` and other operations that cannot be performed in a transaction are allowed as explicitly documented exceptions. For them, the possibility of a partial result and the need for manual recovery are taken into account.
- How the golang-migrate driver executes an SQL file and whether it wraps it in a transaction is checked for the specific driver in the Technical Design of the first migration.

### Down

- For a reversible migration, a correct `down` is created.
- A fake `down` that silently imitates a rollback is prohibited.
- `RAISE EXCEPTION` is not a universal way to handle an irreversible `down`, because it leaves the DB in the dirty state.
- In shared environments, `down` is not run automatically. The rollback policy is defined by Deployment.

### Irreversible Migrations

- In the MVP, an irreversible migration is allowed **only by separate approval**.
- If it is approved:
  - Technical Design explicitly describes the irreversibility and its reasons;
  - the migration file is marked with a clear marker comment;
  - an automatic `down` through such a migration is not supported;
  - locally, the state is restored by recreating the DB and applying `up` from scratch;
  - rollback in shared environments is defined by a separate Deployment decision.
- **The absence of a `down` file is not a protection.** golang-migrate may treat a missing `down` as an empty migration: the version will change, but the schema will not. Before the first approved irreversible migration, a way must be defined that prevents accidentally running `down` through it (for example, a check in the migration run script).
- Expand/contract is recommended for complex changes, but is not mandatory for every migration.

### Dirty State

- There is no automatic `force`.
- The dirty state is resolved only after checking the actual state of the schema.
- `force` is not a rollback mechanism.

## 8. PostgreSQL Errors and Unknown Outcome

**Status:** `APPROVED`

### Principles

- The module's PostgreSQL adapter translates pgx errors into Application errors by semantic category ([Architecture §7](architecture.md#error-handling), [AR-008](architecture.md#8-architectural-invariants)). The HTTP codes of the categories are defined by [API Conventions §8](api-conventions.md#error-categories-and-http-codes).
- The classification lives in the module's adapter. `platform/postgres` is not extended, and no universal translation library is created.
- Business violations are translated by **constraint name**, not by message text.
- Only the SQLSTATE and the constraint name get into logs. Field values, SQL and credentials are not logged ([Security Conventions §9](security-conventions.md#9-security-logging)).

### SQLSTATE Classification

| Error | Application category |
|---|---|
| `23505` unique_violation, a known constraint, the contract defines a business error | An approved business error (for example, "conflict") |
| `23505`, an unknown constraint or without an approved translation | Internal |
| `23503`, `23502`, `23514`, a known constraint, the Story explicitly defined a semantic error | The corresponding Application error |
| `23503`, `23502`, `23514` without an approved translation | Internal |
| Class `22` (data exception) without an approved business translation | Internal |
| `pgx.ErrNoRows` | According to the method contract: "not found" or an empty result |
| `55P03` lock_not_available (`lock_timeout` fired) | Temporarily unavailable |
| `40P01` deadlock_detected | Temporarily unavailable |
| `40001` serialization_failure | Temporarily unavailable |
| Class `08` (connection errors) | Temporarily unavailable |
| Class `53` (insufficient resources) | Temporarily unavailable |
| `57P01` admin_shutdown | Temporarily unavailable |
| Timeout waiting for a connection from the pool, dependency timeout | Temporarily unavailable |
| `57014` query_canceled | By cause: `statement_timeout` or the operation deadline → temporarily unavailable. If the query was canceled because the client broke the connection, it may be impossible to produce an HTTP response ([API Conventions §8](api-conventions.md#dependency-failures-and-unknown-outcome)). During `COMMIT` — [unknown outcome](#unknown-outcome-commit) |
| `40003` statement_completion_unknown | Unknown outcome |
| Any other error, an unknown SQLSTATE | Internal |

- No "temporarily unavailable" category causes an automatic retry ([API Conventions §4](api-conventions.md#4-http-methods-and-idempotency)).
- **Deadline or cancellation of `ctx` before `COMMIT` starts:**
  - the Transactor does not initiate `COMMIT` and initiates rollback and cleanup ([section 4](#transaction-port-and-owner));
  - rollback success under an infrastructure failure is not guaranteed ([section 4](#cleanup-rollback-and-outcome));
  - since `COMMIT` was not performed, uncommitted PostgreSQL changes must not be committed;
  - the situation is usually classified as "temporarily unavailable";
  - a PostgreSQL rollback does not roll back separate Redis side effects.
- An unknown outcome of `COMMIT` itself is handled according to the [unknown outcome](#unknown-outcome-commit) rules.

### Unknown outcome COMMIT

| Situation | PostgreSQL outcome |
|---|---|
| PostgreSQL confirmed `COMMIT` | The changes are committed |
| PostgreSQL reliably rejected the transaction or confirmed rollback | The changes are not committed, the outcome is known |
| A network interruption, a lost response, a timeout, cancellation of `ctx` during `COMMIT`, `40003` or another case where the outcome has not been reliably established | **Unknown outcome** |
| An error before `COMMIT` | The transaction must be rolled back ([section 4](#cleanup-rollback-and-outcome)). This does not guarantee the absence of effects in Redis |

- The presence of an SQLSTATE in the response to `COMMIT` is **not automatic proof of rollback**. The outcome is considered known only if it has been reliably established.
- The PostgreSQL outcome and Redis side effects are considered separately. A Redis command may have executed even if its response was lost or a timeout occurred. A PostgreSQL rollback does not roll back Redis.

Behavior of use cases on unknown outcome (unchanged):

| Use case | Behavior |
|---|---|
| BlockUser | Outcome "unknown" → `500 outcome_unknown` ([Security Conventions §10](security-conventions.md#postgresql), [API Conventions §8](api-conventions.md#dependency-failures-and-unknown-outcome)) |
| Login, refresh | Tokens are not issued, the created or updated session is deleted best-effort ([AR-011](architecture.md#8-architectural-invariants), [Architecture §7](architecture.md#access-state-operations)) |

There are no automatic retries of operations with an unknown outcome ([section 10](#retries-and-idempotency)).

## 9. Redis

### Client

**Status:** `APPROVED`

- The client is `github.com/redis/go-redis/v9`. It is an implementation of the "Redis client" entry of the approved stack ([Security Conventions §1](security-conventions.md#1-purpose)).
- The client is created and closed by `internal/platform/redisclient` ([Package Map §4](package-map.md#4-platform)).
- Configuration:

| Parameter | Value |
|---|---|
| `MaxRetries` | **`-1`**. Automatic command retries are disabled. `0` does not disable retries (it means the default number of retries) and is not used for this |
| `DialTimeout`, `ReadTimeout`, `WriteTimeout`, `PoolTimeout` | Finite, from the configuration ([section 5](#timeout-model)) |
| `ContextTimeoutEnabled` | `true` |
| Logical DB | `0`. Modules are separated by key names, not by DB numbers |

- The library's default retry settings are not relied on.
- Timeout values are defined by Infrastructure Bootstrap, and TLS by the Deployment conventions. Credentials are taken from the configuration ([Security Conventions §8](security-conventions.md#8-secrets)).
- **Startup.** The application performs `PING` with a timeout. If Redis is unavailable, the application does not start (fail fast).

**Rationale:** an automatic retry of a command with an unknown outcome hides the unknown outcome from the use case. For example, a compare-and-set retry may see an already changed `refresh_jti`, and then the next legitimate refresh will look like reuse.

### Key Names

**Status:** `APPROVED`

- The keys from the Security Conventions do not change: `session:<sid>`, `user_sessions:<user_id>` ([Security Conventions §4](security-conventions.md#structure)).
- New keys have the form `<entity>:<id>`. Compound names — `snake_case`, the separator — `:`.
- Entity names are unique within the system, and the key owner is unambiguous. A module does not access another module's keys, by analogy with AR-004.
- A module prefix is not added retroactively to existing keys.
- Key names do not contain secrets and personal data in plain form.

### TTL

**Status:** `APPROVED`

- By default, every key has a TTL.
- A key without a TTL is allowed only as an explicitly approved exception. The current exception is `user_sessions:<user_id>` ([Security Conventions §4](security-conventions.md#index-cleanup)).
- A key and its TTL are set atomically, not with two independent commands.

### Sessions

**Status:** `APPROVED`

Implementation rules for the model from [Security Conventions §4](security-conventions.md#4-sessions-in-redis-and-revocation):

- **Index-first is mandatory.** `ZADD` into `user_sessions:<user_id>` is performed and confirmed before `session:<sid>` is created ([Security Conventions §4](security-conventions.md#session-creation-index-first)).
- The hash `session:<sid>` and its TTL are created **atomically**. This atomicity applies only to Redis and does not mean a shared transaction with PostgreSQL.
- `expires_at` is the single approved moment of session end. The TTL is set with an absolute `EXPIREAT` with the `expires_at` value in Unix seconds.
- `expires_at` must be a valid future moment.
- The precision and rounding of `expires_at` are the same in the JWT `exp`, the hash field, the ZSET score and the TTL ([Security Conventions §3](security-conventions.md#3-jwt)).
- The rules of compare-and-set, revocation, index cleanup and fail-closed behavior do not change ([Security Conventions §4](security-conventions.md#changing-an-existing-session-compare-and-set), [§10](security-conventions.md#redis)).

### Atomic Operations

**Status:** `APPROVED`

- Both mechanisms from [Security Conventions §4](security-conventions.md#changing-an-existing-session-compare-and-set) are allowed: Lua scripts and `WATCH` / `MULTI` / `EXEC`.
- For complex conditional and multi-key changes of session state, Lua is preferred if it simplifies correctness and implementation.
- The specific compare-and-set mechanism for each security-sensitive operation is defined by its Technical Design according to the Security Conventions.
- **The result of `EXEC`:**

| Situation | Result |
|---|---|
| `EXEC` confirmed executed | The commands were executed |
| `EXEC` confirmed canceled due to a change of a watched key | The operation was not executed. This is a conflict, not an unknown outcome |
| The `EXEC` response is lost, timeout | The outcome is unknown. Automatic retry is prohibited |

- A timeout or a lost response to any Redis command or script is an unknown outcome. For access-state operations, this is a Redis error and a rollback according to AR-011 ([Security Conventions §10](security-conventions.md#redis)).
- A universal Redis transaction framework is not created.

### Lua Scripts

**Status:** `APPROVED`

- The scripts of the module's operations lie next to the corresponding Redis adapter (`adapters/outbound/redisstore`).
- Keys are passed through `KEYS`, parameters through `ARGV`.
- Points in time are passed by Application from the approved time source ([section 6](#time)). Redis `TIME` is not used as an alternative time source for business and security-sensitive decisions.
- A runtime error of a Lua script after some writes **does not roll back** the script's previous commands. Therefore, preconditions and expected data types are checked before changes are made, as far as possible.
- A separate framework for Lua is not created.

### Prohibited Techniques

**Status:** `APPROVED`

- `KEYS` and `SCAN` are not used to search for sessions or to ensure the correctness of business or security-sensitive operations. A user's sessions are found only through the `user_sessions:<user_id>` index.
- The application does not perform `FLUSHDB` and `FLUSHALL`.
- `SCAN` is not absolutely prohibited for separately approved diagnostic or maintenance tools.

### Session Value Representation

**Status:** `APPROVED`

| Value | Representation |
|---|---|
| `sid`, `jti` | Canonical lowercase hex. At least 128 bits from `crypto/rand`; for 128 bits — 32 characters. `sid` and `jti` are separate independent values |
| `user_id` | Canonical lowercase UUID ([section 6](#identifiers)) |
| `role` | `user` or `admin` |
| `created_at`, `expires_at`, the score in `user_sessions` | Unix time in seconds, an integer |

- The representation of `sid` and `jti` is the same in JWT claims, keys, ZSET members, hash fields and when computing the `sid` fingerprint for logs ([Security Conventions §9](security-conventions.md#9-security-logging)).
- Secret values are not logged in plain form.
- Unix seconds are the format of session data, not a universal format for future Redis modules.

## 10. PostgreSQL ↔ Redis Consistency

### Consistency Analysis

**Status:** `APPROVED`

Every use case that changes both PostgreSQL and Redis has an explicit **Consistency Analysis** in its Technical Design:

| Item | What to define |
|---|---|
| Order of operations | The sequence of changes in PostgreSQL and Redis relative to `COMMIT` |
| Temporary discrepancies | Which states of discrepancy between the stores are acceptable |
| Invariants | Business and security invariants that must be preserved |
| Errors | Behavior on error and unknown outcome of **each** store ([section 8](#unknown-outcome-commit), [section 9](#atomic-operations)) |
| Cleanup / reconciliation | Whether they are needed and how they are performed |
| Transactions and serialization | Whether a PostgreSQL transaction and per-user serialization are needed |

- For Login, Refresh and Block, the approved [AR-011](architecture.md#8-architectural-invariants) scheme applies ([Architecture §7](architecture.md#access-state-operations)).
- AR-011 does not automatically extend to all future use cases with PostgreSQL and Redis.
- A new use case that changes the access state preserves the approved security invariants. Whether to apply or extend the serialization scheme is decided by separate approval. The ChangePassword mechanism (US-005) is not defined here.
- The "fail toward stricter" principle applies to security-sensitive state. For other business domains, the acceptable discrepancy is defined by the corresponding business requirements.
- Outbox, distributed transactions, distributed locks and other mechanisms are not introduced without a separate architectural decision.

### Retries and Idempotency

**Status:** `APPROVED`

- The Redis client does not retry commands automatically ([section 9](#client)).
- No automatic retries of PostgreSQL operations with an unknown outcome are added for pgx.
- Application does not automatically retry access-state operations and state-changing operations with an unknown outcome ([Security Conventions §10](security-conventions.md#postgresql), [API Conventions §4](api-conventions.md#4-http-methods-and-idempotency)).
- The retry policy for read-only operations is deferred.
- The idempotency of Block / Unblock is defined by Clarification of US-006/007.

## 11. Persistence Testing

**Status:** `APPROVED`

Testing infrastructure, libraries and tooling are defined by the Testing Strategy. Test placement — [Package Map §8](package-map.md#adapter-integration-tests).

### General Rules

- Each scenario becomes mandatory when the corresponding mechanism is implemented, not for all Stories at once.
- Transactions, row locks and atomic Redis operations are checked by **integration tests with real PostgreSQL and Redis**.
- AR-011 concurrent scenarios check the real interaction of PostgreSQL and Redis, not only Application with test doubles.
- Concurrent tests are deterministic and have explicit synchronization points (for example, the test itself opens a transaction and holds `FOR UPDATE`, and then starts a concurrent operation). `time.Sleep` is not used to provoke a race.
- Both orders of concurrent execution are checked where it matters for the BR-007 invariant.

### Mandatory Scenarios

| Mechanism | Scenario | Level |
|---|---|---|
| Transactor | The callback returned `nil` → the changes are committed | Integration |
| Transactor | Callback error, panic, cancellation → rollback, resources released | Integration |
| Transactor | Nested `WithinTx` → error | Integration |
| Transactor | An operation that requires a transaction, without one → an explicit error, not execution through the pool | Integration |
| AR-011 | Block and Login of the same user concurrently, in both orders: after Block completes, there is no session that grants access | Integration (PostgreSQL + Redis) |
| AR-011 | Block and Refresh of the same user concurrently, in both orders — the same | Integration (PostgreSQL + Redis) |
| AR-011 | Waiting for a lock longer than `lock_timeout` → "temporarily unavailable", no session created | Integration |
| Index-first | `sid` appears in the index before `session:<sid>` appears | Integration (Redis) |
| Index-first | A failure between `ZADD` and creating `session:<sid>`: there is no session, and the index entry without a session is harmless | Integration (Redis) |
| Session | Hash and TTL are created atomically; the TTL corresponds to `expires_at` | Integration (Redis) |
| Compare-and-set | Does not create a missing key | Integration (Redis) |
| Compare-and-set | Does not change state when the expected `refresh_jti` does not match | Integration (Redis) |
| Revocation | Block deletes all sessions from the index; `ZREM` only after a confirmed deletion | Integration (Redis) |
| Unknown outcome | Login / Refresh with an unconfirmed `COMMIT` → no tokens, cleanup invoked | Unit, test port implementations |
| Unknown outcome | BlockUser with an unconfirmed `COMMIT` → outcome "unknown" | Unit, test port implementations |
| Classification | SQLSTATE and `COMMIT` errors are classified according to [section 8](#8-postgresql-errors-and-unknown-outcome) | Adapter unit test |
| Constraints | Violation of a known unique constraint → an approved business error | Integration |
| Time | A `timestamptz` moment after writing and reading equals the written one with microsecond precision | Integration |
| Migrations | `up` from an empty DB to the latest version passes (when migrations appear) | Integration |

Real network failures during `COMMIT` and Redis recovery / failover tests are deferred until Testing Strategy / Deployment Verification.

## 12. Mapping to AR / BR / Security / API

| Source | Sections of this document |
|---|---|
| [AR-002](architecture.md#8-architectural-invariants) | 4 (Application does not know pgx), 6 (pgx types do not go beyond the adapter's boundaries) |
| [AR-004](architecture.md#8-architectural-invariants) | 6 (table owner, module boundaries), 9 (key owner) |
| [AR-008](architecture.md#8-architectural-invariants) | 6, 8 (error translation in the adapter) |
| [AR-010](architecture.md#8-architectural-invariants) | 4 (Transactor, the boundary is defined by the use case) |
| [AR-011](architecture.md#8-architectural-invariants) | 5 (`READ COMMITTED` + `FOR UPDATE`, `lock_timeout`), 8 (unknown outcome), 9 (no client retries), 10, 11 (concurrent tests). AR-011 does not change |
| [AR-012](architecture.md#8-architectural-invariants) | 2, 6 (no ORMs, query builders, codegen), 9 (go-redis as an implementation of the stack) |
| [BR-003](../product/business-rules.md#br-003--unique-email) | 6 (unique constraint as the guarantee; email normalization — OPEN) |
| [BR-007](../product/business-rules.md#br-007--blocked-user-access) | 2 (production gate), 5, 9, 10, 11 |
| [Security Conventions §4](security-conventions.md#4-sessions-in-redis-and-revocation) | 9 (keys, index-first, atomic creation, `EXPIREAT`, compare-and-set, prohibition of `KEYS` / `SCAN`, value representation) |
| [Security Conventions §7](security-conventions.md#7-authentication-and-authorization) | 5 (ChangePassword, login — TD dependencies) |
| [Security Conventions §9](security-conventions.md#9-security-logging) | 6 (constraint names in logs), 8 (what is logged), 9 (the `sid` fingerprint) |
| [Security Conventions §10](security-conventions.md#10-failures-and-timeouts) | 5 (timeouts), 8 (unknown outcome), 10 (no retries) |
| [API Conventions §8](api-conventions.md#8-errors) | 8 (categories → HTTP codes) |
| [API Conventions §15](api-conventions.md#15-open-questions-deferred-decisions-and-dependencies) | 6 (Account/User identifier format), 8 (recognizing an unconfirmed `COMMIT`) |

**Closed deferred decisions.** This document closes the following deferred decisions:

- from [Architecture §9](architecture.md#deferred-decisions): "Per-user serialization mechanism for access-state operations" (lock type, isolation level, timeout model; values — Infrastructure Bootstrap); "Go, PostgreSQL, Redis versions and how migrations are run" (major versions and how they are run);
- from [Package Map §9](package-map.md#deferred-decisions): "PostgreSQL and Redis versions, how migrations are run";
- from [Security Conventions §12](security-conventions.md#infrastructure-and-architectural-dependencies): "Serialization mechanism in PostgreSQL, Redis and PostgreSQL timeout values" (mechanism and model; values — Infrastructure Bootstrap);
- from [API Conventions §15](api-conventions.md#15-open-questions-deferred-decisions-and-dependencies): "The physical identifier format" for Account/User and the dependencies on the Persistence conventions.

The corresponding rows in those documents are marked as defined with references to this document. Timeout values and exact versions are marked there as deferred until Infrastructure Bootstrap.

## 13. Open Questions, Deferred Decisions and Dependencies

### Open Questions

**Status:** `OPEN`. The agent does not resolve them without separate approval.

| Question | Type | What depends on it |
|---|---|---|
| Email case and normalization for [BR-003](../product/business-rules.md#br-003--unique-email) | Product, Clarification of US-001 | The form of the unique constraint, the normalization method, the index type |
| Serialization of ChangePassword with login ([Security Conventions §12](security-conventions.md#infrastructure-and-architectural-dependencies)); P2 ([Security Conventions §12](security-conventions.md#product-questions)) | Architectural / product, Technical Design of US-005 | The invariant of the freshness of the checked state during login |
| Redis persistence and failover for production ([section 2](#production-gate-for-redis)) | Infrastructure, Deployment | Production readiness, completeness of the BR-007 guarantee |
| Key names and the counter model for P3 | Product, together with P3 | Protection against automated guessing |

### Deferred Decisions

**Status:** `DEFERRED`. Until the stated point, the agent does not rely on any option.

| Decision | Deferred to |
|---|---|
| Values of all timeouts and pool parameters of PostgreSQL and Redis | Infrastructure Bootstrap |
| Minor and patch versions, images of PostgreSQL and Redis (it is recommended to consider the Redis 8.2 extended release) | Infrastructure Bootstrap |
| golang-migrate CLI integration | Infrastructure Bootstrap |
| `noeviction` check | Infrastructure Bootstrap / Deployment Verification |
| Confirming the freshness of the state during login, rehashing, session index cleanup | Technical Design of US-002 |
| Distinguishing transactional and non-transactional repository operations, Go representation of `AccountID` | Technical Design |
| The compare-and-set mechanism for each security-sensitive operation | Technical Design of the corresponding Story |
| Transactionality of SQL file execution by the golang-migrate driver | Technical Design of the first migration |
| Protection against `down` through an irreversible migration | Before the first approved irreversible migration |
| Migration rollback policy in shared environments | Deployment |
| TLS to PostgreSQL and Redis | Deployment |
| Liveness / readiness | Deployment |
| Cross-module FK policy, separate schemas and roles | The appearance of a second module, a separate architectural approval |
| `FOR NO KEY UPDATE` | When locking needs to be optimized |
| Retry policy for read-only operations | A separate decision |
| Idempotency of Block / Unblock | Clarification of US-006/007 |
| Redis Cluster | A separate architectural review |
| Identifier format of future modules' entities | The corresponding module |
| Test infrastructure, libraries, fault injection on `COMMIT`, recovery / failover tests | Testing Strategy / Deployment Verification |

### Dependencies on Other Stages

| Stage | What is needed |
|---|---|
| Infrastructure Bootstrap | Timeout and pool values, exact versions and images, migration CLI integration, local environment, `noeviction` check |
| Testing Strategy | Integration test infrastructure and libraries, fault injection, running migration tests |
| Deployment | The Redis persistence / failover mechanism and its verification (production gate), TLS, the migration rollback policy, liveness / readiness, Deployment Verification of `noeviction` |
