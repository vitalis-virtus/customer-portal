# Architecture — Customer Portal

## 1. Purpose

This document records the overall logical architecture of Customer Portal. It collects the stable technology decisions that all User Stories rely on.

**In scope:**

- logical architecture: style, layers, modules, dependency rules;
- technology stack;
- cross-cutting concerns at the level of component responsibilities.

**Out of scope:**

- detailed conventions (API, Security, Persistence);
- package structure (Go project layout);
- testing strategy;
- decisions that concern individual User Stories.

These topics, if needed, are described in separate documents under separate tasks.

**Decision statuses:**

- `APPROVED` — the decision is explicitly approved. Agents and people may rely on it.
- Anything not approved does not enter the document as a decision. It is recorded in the section [Open Questions and Deferred Decisions](#9-open-questions-and-deferred-decisions).

Each decision has a short rationale. Its purpose is to preserve the intent of the decision so that, during implementation, the decision is not replaced with an alternative that violates that intent.

**Relationship to product documentation.** Product context is defined by the documents in [docs/product](../product/). Terms are used in the meanings from the [Business Glossary](../product/business-glossary.md). If this document contradicts the product documents, the agent must report the contradiction. The agent must not automatically prefer one of the documents.

Decisions approved on 2026-11-13.

## 2. Architectural Drivers

Product requirements that affect the architecture:

| Source | Impact on the architecture |
|---|---|
| [NFR-001 — Secure Authentication](../product/non-functional-requirements.md#nfr-001--secure-authentication) | Session model and the place where authentication is checked |
| [NFR-002 — Password Confidentiality](../product/non-functional-requirements.md#nfr-002--password-confidentiality) | Component responsibility for storage, responses and logs involving passwords |
| [BR-001 — Admin-Only Capabilities](../product/business-rules.md#br-001--admin-only-capabilities) | The place where authorization is checked |
| [BR-002 — Own Account Self-Service](../product/business-rules.md#br-002--own-account-self-service) | The source of the identifier of one's own account |
| [BR-007 — Blocked User Access](../product/business-rules.md#br-007--blocked-user-access) | Session model and access-state operations: a blocked user immediately loses access |
| [Product Vision](../product/product-vision.md#product-vision): future domains (Product Catalog, Orders, Support) | Modular decomposition with explicit module boundaries |

## 3. System Context

Customer Portal is a backend service that provides a REST API.

| Participant | Role |
|---|---|
| API client | Not defined. The API targets a future browser client. Currently the service is used through HTTP tools |
| PostgreSQL | Primary data store |
| Redis | Session store |

**Status:** `APPROVED`

**Rationale:** no specific client has been chosen yet. Therefore, decisions that depend on it have been moved to open questions.

## 4. Architectural Style

### Hexagonal Architecture

**Status:** `APPROVED`

Each module consists of the following logical components:

| Component | Responsibility |
|---|---|
| **Domain** | Entities, value objects, business invariants. Pure Go without dependencies on frameworks, the DB or HTTP |
| **Application** | Use cases (scenarios). They orchestrate the Domain and call ports. Ports (interfaces) are defined here |
| **Inbound adapters** | HTTP (Gin): request parsing, use case invocation, response construction. They contain no business logic |
| **Outbound adapters** | Port implementations: PostgreSQL (pgx), Redis, password hashing, etc. |
| **Composition root** | Dependency assembly (wiring) and application startup |

**Dependency rule:** dependencies point only inward. Adapters depend on Application, Application on Domain. Domain depends on nothing. Application does not know about Gin, pgx or Redis.

**Rationale:** separate Domain and Application give an unambiguous place both for business rules and for scenarios. The core can be tested without infrastructure.

### Business Rule Placement

**Status:** `APPROVED`

- **Domain:** invariants that can be checked with only the entity itself. Examples: BR-006 (a single role), BR-008 (only the `user` role can be blocked).
- **Application:** rules that need storage, external data or several components. Examples: BR-003 (unique email), BR-005 (default role), BR-007 (blocking → session deletion).

The examples are illustrative. The final placement of a specific rule is determined in the User Story.

**Rationale:** the principle is unambiguous and verifiable.

## 5. Modular Decomposition

### Deployment Form: Modular Monolith

**Status:** `APPROVED`

One service and one binary. Domains are isolated as modules with explicit boundaries.

**Rationale:** this approach combines with the hexagonal architecture and prepares the project for future domains without the operational complexity of microservices.

### MVP Modules

**Status:** `APPROVED`

| Module | Scope |
|---|---|
| `account` | Everything related to users: Account, Profile, Password, Registration, Authentication, sessions, Role, User Blocking / Unblocking |

Future domains become separate modules.

**Rationale:** the MVP functionality is one subject area, the "shared user management foundation" from the Product Vision. Splitting it now would introduce inter-module calls without real benefit. The name `account` is a term from the Business Glossary and, unlike `user`, it does not conflict with the role name.

### Inter-Module Interaction

**Status:** `APPROVED`

A module accesses another module only through its public interface (a defined contract). Access to another module's internal packages and DB tables is prohibited.

How the public interface is expressed technically is defined by the Go project layout.

**Rationale:** this preserves the boundaries of the modular monolith and makes it possible to evolve modules independently.

### Platform

**Status:** `APPROVED`

Platform is a separate logical part outside the modules. It contains shared technical infrastructure: configuration, logging, basic HTTP server infrastructure, connections to PostgreSQL and Redis.

- Module adapters and the composition root may depend on Platform.
- Platform does not depend on modules.
- Platform does not contain business logic.

**Rationale:** infrastructure is not duplicated, and the boundary is clear: everything business-related lives in modules, everything technical and shared lives in Platform.

### Composition Root

**Status:** `APPROVED`

- The application has one composition root at the entry point.
- Each module provides an assembly function. It accepts dependencies from Platform and returns the module's public interface and its HTTP routes.
- The composition root connects the modules to each other and starts the server.
- Wiring is manual, without a DI framework.

**Rationale:** a module is itself responsible for its internal assembly, so the composition root does not know about the modules' internal parts. Manual wiring is explicit and understandable.

## 6. Technology Stack

**Status:** `APPROVED`

| Aspect | Technology | Role |
|---|---|---|
| Language | Go | Backend service |
| API style | REST over HTTP | External interface |
| HTTP | Gin | Inbound adapter |
| Primary DB | PostgreSQL | Data storage |
| DB driver | pgx | Outbound adapter to PostgreSQL |
| Sessions | Redis | Session store. The session is checked on every protected request |
| Tokens | JWT access/refresh in cookies | Session transport. The access token carries the session identifier |
| Migrations | golang-migrate | DB schema management |

**Rationale:** the stack was approved by the project owner. It is consistent with the session model and the hexagonal architecture described in this document.

## 7. Cross-Cutting Concerns

### Session Model and Authentication

**Status:** `APPROVED`

- The access JWT contains the session identifier.
- On every protected request, the JWT signature and the existence and activity of the session in Redis are checked.
- An HTTP middleware (inbound adapter) extracts the token from the cookie and calls the session validation use case in Application. The use case accesses Redis through an outbound port.
- The validation result (who is acting: user identifier and role) is passed on to request processing.
- During blocking, the user's sessions are deleted, so the next request is rejected. How blocking keeps PostgreSQL and Redis consistent is described in the section [Access-State Operations](#access-state-operations).

**Rationale:** [BR-007](../product/business-rules.md#br-007--blocked-user-access) requires that a blocked user immediately loses access. The session model ensures this because there is no need to wait for the access token to expire. The cost of the decision is a Redis call on every request, and Redis becomes a critical dependency. This is acceptable for the MVP. Session validity logic stays in Application; Redis is hidden behind a port.

### Authorization

**Status:** `APPROVED`

- Permissions are finally checked by a use case in Application. It receives information about who is acting and checks permissions itself (BR-001).
- For self-service operations, the identifier of one's own account is taken from the session, not from request parameters (BR-002).
- A middleware filter may be used additionally, but it does not replace the check in the use case.

**Rationale:** access rules are business rules, so they must be in the core. They can be tested without HTTP, and the check cannot be bypassed through a new route that someone forgot to protect.

### Error Handling

**Status:** `APPROVED`

- Domain and Application return business errors by semantic category (for example: not found, conflict, forbidden, unauthenticated, invalid data).
- Outbound adapters translate technical errors (pgx, Redis) into Application errors or mark them as internal.
- The inbound adapter (HTTP) converts the error category into an HTTP response.
- Internal details (SQL errors, stack traces, etc.) do not get into the response.

The list of categories, their mapping to HTTP codes and the error body format belong to the API conventions.

**Rationale:** the core does not know about HTTP, and responses are predictable.

### Password Confidentiality

**Status:** `APPROVED`

This concern implements [NFR-002](../product/non-functional-requirements.md#nfr-002--password-confidentiality):

- hashing is hidden behind a port in Application, and an outbound adapter implements it;
- the Domain stores only the password hash and never holds the password in plain text;
- no component, including logging in Platform and inbound adapters, logs or returns the password.

**Rationale:** NFR-002 is approved, and it can be verified at the level of component responsibilities. The hashing algorithm is not tied to the core.

### Transaction Boundaries

**Status:** `APPROVED`

- The transaction boundary is defined by the use case through an abstract port. The PostgreSQL adapter implements this port.
- Application does not know about pgx.
- PostgreSQL and Redis do not have a shared transaction. How to keep changes in them consistent is decided for each use case separately.

**Rationale:** the scenario knows what must be atomic, so it defines the boundary. The dependency rule is not violated.

### Access-State Operations

**Status:** `APPROVED`

Operations that change a user's access state are blocking, login and refresh. They are performed according to the following scheme (the transactor):

1. The use case opens a PostgreSQL transaction through the transaction port and takes an exclusive lock on the user row. This way, operations for one user are performed one at a time.
2. After acquiring the lock, the use case checks the state on which the decision to grant access depends. The data read under the lock is checked: for login, the user status; for refresh, the user status and the existence of the session in Redis. Checks performed before acquiring the lock (for example, looking up the user by email and checking the password) are not considered sufficient for the access decision.
3. Within the transaction, data in PostgreSQL is changed if the operation changes it (for example, the user status during blocking).
4. Before commit, the Redis action is performed: deleting sessions during blocking, or creating or updating a session during login or refresh.
5. If an error occurred in Redis, the PostgreSQL transaction is rolled back. Otherwise, commit is performed.
6. Login and refresh return tokens to the client only after a successful commit. If the commit failed, the client receives an error, and the created or updated session is deleted on a best-effort basis.

**Caveats:**

- This is not a distributed transaction: the Redis action cannot be rolled back. The scheme is correct only for operations in which the Redis action is safe even without a commit in PostgreSQL. For blocking this holds: if the commit failed, the sessions are deleted, but the user is not blocked, and the administrator receives an error.
- Serialization per user closes the race between blocking and creating a new session. Otherwise, a login or refresh performed concurrently with blocking could create a session for a blocked user.
- The transactor is not a separate component. It is the already approved transaction port from Application. Application accesses Redis only through its port.
- A session that could not be deleted after a failed commit does not grant access, because the tokens were not passed to the client. It disappears when its TTL expires. If a refresh failed, the user may have to log in again.
- Each operation locks only one user row. Locking a single user row reduces the risk of deadlock, but does not exclude it if transactions also acquire other conflicting locks ([Persistence Conventions §5](persistence-conventions.md#isolation-level-and-locking)). Block User holds the row lock while it accesses Redis, so the wait time for login and refresh is bounded by the Redis timeout.
- The [BR-007](../product/business-rules.md#br-007--blocked-user-access) guarantee applies to requests sent after the blocking is completed: their session check results in rejection. Requests that passed the session check before the blocking was completed (in-flight) may complete.
- Unblock User is not part of these operations: it opens access and has no Redis action.

The specific serialization mechanism is deferred; see the section [Deferred Decisions](#deferred-decisions).

**Rationale:** the scheme satisfies BR-007 under network failures, timeouts and unknown outcomes of Redis and PostgreSQL commands. Such failures shift the state toward the stricter side: the session check works on a fail-closed basis, a Redis error causes a rollback, and tokens are issued only after a confirmed commit. At the same time, a protected request accesses only Redis, and there are no new components.

Limits of this guarantee:

- There is no shared ACID transaction between PostgreSQL and Redis. Consistency relies on the order of operations and conditional changes in Redis, not on atomicity ([Security Conventions §4](security-conventions.md#4-sessions-in-redis-and-revocation), [§10](security-conventions.md#10-failures-and-timeouts)).
- The scheme does not guarantee session revocation if Redis is restored from persistence made before the blocking or, after a failover, switches to a replica that has not yet received the deletion. This is an open dependency that must be closed before production ([Security Conventions §2](security-conventions.md#2-assumptions-and-limits-of-guarantees), [§12](security-conventions.md#infrastructure-and-architectural-dependencies)).

### Configuration and Logging

Configuration and logging are part of Platform. Detailed rules for them are not defined by this document.

## 8. Architectural Invariants

**Status:** `APPROVED`

Rules (Architecture Rules) that are checked during Design Review and Implementation Verification of every User Story.

Identifiers are stable. An identifier does not change and is not reused, even if the rule is changed or removed.

| ID | Rule | Verification criterion | Source |
|---|---|---|---|
| AR-001 | Domain does not depend on Application, adapters, Platform, frameworks, the DB or HTTP | Domain packages do not import Application packages, adapters, Platform, Gin, pgx, the Redis client, `net/http` | [§4](#hexagonal-architecture) |
| AR-002 | Application does not depend on specific technologies. Application accesses the outside world only through ports | Application packages do not import Gin, pgx, the Redis client, `net/http`. External calls go only through port interfaces | [§4](#hexagonal-architecture) |
| AR-003 | Inbound adapters do not contain business logic | A handler parses the request, calls a use case and constructs the response. It contains no business rule checks | [§4](#hexagonal-architecture) |
| AR-004 | A module accesses another module only through its public interface | A module's packages do not import another module's internal packages. A module's DB queries do not access another module's tables | [§5](#inter-module-interaction) |
| AR-005 | Platform does not contain business logic and does not depend on modules | Platform packages do not import module packages and do not contain business rule checks | [§5](#platform) |
| AR-006 | Every protected request goes through a session check in Redis via an Application use case | Every protected route goes through a middleware that calls the session validation use case | [§7](#session-model-and-authentication) |
| AR-007 | Authorization is finally checked by the use case. The identifier of one's own account is taken from the session | An administrative action use case checks the role. A self-service use case does not accept the account identifier from the request | [§7](#authorization) |
| AR-008 | Technical errors of adapters do not go beyond their boundaries. Internal details do not get into responses | Adapters do not return pgx or Redis errors beyond their boundaries. Responses do not contain SQL, internal messages or stack traces | [§7](#error-handling) |
| AR-009 | The password is not logged, not returned in responses and not stored in a recoverable form | Domain entities contain only the password hash. The password is absent from responses and logs | [§7](#password-confidentiality), [NFR-002](../product/non-functional-requirements.md#nfr-002--password-confidentiality) |
| AR-010 | The transaction boundary is defined by the use case | Transactions are opened only by a use case through the transaction port | [§7](#transaction-boundaries) |
| AR-011 | Access-state operations (Block User, login, refresh) are performed in a PostgreSQL transaction with an exclusive lock on the user row | The Block User, login and refresh use cases: (1) take an exclusive lock on the user row; (2) make the access decision only based on data read under the lock (status, and for refresh also the existence of the session); (3) perform the Redis action before commit, a Redis error causes a rollback; (4) login and refresh return tokens only after a successful commit | [§7](#access-state-operations), [BR-007](../product/business-rules.md#br-007--blocked-user-access) |
| AR-012 | Only technologies from section 6 are used. A new technology or a replacement of an existing one requires approval and an update of this document. A technology is anything that replaces or supplements entries of the table in section 6 | Dependencies that replace or supplement entries of §6 are absent until the document is updated | [§6](#6-technology-stack) |

### Retired

Identifiers of removed rules are reserved. They are not reused.

There are no removed rules.

## 9. Open Questions and Deferred Decisions

### Open Questions

Unresolved questions that affect the correctness of the approved architecture. Until a question is closed, the agent does not implement functionality that depends on it.

None.

### Deferred Decisions

These decisions are deliberately deferred. They do not block the approved architecture, but they are determined before implementing the functionality that depends on them. Until then, the agent cannot rely on any option.

Rows marked "Defined" have already been closed by the corresponding document. They are kept for traceability.

| Decision | Deferred to |
|---|---|
| API client type and the related cookie, CORS, CSRF settings | **Partially defined** in [Security Conventions §5](security-conventions.md#5-cookies-csrf-cors-https): an initial cookie policy for a same-site client, a session-bound CSRF token, CORS disabled. The client type (A2) and the CSRF token delivery method remain open until the client is chosen ([Security Conventions §12](security-conventions.md#infrastructure-and-architectural-dependencies)) |
| Token TTLs, refresh token rotation, claims format | **Defined:** [Security Conventions §3](security-conventions.md#3-jwt) (claims, TTL), [§4](security-conventions.md#refresh-rotation-and-reuse-detection) (refresh rotation) |
| Password hashing algorithm and parameters | **Defined:** [Security Conventions §6](security-conventions.md#6-passwords). Argon2id queue limits and timeouts are deferred until implementation ([Security Conventions §12](security-conventions.md#infrastructure-and-architectural-dependencies)) |
| Per-user serialization mechanism for access-state operations (row lock type, transaction isolation level, Redis timeouts) | **Defined:** [Persistence Conventions §5](persistence-conventions.md#5-per-user-serialization-row-locks-timeouts): `READ COMMITTED`, `SELECT … FOR UPDATE` of the user row, the PostgreSQL and Redis timeout model. Timeout values remain deferred until Infrastructure Bootstrap |
| HTTP server timeouts and graceful shutdown | **Partially defined:** the model of timeouts, limits and graceful shutdown — [API Conventions §11](api-conventions.md#11-http-server-timeout-and-limit-model). The values remain deferred until Infrastructure Bootstrap |
| List of error categories, their mapping to HTTP codes, error body format | **Defined:** [API Conventions §8](api-conventions.md#8-errors) |
| Go, PostgreSQL, Redis versions and how migrations are run | **Defined:** PostgreSQL 18 and Redis 8.x ([Persistence Conventions §2](persistence-conventions.md#environment)); migrations are run by the golang-migrate CLI as a separate step before the application starts ([Persistence Conventions §7](persistence-conventions.md#7-migrations)); the Go version is defined by the `go` directive in `go.mod` ([Package Map §8](package-map.md#go-version)). Minor and patch versions and images remain deferred until Infrastructure Bootstrap |
