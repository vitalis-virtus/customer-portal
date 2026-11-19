# Security Conventions — Customer Portal

## 1. Purpose

This document defines the stable technical security rules of Customer Portal for all User Stories. It details the logical architecture from [Architecture](architecture.md) and the physical structure from the [Package Map](package-map.md), but does not change them.

Architecture Rules (AR) set invariants, that is, what must be true. These conventions set specific parameters and rules for how to achieve these invariants. The conventions cannot weaken any AR.

**In scope:**

- JWT, sessions in Redis, revocation;
- cookies, CSRF, CORS, HTTPS;
- password hashing and password handling;
- technical rules of authentication and authorization;
- secrets, security logging, behavior during failures.

**Out of scope:**

- product requirements and NFR verification criteria;
- HTTP codes, the error format, route URLs (API conventions);
- the serialization mechanism in PostgreSQL and the timeout model ([Persistence Conventions](persistence-conventions.md)), timeout values (Infrastructure Bootstrap);
- deployment, the secret store, Redis persistence configuration (Deployment conventions);
- Go code.

**NFR-001.** [NFR-001](../product/non-functional-requirements.md#nfr-001--secure-authentication) has the `NEEDS_CLARIFICATION` status. These conventions make authentication more secure, but they do not close NFR-001 and do not establish its verification criteria.

**Libraries and AR-012.** Specific libraries for JWT, the Redis client and password hashing are considered an implementation of the approved stack from [Architecture §6](architecture.md#6-technology-stack) as long as they do not change architectural principles and do not add new technologies. A library that changes the approved architecture requires separate approval. This document does not change the wording of AR-012.

**Statuses:**

- `APPROVED` — the decision is explicitly approved. Agents and people may rely on it.
- `OPEN` — the question is not resolved. Functionality that depends on it is not implemented until the question is closed.
- `DEFERRED` — the decision is deliberately deferred until the stated point. Until then, the agent does not rely on any option.

Items with the `OPEN` and `DEFERRED` statuses are collected in [section 12](#12-open-questions-and-dependencies). Recommendations in that section are not approved decisions.

Terms are used in the meanings from the [Business Glossary](../product/business-glossary.md). If this document contradicts other documents, the agent must report the contradiction. The agent must not automatically prefer one of the documents.

Decisions approved on 2026-11-13.

## 2. Assumptions and Limits of Guarantees

**Status:** `APPROVED`

**Assumptions:**

- The future browser client is hosted on the same *site* as the API. If the client is cross-site, the cookie, CSRF and CORS policy must be revisited ([section 5](#5-cookies-csrf-cors-https)).
- The current environment is educational. The service is used through HTTP tools (curl, Postman), as described in [Architecture §3](architecture.md#3-system-context).
- Everything that comes from the client is untrusted. PostgreSQL and Redis are considered trusted stores.

**Limits of the BR-007 guarantees:**

- The scheme from this document satisfies [BR-007](../product/business-rules.md#br-007--blocked-user-access) under network failures, timeouts and unknown command outcomes described in [sections 4](#4-sessions-in-redis-and-revocation) and [10](#10-failures-and-timeouts).
- The scheme **does not guarantee** session revocation **under any Redis infrastructure failure**. If Redis is restored from persistence (snapshot / AOF) made before the blocking or, after a failover, switches to a replica that has not yet received the deletion, revoked sessions may be restored.
- This risk is an unresolved dependency on the Persistence / Deployment conventions. The guarantees must be defined before production deployment ([section 12](#infrastructure-and-architectural-dependencies)).
- The guarantee applies to requests sent after the blocking is completed, as BR-007 formulates it. Requests that passed the session check before the blocking was completed (in-flight) may complete ([section 12](#br-007-semantics-for-in-flight-requests)).
- These limits are also recorded in [Architecture §7, "Access-State Operations"](architecture.md#access-state-operations).

## 3. JWT

**Status:** `APPROVED`

**Signature:**

- The algorithm is HS256.
- Access and refresh are signed with **different** keys ([section 8](#8-secrets)).
- During validation, the algorithm is fixed on the server. `none` and the algorithm stated in the token header are not accepted.

**Typing.** The token type is set by the JOSE header `typ` (RFC 7515 §4.1.9, explicit typing per RFC 8725 §3.11):

| Token | `typ` |
|---|---|
| Access | `cp-access+jwt` |
| Refresh | `cp-refresh+jwt` |

- The validators are mutually exclusive: the access validator accepts only `cp-access+jwt`, the refresh validator only `cp-refresh+jwt`. Values are compared case-insensitively.
- A token without `typ` is rejected.
- There is no custom claim for the token type. The value `at+jwt` (RFC 9068) is not used, because the tokens do not conform to that profile.

**Claims:**

| Claim | Access | Refresh |
|---|---|---|
| `sid` | yes | yes |
| `jti` | — | yes |
| `iss` | yes | yes |
| `aud` | yes | yes |
| `iat` | yes | yes |
| `exp` | yes | yes |

- `user_id` and `role` are absent from the tokens. "Who is acting" is determined only by the session in Redis.
- The values of `iss` and `aud` are set by configuration.

**Validation.** A token is rejected if:

- the signature or the algorithm is wrong;
- `typ` does not correspond to the place of use;
- any of the mandatory claims is missing;
- `exp` has passed (clock skew tolerance 0);
- `iss` or `aud` do not match the expected values.

**TTL:**

| | Default value |
|---|---|
| Access JWT | 15 minutes |
| Session and refresh JWT | 7 days from login, absolute lifetime |

- Refresh rotation does not extend the session lifetime. After 7 days, the user must log in again.
- Access JWT `exp` ≤ session `expires_at`. The rule applies both during generation and during validation.
- Refresh JWT `exp` equals session `expires_at`.
- TTL values are set by configuration. The document records the model and the default values.

**Rationale:** the tokens are issued and verified by the same service, so a symmetric key is sufficient. Minimal claims do not duplicate Redis data and do not disclose extra data in the cookie. Explicit typing prevents using a token of one type in place of another.

## 4. Sessions in Redis and Revocation

**Status:** `APPROVED`

### Model

- One session corresponds to one successful login. A user may have several concurrent sessions.
- `sid` and `jti` have at least 128 bits and are generated via `crypto/rand`.

### Structure

`session:{sid}` — a Redis hash. The key's TTL equals `expires_at`.

| Field | Content |
|---|---|
| `user_id` | User identifier |
| `role` | The role at the moment of login |
| `refresh_jti` | The `jti` of the current refresh JWT, in plain form |
| `created_at` | Creation time |
| `expires_at` | The absolute session lifetime |

- The user status is not stored in the session. Login and refresh check it in PostgreSQL under the row lock ([AR-011](architecture.md#8-architectural-invariants)), and blocking deletes sessions.
- `role` is not re-read from PostgreSQL on protected requests. Any future role change mechanism must revoke or synchronize the user's active sessions. For the MVP, revocation is preferred.

`user_sessions:{user_id}` — a Sorted Set without TTL: value `sid`, score `expires_at`.

### Session Check on a Protected Request

After validating the access JWT, the session validation use case rejects the request as "unauthenticated" if at least one of the conditions is not met:

1. the key `session:{sid}` exists;
2. the record is readable, and all mandatory fields are present and have a correct format;
3. `role` is `user` or `admin`;
4. `expires_at` has not yet arrived. It is checked by the field, not only by the TTL;
5. access JWT `exp` ≤ `expires_at`.

A corrupted record is considered invalid and is not repaired. `refresh_jti` is not checked on a protected request, and PostgreSQL is not called.

### Session Creation (index-first)

1. `ZADD user_sessions:{user_id} <expires_at> <sid>`. If the command failed or the outcome is unknown, this is a Redis error: the session is not created.
2. Only after a confirmed step 1 is `session:{sid}` created with a TTL.

Bulk index cleanup by `expires_at` is not performed (["Index Cleanup"](#index-cleanup)).

### Changing an Existing Session (compare-and-set)

Refresh and any other change of an existing session are performed **only by an atomic conditional operation** (a Lua script or `WATCH`/`MULTI`/`EXEC`) that:

- does not create the key if it does not exist;
- changes the record only if `refresh_jti` equals the expected value read under the row lock;
- does not change `expires_at` and the TTL.

Unconditional overwriting of an existing session (a full `SET`) is prohibited.

### Refresh Rotation and Reuse Detection

- Each successful refresh issues a new refresh JWT with a new `jti` and writes it into `refresh_jti` with compare-and-set.
- **Reuse:** the refresh JWT signature is correct, the session exists, but `jti ≠ refresh_jti`. Then **only this session** is revoked, and the request is rejected as "unauthenticated". The user's other sessions remain.
- Reuse is treated as a *possible*, not a proven compromise. The risk of false revocation due to concurrent requests or client retries is accepted.

### Revocation

| Event | What is revoked |
|---|---|
| Blocking a user | All `session:{sid}` from the index. A deletion error is a Redis error (rollback according to AR-011). The index is not deleted as a whole. After a confirmed deletion of the sessions, their `sid` values are removed from the index via `ZREM` best-effort |
| Refresh JWT reuse | Only `session:{sid}`. After a confirmed deletion, `ZREM` of this `sid` from the index best-effort |
| Refresh for a session that does not exist | Nothing. The request is rejected |
| The session lifetime has expired | The key is deleted by the Redis TTL |

Logout and revocation after a password change are not defined (P1, P2, [section 12](#product-questions)).

### Index Cleanup

- The `user_sessions:{user_id}` index is **never deleted as a whole**. `DEL`, `UNLINK` and overwriting the key are prohibited. A late command that deletes the key may execute after Unblock and a new login and remove the record of a new active session from the index. Then the next blocking would not find this session, and BR-007 would be violated.
- **Invariant:** an active session whose tokens the client has received cannot be removed from the index during cleanup.
- A record is removed only via `ZREM` of a specific `sid` and only under one of two safe criteria:
  - **after a confirmed deletion** of `session:{sid}` by this same operation (blocking, reuse);
  - **after a confirmed absence** of `session:{sid}` (for example, the key was deleted by the Redis TTL). Such cleanup is performed only under the user row lock (AR-011), that is, serialized with login. Otherwise it could remove the `sid` of an in-progress login between its `ZADD` and the session creation.
- `sid` values are unique and are not reused. A session key that no longer exists cannot appear again: compare-and-set does not create the key. The exception is a late session creation from a login whose commit failed, but no one received the tokens of such a session. Therefore, a late `ZREM` removes only the record of a session that can no longer grant access.
- **Bulk cleanup only by `expires_at`** (`ZREMRANGEBYSCORE`) **is not performed**. A clock skew between instances, even a small one, may remove from the index the `sid` of a session whose key still exists and grants access. Then blocking would not find it.
- Cleanup is best-effort. Its error or unknown outcome does not change the outcome of the main operation.
- When exactly cleanup by key absence is performed, how it is structured (separate commands or an atomic Lua script) and whether the number of records per operation is limited is defined by Technical Design. No new technologies or components are added.
- A stale record (a `sid` in the index, but no `session:{sid}`) is harmless for correctness. The session check does not use the index, and a `DEL` of a non-existent key during the next blocking is not an error.

**Consequences for other operations:**

- **Login after Unblock** adds a new `sid` to the index next to the stale records. The next blocking will find and delete the new session.
- **Unblock** has no Redis action ([Architecture §7](architecture.md#access-state-operations)) and does not clean the index.

**Trade-off — accumulation of stale records.** Cleanup is best-effort and is not a scheduled background task, so the index size is **not guaranteed to be bounded**. Stale records accumulate if cleanup fails or is not run. For example, the index of a user who no longer logs in remains forever, because the key has no TTL. The cost is Redis memory and a longer blocking (more `DEL` of non-existent keys). For the MVP, this is accepted deliberately: the correctness of BR-007 is more important than the compactness of the index. If accumulation becomes a problem, a limiting mechanism requires a separate decision.

### Redis Requirement

`maxmemory-policy=noeviction`.

### What This Scheme Guarantees and What It Does Not

**Guarantees:**

- Every session whose tokens the client has received is recorded in the index at the moment the tokens are issued. The record remains there as long as `session:{sid}` exists: it is removed only after a confirmed deletion of the session or a confirmed absence of its key.
- Neither a late command nor index cleanup deletes the record of a session that can grant access. This does not depend on clock skew.
- A late refresh command does not restore a revoked session and does not overwrite a newer rotation.
- A session created by a late login command after a failed commit does not grant access: no one received tokens with its `sid`, even if its `sid` has already been removed from the index. It disappears by TTL.
- Login and blocking of the same user are serialized by the row lock (AR-011), so there is no race between reading the index and a new `ZADD`.

**Does not guarantee:**

- atomicity: index records without a session are possible, and they are harmless;
- preservation of Redis data during recovery from persistence and failover ([section 2](#2-assumptions-and-limits-of-guarantees));
- precision of lifetimes under clock skew between application instances. The boundary shifts by the amount of skew.

**Rationale:** `MULTI`/`EXEC` does not provide rollback, and a timeout leaves the outcome unknown. Therefore, the invariant "an active session can always be found and revoked" relies on the order of operations and conditional updates, not on atomicity. An index without a TTL cannot disappear earlier than the session. The `noeviction` policy prevents Redis from evicting the index.

## 5. Cookies, CSRF, CORS, HTTPS

### Cookies

**Status:** `APPROVED` — the initial policy for a same-site client.

| Attribute | Access cookie | Refresh cookie |
|---|---|---|
| Name prefix | `__Host-` | `__Secure-` |
| `HttpOnly` | yes | yes |
| `Secure` | yes | yes |
| `SameSite` | `Strict` | `Strict` |
| `Path` | `/` | `/api/v1/auth/refresh`, only the refresh route ([API Conventions §3](api-conventions.md#refresh)) |
| `Domain` | not set | not set |
| `Max-Age` | Access TTL | The remaining session lifetime |

Limitations:

- `__Secure-` protects against cookie substitution from subdomains more weakly than `__Host-`.
- A restricted `Path` is not a standalone security boundary.
- `SameSite=Strict` does not replace CSRF protection.
- If the chosen client is cross-site, the policy must be revisited.

### CSRF

**Status:** `APPROVED` — the session-bound mechanism and the check rules. The token delivery method and, as a consequence, the application of the cookie ↔ header check — `OPEN` ([section 12](#infrastructure-and-architectural-dependencies)).

- **GET, HEAD and OPTIONS do not change state.**
- **Session-bound CSRF token (`APPROVED`).** Token = HMAC(CSRF key, `sid`). The client passes it in the `CSRF-Token` request header ([API Conventions §9](api-conventions.md#csrf-and-origin-check)). The server computes the HMAC for the `sid` of the verified session (from the access JWT, and for refresh — from the refresh JWT) and compares it with the header in constant time. A token without a valid session or a token of another session is rejected. This check is mandatory regardless of the delivery method.
- **Signed double-submit is a possible but not mandatory option.** It applies only if, after A2 is approved, delivering the token via a cookie is chosen. The name "double-submit" refers specifically to this option. In it, the server **additionally** checks that the values of the CSRF cookie and the header match. A match of the cookie and the header by itself is insufficient, because a cookie can be planted, for example, from a subdomain. Therefore, the check of the binding to `sid` remains mandatory.
- **Token delivery method (`OPEN`).** While A2 is open, a specific delivery method is not approved. Therefore, whether the signed double-submit option applies is also not defined.
- **State-changing requests that have a session** (in particular refresh) require a valid CSRF token **always**, regardless of the presence of `Origin` or `Sec-Fetch-Site`.
- **Origin check** for all state-changing requests:
  - if `Origin` is present, it must be in the allowlist;
  - `Sec-Fetch-Site: cross-site` means rejection;
  - the absence of the headers **is not considered proof of a safe origin**.
- **Login and registration** without `Origin` / `Sec-Fetch-Site` are allowed **only in the current educational environment**, to support curl and Postman. This is not a production policy: login CSRF protection must be approved before a public browser client appears ([section 12](#infrastructure-and-architectural-dependencies)).

### CORS

**Status:** `APPROVED`

- CORS is disabled by default: `Access-Control-Allow-*` headers are not sent.
- When a cross-origin browser client is needed:
  - an exact origin allowlist in the configuration (the same as for the origin check);
  - explicit lists of methods and headers;
  - `Access-Control-Allow-Credentials: true` only for origins from the allowlist;
  - no `*` and no reflection of `Origin` from the request.

### HTTPS

**Status:** `APPROVED`

- Outside local development, the service is available only via HTTPS. Where TLS is terminated is decided separately (A3).
- There is no configuration flag that disables `Secure` (for example, `insecure_cookies`).
- Locally, `http://localhost` is used if the environment supports Secure cookies. Otherwise, local HTTPS is used.

## 6. Passwords

**Status:** `APPROVED`

This section implements [NFR-002](../product/non-functional-requirements.md#nfr-002--password-confidentiality) and [AR-009](architecture.md#8-architectural-invariants).

### Hashing

- The algorithm is **Argon2id**.
- Minimum parameters (OWASP): memory 19 MiB, 2 iterations, parallelism 1. The parameters are set by configuration and cannot be lower than the minimum.
- A 16-byte salt from `crypto/rand`, a 32-byte hash.
- Stored in the PHC string format (`$argon2id$v=19$m=…,t=…,p=…$<salt>$<hash>`).
- During verification, the hash is recomputed from the parameters stored in the string and compared in constant time (`crypto/subtle`).
- **Rehashing during login:** if the stored parameters are weaker than the current ones, the hash is recomputed after a successful verification. The organization of this operation is defined by Technical Design for US-002. Resource-intensive hashing is not necessarily performed under the row lock.

### Load Limiting

- The number of concurrent Argon2id operations is limited. There is no unbounded queue.
- Waiting in the queue is time-limited. If the time runs out, the request receives "temporarily unavailable".
- Specific limits and timeouts are defined before implementation.

### Password Handling

- The password is an opaque sequence of UTF-8 bytes: no trim, no case change, no Unicode normalization. It is deliberately accepted that visually identical passwords may have a different byte representation. NIST SP 800-63B-4 recommends NFC. Normalization is not applied in the MVP.
- The technical upper length limit is 1024 bytes. This is protection against abuse, not a password policy. The minimum length and composition are defined by P4.
- The password does not get into responses, logs, validation error messages, Domain and Redis. Domain receives only the hash.
- Types with a password mask the value during formatting (`String()` / `LogValue()`). This is an additional protection, not a replacement for the prohibition of logging secrets ([section 9](#9-security-logging)).

## 7. Authentication and Authorization

### Login

**Status:** `APPROVED`

- An unknown email, a wrong password and a blocked account receive the **same external response** of the "unauthenticated" category: `401`, `invalid_credentials` ([API Conventions §9](api-conventions.md#401-responses)).
- For a non-existent account, an Argon2id verification is performed on a pre-prepared dummy hash with the current parameters. This reduces the difference in response time, but does not guarantee identical time.
- The password is verified before the row lock is taken. The access decision is made based on data read under the lock (AR-011).
- **Freshness of the checked state.** Login does not create a session based on the password verification if the user's authentication state changed between the initial read and acquiring the row lock. For example, the password hash could have been changed by a concurrent ChangePassword. Under the lock, login confirms that the state against which the password was verified is still current. If it is not, a session is not created based on this verification.
- Argon2id is not performed under the lock. The specific confirmation mechanism and the behavior when the state has changed (rejection or a repeated verification outside the lock) are defined by the Technical Design of US-002. It must also take into account that rehashing during login also changes the stored hash.
- The same invariant applies to rehashing: the new hash is written only if the state against which the password was verified is still current. Otherwise, rehashing the old password could overwrite a new password just set by ChangePassword.
- **Dependency on US-005.** ChangePassword must change the authentication state so that login under the row lock is guaranteed to see this change, that is, it must be serialized with login. The specific method is defined by the Technical Design of US-005 within the serialization mechanism from [Persistence Conventions §5](persistence-conventions.md#isolation-level-and-locking) ([section 12](#infrastructure-and-architectural-dependencies)).
- This invariant does not resolve P2. Whether to revoke sessions created before the password change remains an open question.
- The product behavior (P5) remains open.

### Protection Against Automated Guessing

**Status:** the technical basis `APPROVED`, enabling `OPEN` (P3).

If P3 is approved, the following technical rules apply:

- counters in Redis with a TTL, without new technologies;
- separate counters per account and per IP;
- no indefinite account lockout;
- the response to exceeding the limit does not disclose whether the account exists.

**NFR-001 is clarified during Clarification of US-002, before Implementation begins. P3 remains `OPEN` until it is resolved explicitly, and it must be closed no later than the completion of US-002 and before production availability. If the P3 decision is needed for the mandatory criteria of NFR-001, it is made before implementing the dependent functionality. Public login is not launched without approved protection against automated guessing.**

### Refresh

**Status:** `APPROVED`

Refresh is a public route only in the sense that it does not require a valid access JWT. It checks the refresh JWT ([section 3](#3-jwt)), the Redis session and compare-and-set ([section 4](#4-sessions-in-redis-and-revocation)), as well as the CSRF rules ([section 5](#csrf)). The user status is checked under the row lock (AR-011).

### Authorization and "Who Is Acting"

**Status:** `APPROVED`

- **The context key** is an unexported type in the `httpapi` package (for example, `type actorKey struct{}`). The value lies in the request's `context.Context`, not in Gin's string map, and is read only through accessor functions ([Package Map §6](package-map.md#session-check-and-who-is-acting)).
- **The Actor is passed to the use case as an explicit parameter.** Application reads nothing from the context.
- **Deny by default.** Each route is either explicitly public (register, login, refresh) or goes through the session check ([AR-006](architecture.md#8-architectural-invariants)). The list of public routes is explicit and is checked at Design Review.
- **An admin use case** checks the `admin` role before it reads the target user ([BR-001](../product/business-rules.md#br-001--admin-only-capabilities), [AR-007](architecture.md#8-architectural-invariants)). [BR-008](../product/business-rules.md#br-008--administrators-cannot-be-blocked) is checked in Domain.
- **A self-service use case** takes the account identifier only from the Actor and has no identifier parameter ([BR-002](../product/business-rules.md#br-002--own-account-self-service)).
- No valid session → "unauthenticated" (`401`). There is a session, but no permissions → "forbidden" (`403`). The 401 / 403 / 404 order is defined in [API Conventions §8](api-conventions.md#401--403--404-order). The 403 or 404 policy for future resources owned by users is deferred there until the corresponding module.

## 8. Secrets

**Status:** `APPROVED`

- **Three independent keys:** access JWT signing, refresh JWT signing, HMAC for CSRF.
- Each key is cryptographically random and has at least 256 bits. The length is checked **after decoding**. The key representation format in the configuration is defined by Infrastructure Bootstrap.
- Keys and PostgreSQL and Redis credentials are passed through environment variables read by `platform/config`.
- **Fail fast:** if a key is missing or too short, the service does not start.
- There are no secrets in the repository, in default values or in logs. The configuration structure masks secrets during formatting, and this is only an additional protection.
- **Rotation in the MVP is only a key replacement.** After the replacement, existing tokens become invalid, and users log in again.
- If several service instances appear, they must use the same keys. Details are defined by the Deployment conventions.

## 9. Security logging

**Status:** `APPROVED`

### Events

| Event | Key fields |
|---|---|
| Login: success / failure | `user_id` (if known), internal reason code (`unknown_account`, `invalid_password`, `blocked`, `rate_limited`) |
| Refresh: success / failure | `user_id`, `sid` fingerprint, reason code |
| Refresh JWT reuse, session revoked | `user_id`, `sid` fingerprint (warning level) |
| Block / Unblock | `actor_user_id`, `target_user_id`, result |
| Authorization denial | `actor_user_id`, use case |
| Rejection due to CSRF / origin | route, reason |
| Infrastructure failure of the session check | component (Redis / PostgreSQL) |

Common fields for all events: event name, time, request id, result.

- Events determined in Application are passed through an outbound port (for example, `SecurityEvents`). Its adapter writes to `platform/logging` ([Package Map §5](package-map.md#ports)). Transport-level events are logged by the inbound adapter.
- Reason codes are visible only in logs and never in responses.

### Result Semantics

- An event reflects the **actual** outcome of the operation.
- A success event is logged only after the operation has been confirmed to complete.
- If the outcome of a transaction is unknown, the event is logged neither as success nor as a confirmed failure. It is marked as an unknown outcome.
- In the MVP, security logging works best-effort and does not block business operations. There is no guaranteed audit trail.

### Prohibited Data

Never logged, even with masking:

- passwords in any form;
- password hashes;
- JWTs;
- cookie values;
- CSRF tokens;
- keys and secrets;
- the `Cookie`, `Set-Cookie`, `Authorization` headers;
- request and response bodies of authentication and password change;
- **email.**

Additional rules:

- For correlation, `user_id` and the **`sid` fingerprint** are used: the first 8 bytes of SHA-256 in hex.
- Technical PostgreSQL errors are logged without field values and query parameters: only the error code and the constraint name.
- **Tokens are never passed in URLs.**
- Logging is structured. User-provided values are written only as field values.

## 10. Failures and Timeouts

**Status:** `APPROVED`

### Redis

- **Fail closed.** A protected request is not allowed without a successful session check.
- Redis unavailability does not mean an invalid session. The response has the "temporarily unavailable" category (`503`, `service_unavailable`), cookies are not cleared.
- There is no local session cache and no fallback to JWT-only verification.
- Login, refresh, block: a Redis error or an unknown command outcome means a rollback according to AR-011.

### PostgreSQL

- **Fail closed** for operations that need the DB. Protected requests that do not need the DB go through the session check only via Redis.
- If there is no response to `COMMIT`, the actual outcome of the transaction is **unknown** (unknown outcome), not necessarily a failure:
  - login and refresh without a confirmed commit do not issue tokens. The created or updated session in Redis is deleted best-effort;
  - BlockUser without a confirmed commit returns the "unknown" outcome.
- Access-state operations are not retried automatically until their idempotency is approved.

### Timeouts

- Every call to Redis and PostgreSQL, including waiting for a row lock, has a timeout.
- External operations have a bounded maximum deadline even when the incoming `context.Context` has no deadline.
- Waiting in the Argon2id queue is bounded ([section 6](#load-limiting)).
- Specific values: for the HTTP server — Infrastructure Bootstrap (model — [API Conventions §11](api-conventions.md#11-http-server-timeout-and-limit-model)); for Redis and PostgreSQL — Infrastructure Bootstrap (model — [Persistence Conventions §5](persistence-conventions.md#timeout-model)); for individual operations — the Technical Design of the corresponding Stories.

### Responses

- A panic in a handler is caught. The client receives a generic internal error, and the log is written without the request body.
- Responses do not contain technical details ([AR-008](architecture.md#8-architectural-invariants)).
- Error categories and HTTP codes, in particular `503 service_unavailable` for "temporarily unavailable", `500 outcome_unknown` for an unknown outcome and `500 internal_error` for a panic, are defined in [API Conventions §8](api-conventions.md#error-categories-and-http-codes).

## 11. Mapping to BR / NFR / AR

| Source | Sections of the conventions |
|---|---|
| [BR-001](../product/business-rules.md#br-001--admin-only-capabilities) | 7 (role check in the admin use case) |
| [BR-002](../product/business-rules.md#br-002--own-account-self-service) | 7 (identifier only from the Actor) |
| [BR-007](../product/business-rules.md#br-007--blocked-user-access) | 2 (limits of guarantees), 4 (index-first, compare-and-set, revocation, index cleanup), 10 (fail closed) |
| [BR-008](../product/business-rules.md#br-008--administrators-cannot-be-blocked) | 7 (check in Domain) |
| [NFR-001](../product/non-functional-requirements.md#nfr-001--secure-authentication) | 1, 7: status unchanged, P3 open |
| [NFR-002](../product/non-functional-requirements.md#nfr-002--password-confidentiality) | 6, 9 |
| [AR-006](architecture.md#8-architectural-invariants) | 4 (session check conditions), 7 (deny by default) |
| [AR-007](architecture.md#8-architectural-invariants) | 7 (explicit Actor, role check, self-service) |
| [AR-008](architecture.md#8-architectural-invariants) | 9 (PostgreSQL errors in logs), 10 (responses without details) |
| [AR-009](architecture.md#8-architectural-invariants) | 6, 9 |
| [AR-010](architecture.md#8-architectural-invariants), [AR-011](architecture.md#8-architectural-invariants) | 4 (the form of Redis commands within the AR-011 scheme), 7 (login and refresh under the lock), 10 (unknown outcome, rollback). AR-011 does not change |
| [AR-012](architecture.md#8-architectural-invariants) | 1 (libraries as an implementation of the stack) |

These conventions close the following deferred decisions:

- from [Architecture §9](architecture.md#deferred-decisions): "Token TTLs, refresh token rotation, claims format" and "Password hashing algorithm and parameters"; partially — "API client type and the related cookie, CORS, CSRF settings";
- from [Package Map §9](package-map.md#deferred-decisions): "Context key for "who is acting", claims format".

The corresponding rows in both documents are marked as defined with references to this document.

## 12. Open Questions and Dependencies

### Product Questions

**Status:** `OPEN`. The agent does not resolve them without separate approval by the product owner.

| ID | Question | Related Stories |
|---|---|---|
| P1 | Logout. It is not in the [MVP Scope](../product/product-vision.md#mvp-scope) | — |
| P2 | Whether to terminate other sessions after a password change | US-005 |
| P3 | Protection against automated guessing, the corresponding part of NFR-001 | US-002 |
| P4 | Password policy: minimum length, composition | US-001, US-005 |
| P5 | Account enumeration during registration (BR-003); what a blocked user sees during login | US-001, US-002 |
| P6 | Administrator Provisioning ([Epic Map](../product/epic-map.md#open-product-dependencies)) | US-006, US-007 |
| P7 | Whether re-authentication (the current password) is required for a password change | US-005 |

### BR-007 Semantics for In-Flight Requests

**Status:** resolved by the wording of [BR-007](../product/business-rules.md#br-007--blocked-user-access) (product owner decision 2026-11-13). The section is kept for traceability.

BR-007 prohibits executing requests of a blocked user sent after the blocking is completed. Requests that passed the session check before the blocking was completed (in-flight) are not subject to this prohibition and may complete. This corresponds to [Architecture §7](architecture.md#access-state-operations). HTTP server timeouts limit the duration of such requests, but they are not a condition for satisfying BR-007. They are deferred in [Architecture §9](architecture.md#deferred-decisions).

### Backlog and Story Clarification Questions

**Status:** `OPEN`. The recommendations are not approved decisions.

| Question | Recommendation for future approval |
|---|---|
| Refresh Flow does not belong to any Candidate Story in the [Epic Map](../product/epic-map.md) | At Backlog Refinement, include Refresh Flow in US-002 as part of the approved authentication model |
| Whether to perform automatic login after Registration (US-001) | At Story Clarification, do not perform automatic login in US-001, so that the scope of US-001 and US-002 remains separate |

### Infrastructure and Architectural Dependencies

| Dependency | Status | What depends on it |
|---|---|---|
| A2 — API client type | `DEFERRED` until the client is chosen | Final CSRF token delivery, revisiting `SameSite` for cross-site, enabling CORS |
| CSRF token delivery. If the frontend is on `app.example.com` and the API is on `api.example.com`, the frontend cannot read a host-only cookie set by the API | `OPEN` | Finally — A2. Whether the cookie ↔ header check (signed double-submit) applies depends on the delivery method. A temporary method for the educational environment is needed before US-002, because login creates a session |
| ChangePassword is serialized with login so that login under the row lock sees the change of the authentication state ([section 7](#login)) | `OPEN`, until the Technical Design of US-005 | The invariant of the freshness of the checked state during login and rehashing. The serialization mechanism — [Persistence Conventions §5](persistence-conventions.md#isolation-level-and-locking) |
| Login CSRF protection (fallback via `Referer` or another mechanism) | `OPEN` | Before a public browser client appears |
| A3 — where TLS is terminated, HSTS | `DEFERRED` until Deployment | HTTPS in production |
| Redis persistence and failover: revoked sessions may be restored | `OPEN`, until production readiness | Completeness of the BR-007 guarantee ([section 2](#2-assumptions-and-limits-of-guarantees)). The invariant and the production gate — [Persistence Conventions §2](persistence-conventions.md#production-gate-for-redis); the mechanism — the Deployment conventions |
| Error categories ("temporarily unavailable", etc.), HTTP codes, refresh path for `Path`, 404 or 403 | **Defined** in [API Conventions](api-conventions.md) (§3, §8, §9). The 403 or 404 policy for future resources owned by users remains `DEFERRED` until the corresponding module | Sections 5, 7, 10 |
| Serialization mechanism in PostgreSQL, Redis and PostgreSQL timeout values | **Defined** in [Persistence Conventions §5](persistence-conventions.md#5-per-user-serialization-row-locks-timeouts): the serialization mechanism (`READ COMMITTED`, `SELECT … FOR UPDATE`) and the timeout model. Timeout values — `DEFERRED` until Infrastructure Bootstrap | Section 10 |
| Argon2id queue limits and timeouts | `DEFERRED` until implementation | Section 6 |
| Key representation format, keys for local development | `DEFERRED` until Infrastructure Bootstrap | Section 8 |
| Secret store, identical keys on all instances, `noeviction` configuration | `DEFERRED` until the Deployment conventions | Sections 4, 8 |
| Idempotency of access-state operations, in particular repeated blocking | `DEFERRED` until Clarification of US-006 | Section 10 |
| Smooth key rotation (`kid`) | `DEFERRED`, when needed | Section 8 |
| Guaranteed audit trail | `DEFERRED`, a separate decision | Section 9 |
| Whether to expose the CSRF middleware through the `account` module's public contract for future modules | `DEFERRED`, a possible future clarification of the [Package Map](package-map.md#6-module-public-contract) | Future modules' protected state-changing routes |

### What Must Be Closed Before Implementing US-001 and US-002

| Story | What to close |
|---|---|
| US-001 Register Account | P4; P5 (registration); Clarification on automatic login; HTTP server timeout and limit values (Infrastructure Bootstrap) |
| US-002 Log In | NFR-001 (clarify at Clarification of US-002, before Implementation); P3 (no later than the completion of the Story and before production; before implementing the dependent functionality, if its decision is needed for the mandatory criteria of NFR-001); P5 (blocked user); temporary CSRF token delivery; placement of Refresh Flow; key format (Infrastructure Bootstrap); limits and timeouts (Argon2id, Redis, PostgreSQL); clearing cookies in `401` responses ([API Conventions §9](api-conventions.md#cookies-in-error-responses)); Technical Design (confirming the freshness of the checked state under the lock, rehashing, session index cleanup, libraries) |

### Contradictions Between Documents

| Documents | Contradiction | Status |
|---|---|---|
| This document ↔ [Architecture §7, "Access-State Operations"](architecture.md#access-state-operations) | The rationale in architecture.md claimed that the scheme "guarantees BR-007 under any failure" and that "all failures shift the state toward the stricter side". According to [section 2](#2-assumptions-and-limits-of-guarantees), this is not so for recovery from persistence and Redis failover | Resolved 2026-11-13: architecture.md was synchronized, and the limits of the guarantee are described there explicitly. The dependency on Redis persistence and failover itself remains `OPEN` ([infrastructure dependencies](#infrastructure-and-architectural-dependencies)) |
