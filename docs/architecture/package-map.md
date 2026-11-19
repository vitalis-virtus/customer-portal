# Package Map — Customer Portal

## 1. Purpose

This document defines the physical structure of the Customer Portal Go project: directories, packages, their responsibilities and the allowed import directions. It maps the logical architecture from [Architecture](architecture.md) onto Go packages and does not introduce new architectural decisions.

**In scope:**

- the directory tree and the package naming rule;
- placement of the composition root, Platform, module layers, ports and the module's public contract;
- allowed import directions and how they are checked;
- physical placement of tests, test doubles and migrations.

**Out of scope:**

- runtime parameters: timeouts, graceful shutdown parameters, PostgreSQL and Redis versions;
- API, Security and Persistence conventions;
- testing strategy, including the choice of test libraries and test infrastructure;
- Go code.

**Decision statuses** are the same as in [Architecture](architecture.md#1-purpose): `APPROVED` means the decision is explicitly approved. Anything not approved is recorded in the section [Known Consequences and Deferred Decisions](#9-known-consequences-and-deferred-decisions).

**Mandatory names.** The directory and package names given in this document are mandatory. Names of types, methods and files marked with the word "for example" are illustrative.

**Relationship to other documents.** Terms are used in the meanings from the [Business Glossary](../product/business-glossary.md). If this document contradicts [Architecture](architecture.md) or the product documents, the agent must report the contradiction. The agent must not automatically prefer one of the documents.

Decisions approved on 2026-11-13.

## 2. Project Tree

**Status:** `APPROVED`

```
customer-portal/
├── go.mod                              # module path; the go directive defines the Go version
├── cmd/
│   └── customer-portal/                # package main: entry point and composition root
├── migrations/                         # golang-migrate SQL migrations
└── internal/
    ├── platform/                       # shared technical infrastructure
    │   ├── config/
    │   ├── logging/
    │   ├── httpserver/
    │   ├── postgres/
    │   └── redisclient/
    └── modules/
        └── account/                    # package account: public contract and module assembly
            └── internal/               # the module's private implementation
                ├── domain/
                ├── application/
                └── adapters/
                    ├── inbound/
                    │   └── httpapi/
                    └── outbound/
                        ├── postgres/
                        ├── redisstore/
                        └── passwordhash/
```

| Directory | Responsibility |
|---|---|
| `cmd/customer-portal/` | Entry point and composition root, see [section 3](#3-entry-point-and-composition-root) |
| `internal/` | All application code. There is no `pkg/` directory: the service does not provide code for external import |
| `internal/platform/` | Platform, see [section 4](#4-platform) |
| `internal/modules/<module>/` | A module, see [section 5](#5-module-structure) and [section 6](#6-module-public-contract) |
| `migrations/` | DB schema migrations, see [section 8](#8-tests-migrations-go-version) |

The tree shows the target structure. A package or directory is created when an implementation that needs it appears. There is no need to create empty packages just to make the structure match the tree.

Future modules are created in `internal/modules/<module>/` following the same template as `account`.

**Rationale:** `cmd/` + `internal/` is the standard structure of a Go service. The compiler's `internal/` rule prohibits importing application code from outside. A separate `internal/modules/` prefix shows the module boundary directly in the path, so import rules can be formulated through prefixes.

### Package Naming Rule

**Status:** `APPROVED`

A package name does not match the name of a library that the package imports.

| Place | Package name | Note |
|---|---|---|
| Inbound HTTP adapter | `httpapi` | Not `http`, because it matches `net/http` |
| Outbound PostgreSQL adapter | `postgres` | pgx is imported as `pgx` / `pgxpool`, there is no collision |
| Outbound Redis adapter | `redisstore` | Not `redis`, because it matches the Redis client |
| Outbound hashing adapter | `passwordhash` | Named by role: the algorithm is deferred to the Security conventions |
| Platform: PostgreSQL | `postgres` | There is no collision |
| Platform: Redis | `redisclient` | Not `redis` |

**Rationale:** files that import both a project package and a library of the same name do not need aliases. Agents do not have to invent names in every User Story.

## 3. Entry Point and Composition Root

**Status:** `APPROVED`

The composition root lies entirely in `cmd/customer-portal/` (package `main`). It may be split into several files. This is the single composition root at the entry point that [Architecture §5](architecture.md#composition-root) refers to.

Responsibilities, in this order:

1. load the configuration (`platform/config`);
2. initialize logging (`platform/logging`);
3. open connections to PostgreSQL and Redis (`platform/postgres`, `platform/redisclient`);
4. call the module assembly functions, passing them Platform dependencies and, if needed, public types of other modules;
5. create the HTTP server (`platform/httpserver`), create route groups and pass them to the modules for route registration;
6. start the server and stop it correctly (graceful shutdown).

The composition root imports only module root packages and Platform packages. It does not see the modules' internal packages: the compiler guarantees this, see [section 5](#module-placement-and-private-implementation).

This document does not define timeout values and shutdown parameters, see [section 9](#deferred-decisions).

**Rationale:** the placement corresponds to [Architecture §5](architecture.md#composition-root) literally. A module assembles its internal parts itself, so the composition root connects only ready-made modules.

## 4. Platform

**Status:** `APPROVED`

Platform is divided into separate packages by responsibility:

| Package | Responsibility |
|---|---|
| `internal/platform/config` | Configuration loading |
| `internal/platform/logging` | Logging |
| `internal/platform/httpserver` | Basic HTTP server infrastructure (Gin) |
| `internal/platform/postgres` | Creating and closing the PostgreSQL connection pool |
| `internal/platform/redisclient` | Creating and closing the Redis client |

- `platform/postgres` and `platform/redisclient` only create and close connections. Repositories and session stores do not belong here: they are module outbound adapters.
- Platform packages do not import module packages and do not contain business logic ([AR-005](architecture.md#8-architectural-invariants)).
- Logic related to sessions, cookies and "who is acting" belongs to the `account` module, not to Platform. See [section 6](#session-check-and-who-is-acting).

**Rationale:** separate packages do not force a module that needs only one connection to depend transitively on the whole infrastructure. The boundary "shared technical things live in Platform" is checked at the level of each package.

## 5. Module Structure

### Module Placement and Private Implementation

**Status:** `APPROVED`

- A module lies in `internal/modules/<module>/`.
- The module's private implementation lies in the nested `internal/modules/<module>/internal/`. The compiler does not allow importing these packages outside the `internal/modules/<module>/` tree. This restriction applies both to the composition root and to other modules.
- Outside the nested `internal/`, there is only the module's root package with the public contract ([section 6](#6-module-public-contract)).

**Rationale:** importing another module's private packages becomes a compilation error without additional tools. The composition root also does not see the module's internal parts, as [Architecture §5](architecture.md#composition-root) requires. What the compiler does not guarantee is described in [section 7](#limits-of-compiler-guarantees).

### Layers as Separate Packages

**Status:** `APPROVED`

| Package | Component from [Architecture §4](architecture.md#hexagonal-architecture) | Contents |
|---|---|---|
| `internal/domain` | Domain | Entities, value objects, business invariants |
| `internal/application` | Application | Use cases and outbound ports |
| `internal/adapters/inbound/<tech>` | Inbound adapters | HTTP: handlers, session check middleware |
| `internal/adapters/outbound/<tech>` | Outbound adapters | Outbound port implementations |

(Paths are given relative to `internal/modules/<module>/`.)

- `domain` and `application` are each one package per module.
- A convention, not a requirement: one file per use case, for example `register_account.go`, `block_user.go`.

**Rationale:** separate packages make it possible to check dependency rules by imports. The `account` module covers one small subject area, so splitting the core into several packages would create inter-package dependencies without benefit.

### Adapters

**Status:** `APPROVED`

Adapters are grouped by direction: `adapters/inbound/<tech>/` and `adapters/outbound/<tech>/`. Package names are set by the [naming rule](#package-naming-rule). For the `account` module:

| Package | Responsibility |
|---|---|
| `adapters/inbound/httpapi` | HTTP handlers (Gin), session check middleware |
| `adapters/outbound/postgres` | The module's data store in PostgreSQL (pgx), implementation of the transaction port |
| `adapters/outbound/redisstore` | Session store in Redis |
| `adapters/outbound/passwordhash` | Password hashing |

**Rationale:** the grouping reproduces the terms from [Architecture §4](architecture.md#hexagonal-architecture). The rules for inbound and outbound adapters differ (AR-003 and AR-008), and now it is visible which rule applies to which package.

### Ports

**Status:** `APPROVED`

- **Outbound ports** are interfaces declared in the `application` package. Convention: a separate file, for example `ports.go`. Outbound adapters implement these interfaces.
- **Inbound ports** are not declared as separate interfaces in Application. Application provides concrete use case types. A handler depends on a narrow interface declared in the inbound adapter package itself, that is, on the consumer side.

One rule applies at all boundaries: the interface is declared by whoever consumes it.

**Rationale:** the placement of outbound ports directly embodies [Architecture §4](architecture.md#hexagonal-architecture): ports are defined in Application. Consumer-side interfaces are a Go idiom: a handler can be tested with a test double, and Application does not accumulate abstractions that nothing uses.

## 6. Module Public Contract

### Module Root Package

**Status:** `APPROVED`

The module's public contract lies in the root package `internal/modules/<module>/`. For `account`, this is package `account`. It contains:

- **the assembly function.** It accepts Platform dependencies and returns the module's concrete type, for example `*account.Module`;
- **the public methods** of the module type. Together they are the module's public interface from [Architecture §5](architecture.md#inter-module-interaction). There is no separate Go interface for the whole module. A consumer module declares a narrow interface for what it needs itself;
- **the public types** that appear in the public methods, for example `account.Actor`.

The root package may import the module's own internal packages, Platform and Gin, because its role is module assembly. It does not contain business logic: use cases stay in `application`.

**Rationale:** the module's contract and assembly lie in one place visible from outside. The "concrete type + consumer-side interface" form applies the same rule as the [ports](#ports).

### HTTP Routes

**Status:** `APPROVED`

The module type has a route registration method: `RegisterRoutes(r gin.IRouter)`. The composition root creates a route group and passes it to the module. The module registers its routes but does not decide under which prefix they will be. URL prefixes and versioning are defined by the API conventions.

**Rationale:** Platform is not tied to the format of the modules' routes, and shared Gin middleware remains available.

### Session Check and "Who Is Acting"

**Status:** `APPROVED`

- The session check middleware lies in `account/internal/adapters/inbound/httpapi`. It calls the session validation use case from `application` ([AR-006](architecture.md#8-architectural-invariants)).
- The module type exposes this middleware through a public method, for example `RequireSession() gin.HandlerFunc`. The composition root and other modules' adapters apply it to protected routes.
- The middleware puts the check result ("who is acting") into the request context. The handler retrieves it and passes it to the use case **as an explicit parameter**. Application reads nothing from the request context ([AR-007](architecture.md#8-architectural-invariants)).
- For the `account` core, the "who is acting" type is defined inside the module. The public type in the root package (for example `account.Actor`) and the function that reads it from the request context are intended for other modules' adapters.

The context key and the claims format are defined by the Security conventions.

**Rationale:** all session logic stays in the `account` module, and Platform stays purely technical. The use case signature shows that it depends on who is acting, so the permission check is hard to skip.

### Organizing Public Types Without Import Cycles

**Status:** `APPROVED`

Dependencies inside a module point only from the root package to the internal ones:

```
account  ──►  account/internal/adapters/...  ──►  account/internal/application  ──►  account/internal/domain
   │                                                        ▲
   └────────────────────────────────────────────────────────┘
```

- The module's root package may import its own internal packages.
- The module's internal packages **do not import** its root package. In particular, `account/internal/adapters/...` does not import `account`.
- Public types are defined only in the root package. Internal packages work with their own types (`domain`, `application`, `httpapi`).
- Translation between internal and public types is performed by the root package. Example: the function that reads "who is acting" from the context obtains the internal value through an exported `httpapi` function and converts it into `account.Actor`.
- Public methods of the module type accept and return public types and translate internal types themselves.

**Rationale:** the import graph inside the module becomes acyclic, because the root package only consumes internal packages. Internal types do not get into the public contract, so the internal implementation can be changed without touching consumers.

### Who May Import a Module's Root Package

**Status:** `APPROVED`

- A module's root package is imported only by the composition root and by adapters of **other** modules.
- A module's Domain and Application do not import any other module.
- An inbound adapter translates another module's data (for example, `account.Actor`) into its own module's types before passing them to a use case.
- A module's root package does not import other modules' root packages. The composition root passes a dependency on another module through a parameter of the assembly function. The parameter type is a narrow interface declared by the consumer module.

**Rationale:** the root package depends transitively on Gin, pgx and the Redis client. If another module's Domain or Application imported it, these technologies would become their dependency, which contradicts the intent of [Architecture §4](architecture.md#hexagonal-architecture). The restriction for module roots reduces the risk of cycles between modules. The residual risk is described in [section 9](#known-consequences).

## 7. Allowed Import Directions

### Import Table

**Status:** `APPROVED`

"Own" means packages of the same module. "Other module roots" means the root packages `internal/modules/<other>`.

| Package | May import | May not import |
|---|---|---|
| `domain` | stdlib, if the package does not tie to infrastructure | Any other project packages, frameworks, drivers, the Redis client, infrastructure stdlib packages (in particular `net/http`, `database/sql`) |
| `application` | Own `domain`, stdlib, if the package does not tie to infrastructure | Adapters, Platform, any modules (its own root as well), Gin, pgx, the Redis client, infrastructure stdlib packages (in particular `net/http`, `database/sql`) |
| `adapters/inbound/*` | Own `application` and `domain`, Platform, Gin, other module roots | Own module root, own module's outbound adapters |
| `adapters/outbound/*` | Own `application` and `domain`, Platform, pgx, the Redis client, other module roots | Own module root, own module's inbound adapters |
| Module root | Own `internal/...`, Platform, Gin | Other module roots |
| `platform/*` | stdlib, Gin, pgx, the Redis client, other `platform/*` packages | Any module packages |
| `cmd/customer-portal` | Module roots, Platform | Modules' internal packages |

Consequences of the table:

- Application does not import Platform. If a use case needs to log, this is done through an outbound port or in adapters, not through `platform/logging` directly.
- Inbound and outbound adapters of the same module do not import each other.
- The set of external libraries is limited by [Architecture §6](architecture.md#6-technology-stack) and [AR-012](architecture.md#8-architectural-invariants).

**Rationale:** the table carries the "inward only" dependency rule from [Architecture §4](architecture.md#hexagonal-architecture) and the Platform boundaries from [Architecture §5](architecture.md#platform) over to specific packages. A violation is visible from the `import` block.

### Limits of Compiler Guarantees

**Status:** `APPROVED`

| What is guaranteed | How |
|---|---|
| Packages outside the `internal/modules/<module>/` tree cannot import private packages from its nested `internal/` (part of AR-004). The nested `internal/` does not restrict what the private packages themselves import | The compiler, the nested `internal/` |
| There are no import cycles | The compiler: a cycle means a build error |
| The rest of the import table | Design Review and Implementation Verification |
| A module's DB queries do not access another module's tables (part of AR-004) | Design Review and Implementation Verification |
| The substance of the rules: no business logic in adapters and Platform, Application's external calls only through ports (AR-002, AR-003, AR-005) | Design Review and Implementation Verification |

The package structure makes the dependency rules checkable by imports, but Go itself does not guarantee them. Automatic checking of the import table is a separate future task. The tools for it are chosen by a separate decision when Engineering Tooling is set up.

**Rationale:** the document explicitly shows where automatic protection ends, so that at review no one assumes the compiler checks more than it actually does.

### Mapping to Architecture Rules

| Rule | How it is expressed in the Package Map |
|---|---|
| [AR-001](architecture.md#8-architectural-invariants) | The `domain` row of the import table: only stdlib without infrastructure packages |
| [AR-002](architecture.md#8-architectural-invariants) | The `application` row: without Gin, pgx, the Redis client, infrastructure stdlib packages, Platform and other modules; outbound ports are declared in `application` |
| [AR-003](architecture.md#8-architectural-invariants) | HTTP logic only in `adapters/inbound/httpapi`. The content of handlers is checked at review |
| [AR-004](architecture.md#8-architectural-invariants) | The nested `internal/` (compiler), the public contract only in the root package, other module roots are imported only by adapters and the composition root. Table access is checked at review |
| [AR-005](architecture.md#8-architectural-invariants) | The `platform/*` row: without module imports. Session logic stays in `account`. The absence of business logic is checked at review |

## 8. Tests, Migrations, Go Version

Only physical placement is defined here. The testing strategy, the choice of test libraries and test infrastructure are defined by the future Testing Strategy.

### Unit Tests

**Status:** `APPROVED`

- `_test.go` files lie in the same directory as the code they test.
- By default, an external test package is used (for example `package application_test`): the test works through the exported API.
- An internal test package (for example `package application`) is acceptable when unexported logic needs to be checked.

**Rationale:** tests from a separate tree would not see packages in the module's nested `internal/`.

### Test Doubles

**Status:** `APPROVED`

- Test doubles of ports and consumer-side interfaces lie in `_test.go` files of the package that uses them.
- A separate package for doubles used in several packages is created only by a separate decision, when such a need appears.

Whether to write doubles manually or generate them, and with which libraries, is decided by the Testing Strategy.

**Rationale:** a double lies next to the consumer of the interface, as does the interface itself ([Ports](#ports)), and does not get into the production build.

### Adapter Integration Tests

**Status:** `APPROVED`

- Adapter integration tests that work with real PostgreSQL or Redis lie next to the adapter, for example in `adapters/outbound/postgres/*_test.go`.
- They are marked with the build tag `//go:build integration`. A regular `go test ./...` does not run them.

How PostgreSQL and Redis are started for tests and whether end-to-end / API tests exist is defined by the Testing Strategy.

**Rationale:** the test has access to the adapter package in the nested `internal/`, and the split into unit and integration tests is explicit.

### Migrations

**Status:** `APPROVED`

- All golang-migrate migrations lie in a single `migrations/` directory in the repository root and form one sequence.
- The file name states the module that owns the tables, for example `000001_account_create_accounts.up.sql`.

Migrations are run by the golang-migrate CLI as a separate step before the application starts. Migration rules are defined by [Persistence Conventions §7](persistence-conventions.md#7-migrations).

**Rationale:** this is the standard form for golang-migrate, so it does not determine in advance how migrations will be run. The module in the file name shows who owns the tables, for checking AR-004.

### Go Version

**Status:** `APPROVED`

The Go version is defined by the `go` directive in [go.mod](../../go.mod). This document does not duplicate the version number. The major versions of PostgreSQL (18) and Redis (8.x) are defined by [Persistence Conventions §2](persistence-conventions.md#environment). Minor and patch versions and images are defined by Infrastructure Bootstrap.

**Rationale:** a single source of truth without manual synchronization.

## 9. Known Consequences and Deferred Decisions

### Known Consequences

These consequences of approved decisions are accepted deliberately.

| Consequence | Cause | What to do |
|---|---|---|
| Risk of an import cycle between modules. Example: `account` adapters import the root of module B, and B's adapters import the `account` root. Then a cycle arises through the root packages, which assemble their adapters | The module's public contract and assembly lie in one root package | The compiler will detect the cycle. If a need for mutual dependency between two modules appears, the organization of the contract is decided separately, and this document is updated |
| Type translation is needed at the boundaries: between the module's internal and public types (in the root package) and between another module's public types and own types (in the inbound adapter) | Internal packages do not import their root. Domain and Application do not import other modules | Apply as a normal part of the implementation |
| Application logs only through an outbound port | Application does not import Platform | Apply if a use case needs logging |

### Open Questions

None.

### Deferred Decisions

These decisions are deliberately deferred. Until they are defined, the agent cannot rely on any option.

Rows marked "Defined" have already been closed by the corresponding document. They are kept for traceability.

| Decision | Deferred to |
|---|---|
| HTTP server timeout values and graceful shutdown parameters | **Partially defined:** the model of timeouts, limits and graceful shutdown — [API Conventions §11](api-conventions.md#11-http-server-timeout-and-limit-model). The values remain deferred until Infrastructure Bootstrap |
| PostgreSQL and Redis versions, how migrations are run | **Defined:** [Persistence Conventions §2](persistence-conventions.md#environment) (PostgreSQL 18, Redis 8.x), [§7](persistence-conventions.md#7-migrations) (golang-migrate CLI as a separate step). Minor and patch versions and images remain deferred until Infrastructure Bootstrap |
| Context key for "who is acting", claims format | **Defined:** [Security Conventions §7](security-conventions.md#authorization-and-who-is-acting) (context key), [§3](security-conventions.md#3-jwt) (claims) |
| URL prefixes and route versioning | **Defined:** [API Conventions §3](api-conventions.md#3-urls-versioning-route-groups) |
| Representation of business errors in the module's public methods | When the first consumer module appears |
| Choice of test libraries, how test doubles are created, integration test infrastructure, end-to-end / API tests | Testing Strategy |
| A separate package for test doubles used in several packages | When such a need appears |
| Automatic checking of the import table and the choice of tools for it | Engineering Tooling setup |
