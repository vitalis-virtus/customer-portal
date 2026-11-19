# API Conventions — Customer Portal

## 1. Purpose

This document defines the stable HTTP API design rules of Customer Portal for all User Stories. Agents apply them during Clarification, Technical Design, Implementation and Verification.

The document details [Architecture](architecture.md), [Package Map](package-map.md) and [Security Conventions](security-conventions.md), but does not change them. The conventions cannot weaken any Architecture Rule (AR) or Security Conventions rule.

**In scope:**

- resource naming, URLs, versioning, route groups;
- HTTP methods, idempotency, success codes;
- request parsing rules and the JSON format;
- error format, HTTP codes, catalog of common error codes;
- the HTTP level of authentication and CSRF (with references to Security Conventions);
- router behavior, caching, Request ID;
- the HTTP server timeout and limit model;
- general rules for collections;
- OpenAPI maintenance rules.

**Out of scope:**

- route contracts of specific Stories. They are defined by the Story's Technical Design according to these conventions;
- security rules: JWT, sessions, cookies, the CSRF mechanism, passwords (Security Conventions);
- timeout and limit values (Infrastructure Bootstrap);
- the serialization mechanism and PostgreSQL and Redis timeouts (Persistence conventions);
- the client application and its refresh flow;
- Go code.

**Statuses:**

- `APPROVED` — the decision is explicitly approved. Agents and people may rely on it.
- `OPEN` — the question is not resolved. Functionality that depends on it is not implemented until the question is closed.
- `DEFERRED` — the decision is deliberately deferred until the stated point. Until then, the agent does not rely on any option.

