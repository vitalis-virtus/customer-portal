# Testing Strategy — Customer Portal

**Document status:** `DRAFT` — created to close FG-01 ([workflow-policies.md §15](../workflow/workflow-policies.md#15-prerequisites-and-foundation-gaps)). Every decision below has the status `PROPOSED` until a human approves this document by separate work outside the Story Workflow. Until then FG-01 is not closed for real Stories, and agents may rely on it only in `mode=pilot`.

## 1. Purpose and Scope

This document is the project-wide Testing Strategy (`testing-strategy` source, [artifact-registry.yaml](../contracts/artifact-registry.yaml)), the first of the three sources of mandatory checks ([workflow-policies.md §18](../workflow/workflow-policies.md#18-testing-requirements-od-15), OD-15). It defines test kinds, tools, scope, evidence and execution rules for all Stories.

It details, and does not change, [Architecture](architecture.md), [Package Map §8](package-map.md#8-tests-migrations-go-version) (test placement), [Persistence Conventions §11](persistence-conventions.md#11-persistence-testing) (mandatory persistence scenarios), [Security Conventions](security-conventions.md) and [API Conventions](api-conventions.md). It cannot weaken any of their rules.

**Out of scope:** test placement (Package Map §8); Story-specific checks (the Testing Approach); CI pipelines, deployment and Redis failover validation (Deployment Verification, not defined yet); timeout values and images (Infrastructure Bootstrap).

### Three levels of responsibility

| Level | Owner | Defines |
|---|---|---|
| **Testing Strategy** (this document) | Project Foundation | Which test kinds exist, tools, mandatory checks per risk category, evidence format, execution rules |
| **Testing Approach** | `implementation-plan`, IMPLEMENTATION_PLANNING; checked by PLAN_REVIEW; approved at HUMAN_TECHNICAL_APPROVAL | For each Acceptance Criterion and each touched risk category of the Story Risk Assessment — the concrete checks, their level, Test-First where appropriate (Q3) |
| **Verification execution** | VERIFICATION (and Post-Merge Validation in INTEGRATION_CONFIRMATION) | Runs the checks of all three sources in a prepared workspace, records state and evidence per check |

The Testing Approach may add checks, never remove the checks this document makes mandatory.

## 2. Current State (facts)

**Status:** `PROPOSED`

- Go module `github.com/vitalis-virtus/customer-portal`, `go 1.25.0` (`go.mod`); `main.go` is an empty `main`. There is no application code and **no tests** yet.
- A local PostgreSQL 18 test database and golang-migrate runs are **configured** ([Infrastructure Bootstrap](infrastructure-bootstrap.md), `compose.test.yaml`), but **not yet started or verified**. Redis test infrastructure is not configured.
- There is no CI pipeline, no Makefile and no linter configuration; no application tests and no integration tests exist, and none has been executed.
- Therefore no test result exists. This document defines rules, it does not record any execution.

## 3. Principles

**Status:** `PROPOSED`

1. **Evidence over claims.** A check counts only with the actual command, its exit status and relevant output, bound to the checked commit.
2. **Lowest sufficient level.** Business rules and access rules are tested in Domain and Application without infrastructure (Architecture: the core is testable without HTTP and stores). Real stores are used only where the mechanism is in the store (transactions, locks, Redis atomicity).
3. **Proportionality.** Tests are added with the mechanism they check ([Persistence Conventions §11](persistence-conventions.md#general-rules)); no coverage percentage target.
4. **Determinism.** No `time.Sleep` to provoke races, no dependence on wall clock or test order; time and randomness are injected through ports.
5. **No new technologies.** Only the Go toolchain and the standard library (`testing`, `net/http/httptest`) are used. A test library (assertions, mocks, testcontainers) is added only by a separate approved decision ([AR-012](architecture.md#8-architectural-invariants)).
6. **Unavailable is not passed.** A check that cannot be run is reported `blocked` or `not_executed`, never `passed`.

## 4. Test Kinds

**Status:** `PROPOSED`

| Kind | Scope | Where (Package Map §8) | Tooling | Run by |
|---|---|---|---|---|
| Unit | Domain, Application use cases with test doubles of ports, adapter logic without I/O (for example error classification) | `_test.go` next to the code, external test package by default | `go test` | `go test ./...` |
| HTTP (API) | Inbound adapter `adapters/inbound/httpapi`: routing, status codes, error body, cookies, CSRF, request limits, JSON decoding | `_test.go` next to the adapter | `net/http/httptest` with the Gin engine and Application doubles | `go test ./...` |
| Integration | Outbound adapters with real PostgreSQL / Redis, migrations, AR-011 concurrent scenarios | next to the adapter, `//go:build integration` | `go test -tags=integration` | see section 9 |
| Static | Compilation and vet of all packages | — | `go build ./...`, `go vet ./...` | always |

**Test doubles** are written manually in `_test.go` of the consumer package (no generators).

End-to-end tests against a running service are not defined; they need Infrastructure Bootstrap.

## 5. Mandatory Checks by Area

**Status:** `PROPOSED`

### Every Story with code changes

- `gofmt -l .` prints nothing; `go build ./...`, `go vet ./...`, `go test ./...` pass.
- `go test -race ./...` for Stories touching concurrency (requires cgo; see section 9).

### API and HTTP behavior

For each new or changed endpoint, HTTP tests check: the success code and body; each documented error category with its code and body format ([API Conventions §8](api-conventions.md#8-errors)); request validation (Content-Type, body size, unknown fields, malformed JSON); idempotency rules where defined.

### Persistence

- The mandatory scenarios of [Persistence Conventions §11](persistence-conventions.md#mandatory-scenarios) become mandatory when their mechanism is implemented. They are integration tests, except those marked Unit there.
- A migration Story checks `up` from an empty database to the latest version.

### Authentication and authorization

- Access rules are unit-tested in Application: allowed and denied actor for each operation; a blocked user is denied (BR-007).
- HTTP tests: a protected route without, with an invalid, expired or revoked session returns the documented error; cookie attributes and CSRF behavior per [Security Conventions §5](security-conventions.md#5-cookies-csrf-cors-https).
- Session, revocation and refresh rotation rules per [Security Conventions §4](security-conventions.md#4-sessions-in-redis-and-revocation) are covered by unit tests plus the Redis integration scenarios of Persistence Conventions §11.

### Security baseline (VERIFICATION basic security checks)

For changed code: no secrets in code or test fixtures; prohibited data is not logged ([Security Conventions §9](security-conventions.md#9-security-logging)) — checked by a test or by review evidence; passwords are hashed only through the approved hashing port ([§6](security-conventions.md#6-passwords)); errors do not leak internals in HTTP responses. Dependency vulnerability scanning (`govulncheck`) is not installed and is not mandatory until added by a separate decision.

## 6. Risk Categories → Additional Checks

**Status:** `PROPOSED`

Maps the categories of [workflow-policies.md §18](../workflow/workflow-policies.md#18-testing-requirements-od-15) used in the Story Risk Assessment.

| Risk category | Mandatory additional checks |
|---|---|
| Authentication, authorization | The tests of section 5 "Authentication and authorization" for every affected operation, including negative cases |
| Persistence | Integration tests with the real store for every affected mechanism (section 5 "Persistence") |
| Concurrency | Deterministic concurrent integration tests with explicit synchronization points, both orders where BR-007 depends on it; `go test -race` on affected packages |
| Infrastructure | Configuration validation as far as possible without a deployment: config loading tests, startup failure on missing required config. Deployment validation is deferred (Deployment Verification) |

## 7. Acceptance Criteria Verification

**Status:** `PROPOSED`

- Each Acceptance Criterion has at least one check in the Testing Approach, derived from its Verification Method in `story-spec`.
- Preferred method: an automated test (unit or HTTP) named or commented with the `AC` identifier.
- Allowed only when automation is not reasonable: a static check (inspection of code or configuration) with an explicit reason in the Testing Approach.
- Manual checks against a running service are not available until Infrastructure Bootstrap.

## 8. Evidence and Reporting

**Status:** `PROPOSED`

Each executed check in `verification-report` (and `post-merge-validation-record`) records: the check identifier and its source (Testing Strategy / Story Risk Assessment / Testing Approach, `AC`); the exact command; the workspace commit; the state `passed` / `failed` / `not_executed` / `blocked`; the exit status and the relevant output excerpt (failing tests in full). `go test` output with `-v` or `-json` for the affected packages is sufficient evidence; summaries without output are not.

## 9. Local Execution and Environment Constraints

**Status:** `PROPOSED`

- Checks run locally in the workspace prepared for VERIFICATION or Post-Merge Validation, with the Go toolchain matching `go.mod`. There is no CI; the local run is the evidence.
- **Integration test infrastructure — PostgreSQL configured, not yet verified.** [Infrastructure Bootstrap](infrastructure-bootstrap.md) defines a local PostgreSQL 18 test database (`compose.test.yaml`), migration runs with the golang-migrate CLI and the `TEST_DATABASE_URL` variable for integration tests; tests must not hard-code credentials. Configured is not the same as started or tested: an integration check is `passed` only with the actual run evidence (section 8); if the database cannot be started in the workspace, the check is `blocked`. Redis test infrastructure is not configured: Redis integration checks are `blocked` until it is added.
- `-race` requires cgo and a C compiler; if it is unavailable in the workspace, the check is `blocked` with that reason.
- Tests must not need network access other than the test stores, and must clean up the data they create.

## 10. Open Items

| Item | Deferred to |
|---|---|
| Redis for integration tests; verification of the PostgreSQL test setup by a first real run | Infrastructure Bootstrap |
| CI pipeline | Separate decision |
| Network failure during `COMMIT`, Redis recovery / failover tests | Deployment Verification ([Persistence Conventions §11](persistence-conventions.md#mandatory-scenarios)) |
| Test libraries (assertions, mocks, testcontainers), `govulncheck`, linters | Separate approved decision |
| End-to-end tests against a running service | After Infrastructure Bootstrap |