Items with the `OPEN` and `DEFERRED` statuses are collected in [section 15](#15-open-questions-deferred-decisions-and-dependencies).

**Mandatory values.** Paths, header names, HTTP codes, error codes and media types in this document are mandatory. Examples marked with the word "for example" are illustrative.

**Relationship to other documents.** Terms are used in the meanings from the [Business Glossary](../product/business-glossary.md). If this document contradicts other documents, the agent must report the contradiction. The agent must not automatically prefer one of the documents.

Decisions approved on 2026-11-13.

## 2. General Principles and Resource Naming

**Status:** `APPROVED`

- The API is resource-oriented (REST): the path names a resource, the method names an action.
- Resources are named with nouns. Collections are plural nouns (for example, `/users`).
- A command-style endpoint is allowed as an exception for actions with business meaning that do not reduce to changing a resource. Form: `POST /{resource}/{id}/{command}`, where `command` is a verb in kebab-case.
- Path segments are lowercase. Compound names are kebab-case.
- Path parameters in the contract description are camelCase (for example, `{userId}`).
- No trailing `/` and no file extensions (`.json`).

### Data in URLs

- Email, other personal data and secrets are not passed in URLs: neither in the path nor in query parameters. A search by such data passes them in the request body.
- Tokens are not passed in URLs ([Security Conventions §9](security-conventions.md#9-security-logging)).
- Identifiers in URLs do not contain personal data. The client treats them as opaque values.
- The Account/User identifier is a UUID v4, a canonical lowercase string ([Persistence Conventions §6](persistence-conventions.md#identifiers)). The identifier format of future modules' entities is not defined, see [section 15](#deferred-decisions).

**Rationale:** uniform naming rules do not force agents to invent the route form in every Story. URLs get into logs, history and proxies, so personal data has no place there.

## 3. URLs, Versioning, Route Groups

### Prefix and Versioning

**Status:** `APPROVED`

- All API routes are placed under the `/api/v1` prefix.
- There is one version for the whole API. Only the major version is stated in the path.
- A breaking change requires a new major version. Adding an optional response field is not a breaking change.
- The composition root creates the `/api/v1` group and passes it to the module. The module registers its routes inside the group and does not define the prefix ([Package Map §6](package-map.md#http-routes)).

The policy of supporting several major versions simultaneously is deferred ([section 15](#deferred-decisions)).

### Route Groups

**Status:** `APPROVED`

| Group | Purpose | Rules |
|---|---|---|
| `/api/v1/me` | Self-service resources of one's own Account | The user is determined only from the Actor. Routes have no parameter with an account identifier ([BR-002](../product/business-rules.md#br-002--own-account-self-service), [AR-007](architecture.md#8-architectural-invariants)). Subresources are defined by the corresponding Stories |
| `/api/v1/auth` | Session operations | The login and registration routes are defined by the corresponding Stories. Logout is product question P1 |
| `/api/v1/admin/...` | Administrative API surface | This is a naming convention, not an authorization mechanism. The `admin` role is finally checked by a use case in Application ([BR-001](../product/business-rules.md#br-001--admin-only-capabilities), [AR-007](architecture.md#8-architectural-invariants)) |

### Refresh

**Status:** `APPROVED`

- Refresh route: `POST /api/v1/auth/refresh`.
- The `Path` of the refresh cookie equals `/api/v1/auth/refresh` ([Security Conventions §5](security-conventions.md#cookies)).
- After switching to a new major version, the refresh path will change, and existing refresh cookies will no longer be sent. This is an acceptable consequence of a breaking change.

**Rationale:** a version in the path is simple, visible in logs and compatible with Gin route groups. A separate `/me` root without an identifier makes it impossible to pass someone else's identifier into a self-service operation.

## 4. HTTP Methods and Idempotency

**Status:** `APPROVED`

| Method | Purpose | Safe | Idempotent |
|---|---|---|---|
| `GET`, `HEAD` | Reading | yes | yes |
| `POST` | Resource creation, command | no | no guarantee by default |
| `PUT` | Full replacement of a resource or an approved idempotent state setting | no | yes |
| `PATCH` | Partial update ([section 7](#partial-updates)) | no | no guarantee by default |
| `DELETE` | Deletion | no | yes |

- `GET`, `HEAD` and `OPTIONS` do not change state ([Security Conventions §5](security-conventions.md#csrf)).
- Idempotency means that repeated requests have the same end effect, not an identical HTTP response. For example, a repeated `DELETE` may return `404` after the first `204`.
- If a `POST` or `PATCH` operation is idempotent, this is explicitly stated in its contract. Without such a statement, the agent does not consider the operation idempotent.
- `Idempotency-Key` is not used in the MVP.
- The server does not automatically retry state-changing operations with an unknown outcome. The client also must not do this until a safe-retry guarantee is approved for the operation ([Security Conventions §10](security-conventions.md#10-failures-and-timeouts)).
- There is no global prohibition on retrying read-only operations. A retry policy for them may be defined later.

The idempotency and form of Block / Unblock are deferred ([section 15](#deferred-decisions)).

**Rationale:** RFC 9110 semantics are well known and do not require new mechanisms. `Idempotency-Key` requires a key store and has no consumer in the MVP.

## 5. Success Codes

**Status:** `APPROVED`

| Code | When |
|---|---|
| `200 OK` | Success with a response body |
| `201 Created` | The request created a new resource |
| `204 No Content` | Success without a response body |

- `Location` in a `201` response is returned if the created resource has a defined canonical URI. It is not required that the client can read the resource at this URI now. An artificial URI is not created just for the sake of `Location`.
- The `201` body is a representation of the created resource or is absent. The Story decides.
- `202 Accepted` is not used, because there are no asynchronous operations in the MVP. A future Story may introduce it.
- A `2xx` response does not have an error body. An error is never returned with a `2xx` code.
- The specific success code and body of a route are defined by the Story according to these rules.

## 6. Requests

### Content-Type

**Status:** `APPROVED`

- A non-empty request body must have `Content-Type: application/json`. The `charset=utf-8` parameter is allowed.
- For a request without a body, `Content-Type` is not required.

Check priority:

| Route | Request | Response |
|---|---|---|
| Does not accept a body | A non-empty body with any `Content-Type` or without one | `400`, `invalid_request_body` |
| Accepts a JSON body | A non-empty body with a media type other than `application/json` | `415`, `unsupported_media_type` |
- The `Accept` header is not negotiated. The `406` response is not used.

### Body Size

**Status:** `APPROVED`

- One global request body size limit applies. It applies to reading the body, that is, to JSON parsing and to any processing.
- Exceeding it → `413`, `request_too_large`.
- The limit value is set by the configuration ([section 11](#11-http-server-timeout-and-limit-model)).

Overriding the limit for individual routes is deferred ([section 15](#deferred-decisions)).

### JSON decoding

**Status:** `APPROVED`

The body is parsed strictly, using Go stdlib. The following are rejected with `400`:

| Case | `code` |
|---|---|
| Syntactically invalid JSON | `malformed_json` |
| Unknown field | `invalid_request_body`, an `errors[]` element with `code: unknown_field` |
| Field type mismatch | `invalid_request_body`, an `errors[]` element with `code: invalid_type` |
| Data after a single JSON value | `invalid_request_body` |
| Unexpected root value type (an object is expected by default) | `invalid_request_body` |
| Empty body where it is required | `invalid_request_body` |
| Violation of field rules | `validation_failed` |

**Accepted limitations of `encoding/json`.** No custom parser is created for the MVP. It is deliberately accepted that:

- duplicate keys are processed sequentially: a later value may replace the earlier one or be merged with it depending on the Go type;
- field names may be matched case-insensitively.

### Query Parameters

**Status:** `APPROVED`

- Query parameters are camelCase.
- A query parameter unknown to the route contract → `400`, `unknown_query_parameter`.
- Personal data and secrets are not passed in the query ([section 2](#data-in-urls)).

The semantics of field-level validation errors for query parameters are deferred ([section 15](#deferred-decisions)). JSON Pointer is not used for query parameters.

**Rationale:** strict parsing turns typos and attempts to pass extra fields (for example `role`) into an explicit rejection. The `application/json` requirement additionally closes cross-site requests with "simple" media types, but does not replace CSRF protection.

## 7. JSON Conventions, DTOs, Partial Updates

### JSON Format

**Status:** `APPROVED`

- Field names are camelCase.
- A successful response returns the resource without a universal envelope (`data`, etc.).
- Dates and times in responses are RFC 3339 in UTC (for example, `2026-11-13T10:00:00Z`).
- Identifiers in JSON are strings.
- Optional response fields are omitted by default. There is no global prohibition of `null`.
- Response media types: success with a body — `application/json`; error — `application/problem+json` ([section 8](#problem-details)).

### DTO

**Status:** `APPROVED`

- Request and response DTOs are separate types in the `httpapi` inbound adapter package ([Package Map §5](package-map.md#adapters)).
- Domain entities are not serialized directly.
- DTOs do not contain the password in responses or the password hash ([AR-009](architecture.md#8-architectural-invariants)).

### Partial Updates

**Status:** `APPROVED`

Partial updates are performed via `PATCH` with `Content-Type: application/json`. The model is called **field-level partial update**. This is not RFC 7396 JSON Merge Patch, and the `application/merge-patch+json` media type is not used.

| Field in the request | Action |
|---|---|
| Absent | Do not change |
| Has a value | Set the value |
| `null`, the field is explicitly allowed to be cleared | Clear |
| `null`, the field is not allowed to be cleared | `400`, `validation_failed` |

- There is no recursive merge for nested structures.
- Which fields can be cleared, as well as the contract of nested objects and collections, is defined by the Story.
- How a DTO distinguishes "field absent" from `null` is defined by Technical Design without new technologies.
- `PUT` is used only for full replacement of a resource or an approved idempotent state setting.

The composition of Profile fields is not defined ([Business Glossary — Profile](../product/business-glossary.md#profile)).

## 8. Errors

### Problem Details

**Status:** `APPROVED`

All error responses produced by the application have the RFC 9457 Problem Details format with `Content-Type: application/problem+json`. The limits of this guarantee are described in [section 10](#limits-of-http-response-guarantees).

Standard HTTP restrictions take precedence over the body presence rule:

- a response to `HEAD` contains no body regardless of the HTTP status, including an error status;
- a `204 No Content` response contains no body.

For the remaining error responses in which a body is allowed, Problem Details are mandatory.

| Field | Required | Content |
|---|---|---|
| `type` | yes | `about:blank` |
| `title` | yes | The standard HTTP status phrase per RFC 9110 in English. For example, for `413` — `Content Too Large` |
| `status` | yes | The HTTP response code |
| `detail` | no | An explanation in English, without localization |
| `code` | yes | A stable machine error code in snake_case |
| `requestId` | yes | The request's Request ID ([section 10](#request-id)) |
| `errors` | only for validation errors | An array of errors for individual fields |

Example:

```json
{
  "type": "about:blank",
  "title": "Bad Request",
  "status": 400,
  "detail": "Request validation failed",
  "code": "validation_failed",
  "requestId": "3f2a9c1e8b7d4a6f0e5c2b1a9d8e7f60",
  "errors": [
    { "field": "/email", "code": "invalid_format", "detail": "Invalid email format" }
  ]
}
```

- The client uses `code` for program logic. `detail` is not a machine code.
- `title`, `detail` and `errors` do not contain internal technical details: SQL, driver messages, stacktrace ([AR-008](architecture.md#8-architectural-invariants)).
- Errors do not contain entered secrets or other sensitive values, in particular the password ([AR-009](architecture.md#8-architectural-invariants)).
- Possible discrepancies between Go's `http.StatusText` and the RFC 9110 phrases are checked during implementation.

### Validation Errors

**Status:** `APPROVED`

An `errors[]` element:

| Field | Content |
|---|---|
| `field` | The path to the request body field in JSON Pointer notation (RFC 6901), for example `/email` |
| `code` | A stable code in snake_case |
| `detail` | An explanation in English, without sensitive values |

- `errors[]` is added only to body validation errors: mandatorily for `validation_failed`, and for `invalid_request_body` when the error concerns a specific field.
- Common element codes: `unknown_field`, `invalid_type`. They are not used as a top-level `code`.
- Element codes for the rules of specific fields (for example, `invalid_format` in the example) are defined by the Story.

### Error Categories and HTTP Codes

**Status:** `APPROVED`

The categories from [Architecture §7](architecture.md#error-handling) are translated into HTTP by the inbound adapter:

| Category | HTTP | Common `code` |
|---|---|---|
| Invalid data | `400` | `malformed_json`, `invalid_request_body`, `validation_failed`, `unknown_query_parameter` |
| Unauthenticated | `401` | `unauthenticated`, `invalid_credentials` |
| Forbidden | `403` | `forbidden`, `csrf_rejected` |
| Not found | `404` | `not_found` |
| Conflict | `409` | `conflict` |
| Temporarily unavailable | `503` | `service_unavailable` |
| Internal error | `500` | `internal_error` |
| Unknown outcome | `500` | `outcome_unknown` |

Transport and routing codes:

| Situation | HTTP | `code` |
|---|---|---|
| The route does not exist | `404` | `route_not_found` |
| The method is not supported by an existing route | `405` | `method_not_allowed` |
| The body exceeds the limit | `413` | `request_too_large` |
| Unsupported media type | `415` | `unsupported_media_type` |
| Request limit exceeded | `429` | `rate_limited` |

- `400` is used for all validation errors. Syntax errors and field errors are distinguished through `code`.
- `409` applies to the "conflict" category. Whether to return `409` for a duplicate email during registration depends on P5.
- `429` is reserved for future protection against automated guessing. The rules, limits, `Retry-After` and the external behavior of login are defined after P3 and must not disclose the existence of an account ([Security Conventions §7](security-conventions.md#protection-against-automated-guessing)).
- `503` is returned in the MVP without `Retry-After`.
- `504` is not used, because the service is not a gateway or proxy.
- A panic in a handler → `500`, `internal_error` ([Security Conventions §10](security-conventions.md#responses)).

### Catalog of Common Codes

**Status:** `APPROVED`

The full list of common top-level codes: `malformed_json`, `invalid_request_body`, `validation_failed`, `unknown_query_parameter`, `unauthenticated`, `invalid_credentials`, `forbidden`, `csrf_rejected`, `not_found`, `route_not_found`, `method_not_allowed`, `conflict`, `request_too_large`, `unsupported_media_type`, `rate_limited`, `internal_error`, `outcome_unknown`, `service_unavailable`.

- Common codes are kept only in this document. A new common code is added only through an update of this document.
- Codes for specific business errors (for example, a more precise conflict code) are defined by the Story in the approved HTTP contract.
- Codes are stable: changing the value of a code is a breaking change.

### 401 / 403 / 404 Order

**Status:** `APPROVED`

For existing protected endpoints with a supported HTTP method:

1. No valid session → `401`.
2. There is a session, insufficient permissions → `403`.
3. There are permissions, the resource does not exist → `404`.

- An admin use case checks the role before reading the target user ([Security Conventions §7](security-conventions.md#authorization-and-who-is-acting)). A non-administrator on `/api/v1/admin/...` receives `403`, not a masked `404`.
- The router may return `404 route_not_found` and `405 method_not_allowed` without an authentication check ([section 10](#router)).
- Self-service routes have no identifiers of other users, so the question of hiding the existence of a resource does not arise for them.

The `403` or `404` policy for future resources owned by users is deferred ([section 15](#deferred-decisions)).

### Dependency Failures and Unknown Outcome

**Status:** `APPROVED`

- Unavailability or timeout of Redis, PostgreSQL or the Argon2id queue → `503`, `service_unavailable`, if an HTTP response can still be produced.
- An unconfirmed PostgreSQL `COMMIT` → `500`, `outcome_unknown`. Such a response does not mean a confirmed failure and must not cause an automatic retry of the operation ([section 4](#4-http-methods-and-idempotency), [Security Conventions §10](security-conventions.md#postgresql)).
- After the client breaks the connection, a write timeout fires, or a response has been partially sent, it may be impossible to return Problem Details. This is not a violation of the conventions.

**Rationale:** RFC 9457 is a standard, so a custom format does not need to be specified. A stable `code` separates the client's program logic from the text, and a centralized catalog does not let Stories invent different names for the same situation.

## 9. Authentication and CSRF at the HTTP Level

The authentication, cookie and CSRF mechanism is defined in [Security Conventions](security-conventions.md). Only the HTTP contract is recorded here.

### Cookie-based authentication

**Status:** `APPROVED`

- The session is passed in cookies. The attributes of the access and refresh cookies are defined in [Security Conventions §5](security-conventions.md#cookies), the `Path` of the refresh cookie — in [section 3](#refresh).
- Each route is either explicitly public (registration, login, refresh) or goes through the session check ([Security Conventions §7](security-conventions.md#authorization-and-who-is-acting), [AR-006](architecture.md#8-architectural-invariants)).

### 401 Responses

**Status:** `APPROVED`

- Protected routes and refresh return a single code `unauthenticated`. Separate codes do not disclose whether the access JWT has expired, the session has been revoked, or the token is invalid.
- A failed login returns `401`, `invalid_credentials`. The response is the same for an unknown email, a wrong password and a blocked account ([Security Conventions §7](security-conventions.md#login)).
- If a future P5 decision requires changing the non-disclosure of the reasons for a failed login, this requires separate approval of changes to the Security Conventions.
- **`WWW-Authenticate` is not sent.** This is a deliberate deviation from RFC 9110 §15.5.2, which requires this header in a `401` response. There is no standard scheme for cookie-based authentication. A custom unregistered scheme (for example, `Cookie`) is not introduced. `Bearer` and `Basic` are not used, because they do not fit the model. The decision is revisited if a standard scheme appears.

### Cookies in Error Responses

**Status:** restrictions `APPROVED`, the full rule `DEFERRED`.

- A `503` response due to Redis unavailability does not clear cookies ([Security Conventions §10](security-conventions.md#redis)).
- A `401` response of a protected route does not automatically clear the refresh cookie.

Whether cookies are cleared in other `401` responses is defined by the Technical Design of US-002.

This document does not establish a client strategy for reacting to `401` (for example, "refresh, then login"). It depends on A2 and Client Design and must take into account refresh token rotation, reuse detection and concurrent refresh requests ([Security Conventions §4](security-conventions.md#refresh-rotation-and-reuse-detection)).

### CSRF and Origin Check

**Status:** `APPROVED`. The token delivery method — `OPEN`.

- The CSRF token is passed in the request header **`CSRF-Token`**.
- The rules for when the token is mandatory and the rules for checking `Origin` / `Sec-Fetch-Site` are defined by [Security Conventions §5](security-conventions.md#csrf).
- A failed CSRF check or origin check → `403`, `csrf_rejected`. The specific reason is visible only in security logs ([Security Conventions §9](security-conventions.md#9-security-logging)).

The classification of responses must be preserved regardless of the middleware order:

| Situation | Response |
|---|---|
| The session is absent or invalid | `401`, `unauthenticated` |
| Infrastructure error of the session check | `503`, `service_unavailable` |
| The session is valid, the CSRF check failed | `403`, `csrf_rejected` |
| A disallowed `Origin` for login or registration | `403`, `csrf_rejected` |

The middleware order is defined by Technical Design. The method of delivering the CSRF token to the client and the signed double-submit option remain open together with A2 ([section 15](#open-questions)).

## 10. Router, Caching, Request ID

### Router

**Status:** `APPROVED`

| Situation | Behavior |
|---|---|
| The route does not exist | `404`, `route_not_found`, Problem Details |
| The method is not supported by an existing route | `405`, `method_not_allowed`, Problem Details, the `Allow` header |
| A path with a trailing `/` or in a different case | No redirect. If the route does not exist → `404` |

Gin settings:

- `HandleMethodNotAllowed = true` and a `NoMethod` handler that produces Problem Details;
- a `NoRoute` handler that produces Problem Details;
- `RedirectTrailingSlash = false`;
- `RedirectFixedPath = false`.

The router may return `404` and `405` responses before the authentication middleware. The correctness of the `Allow` header and the response format is checked during Technical Design and Verification.

### Caching

**Status:** `APPROVED`

- All API responses in the MVP, both successful and error ones, have the `Cache-Control: no-store` header ([guarantee limits](#limits-of-http-response-guarantees)).
- Exceptions for public caching may be introduced only by separate approval.

### Request ID

**Status:** `APPROVED`

- The server generates a Request ID for each request: an opaque string with at least 128 bits of entropy from `crypto/rand`.
- An incoming Request ID from the client is not used as the server identifier.
- The Request ID is created by middleware in `internal/platform/httpserver` ([Package Map §4](package-map.md#4-platform)).
- The same Request ID is used at all processing points of one request.
- The Request ID is returned:
  - in the `Request-Id` response header on every response ([guarantee limits](#limits-of-http-response-guarantees));
  - in the `requestId` field in Problem Details.
- The Request ID is used as the `request id` field in structured logs ([Security Conventions §9](security-conventions.md#9-security-logging)).

**Request ID and `context.Context`.**

- The Actor is application information. Application receives it as an explicit use case parameter ([Security Conventions §7](security-conventions.md#authorization-and-who-is-acting)).
- The Request ID is technical metadata. It may be passed through `context.Context`; a separate use case parameter for it is not needed.
- Application extracts neither the Actor nor the Request ID from the context on its own. Technical metadata from `context.Context` is used by Platform and adapters, for example the security events port adapter.

W3C Trace Context is deferred ([section 15](#deferred-decisions)).

### Limits of HTTP Response Guarantees

**Status:** `APPROVED`

- The guarantees of `application/problem+json` ([section 8](#problem-details)), the `Request-Id` header and `Cache-Control: no-store` apply to HTTP responses that the application produces after Go `net/http` has passed the request to Gin.
- Low-level HTTP parsing errors, violation of the header size limit, some timeouts and network breaks may be handled before Gin or without the possibility of producing a standard response. Such responses (or their absence) do not violate these conventions.
- No custom HTTP parser or changes to `net/http` are created.
- For responses produced by Gin or the application, all requirements of this document are mandatory.

**Rationale:** Gin's default settings (a text `404`, `404` instead of `405`, path redirects) contradict the error format and the naming rules. A server-side Request ID does not let the client forge correlation in logs.

## 11. HTTP Server Timeout and Limit Model

**Status:** `APPROVED` — the model. Values — `DEFERRED` until Infrastructure Bootstrap.

The HTTP server (`internal/platform/httpserver`) has the following parameters:

| Parameter | What it limits |
|---|---|
| Read header timeout | Time to read the request headers |
| Read timeout | Time to read the whole request |
| Write timeout | Time to produce and write the response |
| Idle timeout | Idle time of a keep-alive connection |
| Shutdown timeout | Time to complete active requests during graceful shutdown |
| Header size limit | The size of the request headers |
| Body size limit | The size of the request body ([section 6](#body-size)) |

- All parameters are set by configuration and have finite values.
- A configuration in which a parameter is absent, zero or unlimited is invalid. The application with such a configuration does not start. In Go, a zero HTTP server timeout means no limit, so without this rule unlimited waits would remain.
- Timeouts of calls to Redis and PostgreSQL and of the Argon2id queue do not belong to this model ([Security Conventions §10](security-conventions.md#timeouts)).

**Rationale:** agents at Technical Design need a model, and specific numbers without load data would be an invention.

## 12. Collections

**Status:** invariants `APPROVED`. The pagination, sorting and filtering scheme — `DEFERRED`.

There are no routes returning collections in the MVP. For any potentially unbounded collection, the following invariants apply:

- the server always applies a maximum page size, even if the client did not specify one;
- the order of results is deterministic and has a stable tie-breaker (for example, an identifier);
- query parameters are camelCase;
- personal data and secrets are not passed in query parameters ([section 2](#data-in-urls)).

The pagination scheme (offset or cursor), the sorting and filtering format and page metadata are defined by the first Story with a collection. The decision goes through separate approval and is then added to this document as a general rule. The agent does not establish it on its own.

## 13. OpenAPI

**Status:** `APPROVED`

- The API contract is described by an **OpenAPI 3.1** specification maintained manually.
- Location: `docs/api/openapi.yaml`. The file is created by the first Story that adds an HTTP endpoint.
- Shared components are described once: the Problem Details schema, the `errors[]` element, the `CSRF-Token` header, cookie authentication, the `Request-Id` header. Routes reference these components.

**Roles in the SDLC:**

| Stage | What it does with the contract |
|---|---|
| Story Technical Design | Defines the approved changes to the HTTP contract |
| Implementation | Updates OpenAPI in the same change as the corresponding handler |
| Verification | Checks the consistency of OpenAPI, Technical Design and the implementation |

- OpenAPI reflects the accumulated contract of the implemented API.
- If OpenAPI, Technical Design and the implementation contradict each other, the agent reports the contradiction and does not choose the source of truth on its own.
- New Go dependencies are not added.

Swagger UI, code generation and automatic specification validation are deferred ([section 15](#deferred-decisions)).

**Rationale:** a manually written specification gives agents a machine-readable contract without new technologies ([AR-012](architecture.md#8-architectural-invariants)). Generators either add a new dependency or dictate the form of handlers.

## 14. Mapping to AR / BR / Security Conventions

| Source | Sections of this document |
|---|---|
| [AR-003](architecture.md#8-architectural-invariants) | 7 (DTOs in `httpapi`), 8 (the inbound adapter translates the category into HTTP) |
| [AR-006](architecture.md#8-architectural-invariants) | 9 (explicit public routes) |
| [AR-007](architecture.md#8-architectural-invariants), [BR-001](../product/business-rules.md#br-001--admin-only-capabilities), [BR-002](../product/business-rules.md#br-002--own-account-self-service) | 3 (`/me` without an identifier, `/admin` as a naming convention), 8 (the 401/403/404 order), 10 (Actor as an explicit parameter) |
| [AR-008](architecture.md#8-architectural-invariants) | 8 (no technical details in Problem Details) |
| [AR-009](architecture.md#8-architectural-invariants), [NFR-002](../product/non-functional-requirements.md#nfr-002--password-confidentiality) | 7 (DTOs without the password and hash), 8 (no sensitive values in errors) |
| [AR-012](architecture.md#8-architectural-invariants) | 6 (stdlib for JSON), 13 (OpenAPI without new dependencies) |
| [BR-003](../product/business-rules.md#br-003--unique-email) | 8 (`409` — depending on P5) |
| [BR-007](../product/business-rules.md#br-007--blocked-user-access) | 8 (`outcome_unknown` without automatic retry), 9 (`invalid_credentials` for a blocked account) |
| [Security Conventions §5](security-conventions.md#5-cookies-csrf-cors-https) | 3 (`Path` of the refresh cookie), 6 (`application/json`), 9 (`CSRF-Token`, `csrf_rejected`) |
| [Security Conventions §7](security-conventions.md#7-authentication-and-authorization) | 8 (`401`, `403`, `404`), 9 (`invalid_credentials`) |
| [Security Conventions §9](security-conventions.md#9-security-logging) | 2 (tokens and personal data not in URLs), 10 (Request ID in logs) |
| [Security Conventions §10](security-conventions.md#10-failures-and-timeouts) | 4 (no automatic retries), 8 (`503`, `outcome_unknown`, panic), 9 (cookies on `503`) |

**NFR-001.** [NFR-001](../product/non-functional-requirements.md#nfr-001--secure-authentication) has the `NEEDS_CLARIFICATION` status. This document does not close it and does not establish verification criteria.

**Closed deferred decisions.** This document closes the following deferred decisions:

- from [Architecture §9](architecture.md#deferred-decisions): "List of error categories, their mapping to HTTP codes, error body format"; the HTTP server timeout model and graceful shutdown;
- from [Package Map §9](package-map.md#deferred-decisions): "URL prefixes and route versioning"; the HTTP server timeout model and graceful shutdown;
- from [Security Conventions §12](security-conventions.md#infrastructure-and-architectural-dependencies): "Error categories, HTTP codes, refresh path for `Path`, 404 or 403".

The corresponding rows in those documents are marked as defined with references to this document.

## 15. Open Questions, Deferred Decisions and Dependencies

### Open Questions

**Status:** `OPEN`. The agent does not resolve them without separate approval.

| Question | Type | What depends on it |
|---|---|---|
| P1 — Logout ([Security Conventions §12](security-conventions.md#product-questions)) | Product | The logout route in `/api/v1/auth` |
| P3 — protection against automated guessing | Product | The rules of `429`, `rate_limited`, `Retry-After`, the external behavior of login |
| P4 — password policy | Product | Password validation rules |
| P5 — account enumeration during registration; what a blocked user sees during login | Product | `409` for a duplicate email (US-001). Changing the non-disclosure of the reasons for a failed login is possible only through approval of the Security Conventions |
| P6 — Administrator Provisioning ([Epic Map](../product/epic-map.md#open-product-dependencies)) | Product | US-006, US-007 |
| Target User Identification ([Epic Map](../product/epic-map.md#open-product-dependencies)) | Product | The form of the admin routes of US-006/007, a possible user search |
| Profile composition | Product | `/me` subresources, fields that can be cleared |
| A2 — API client type, CSRF token delivery method, signed double-submit ([Security Conventions §12](security-conventions.md#infrastructure-and-architectural-dependencies)) | Architectural | How the client obtains the `CSRF-Token` value; the client strategy for reacting to `401` |

### Deferred Decisions

**Status:** `DEFERRED`. Until the stated point, the agent does not rely on any option.

| Decision | Deferred to |
|---|---|
| Values of timeouts, the body limit, the header limit, the shutdown timeout | Infrastructure Bootstrap |
| Form and idempotency of Block / Unblock | Clarification and Technical Design of US-006/007 |
| Clearing cookies in `401` responses (beyond the restrictions from [section 9](#cookies-in-error-responses)) | Technical Design of US-002 |
| The physical identifier format of future modules' entities. For Account/User **defined:** UUID v4 ([Persistence Conventions §6](persistence-conventions.md#identifiers)) | The corresponding module |
| Simultaneous support of several major API versions | When the need appears |
| Overriding the body limit for individual routes | When the need appears |
| Pagination scheme, sorting and filtering format, page metadata | The first Story with a collection, through separate approval |
| Semantics of field-level validation errors of query parameters | The first Story that needs it |
| `403` or `404` for resources owned by users | The first module with such resources |
| Retry policy for read-only operations | A separate decision |
| Exceptions to `Cache-Control: no-store` | Separate approval |
| W3C Trace Context | When tracing appears |
| Swagger UI, code generation, automatic OpenAPI validation | Engineering Tooling setup |
| `WWW-Authenticate` for `401` | Revisit if a standard scheme for cookie-based authentication appears |

### Dependencies on Security Conventions

| Dependency | Status in Security Conventions | Impact on the API |
|---|---|---|
| CSRF token delivery method, A2 | `OPEN` / `DEFERRED` | The client obtaining the `CSRF-Token` value |
| Login CSRF protection | `OPEN` | Rules for login and registration before a public browser client appears |
| P3 and the corresponding part of NFR-001 | `OPEN` | `429` for login |
| Idempotency of access-state operations | `DEFERRED` until Clarification of US-006 | Form of Block / Unblock, retry rules |
| Argon2id queue limits and timeouts | `DEFERRED` until implementation | When `503` is returned for login and password change |

### Dependencies on Persistence Conventions

| Dependency | Impact on the API | Status |
|---|---|---|
| The per-user serialization mechanism and PostgreSQL and Redis timeouts | When `503` and `500 outcome_unknown` are returned | **Defined:** the mechanism and the timeout model — [Persistence Conventions §5](persistence-conventions.md#5-per-user-serialization-row-locks-timeouts). Timeout values — `DEFERRED` until Infrastructure Bootstrap |
| Recognizing an unconfirmed `COMMIT` | Correct distinction between `500 internal_error` and `500 outcome_unknown` | **Defined:** [Persistence Conventions §8](persistence-conventions.md#unknown-outcome-commit) |
| Identifier format in storage | Identifier format in URLs and JSON (in JSON they are always strings) | **Defined** for Account/User: UUID v4 ([Persistence Conventions §6](persistence-conventions.md#identifiers)). For future modules — `DEFERRED` |
