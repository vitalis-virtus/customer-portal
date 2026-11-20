# Infrastructure Bootstrap — Customer Portal (Minimal, Local Testing)

**Document status:** `DRAFT`. Sections 2–4 describe configured files; section 5 contains `PROPOSED` values awaiting a human decision. Nothing here is a deployment configuration.

## 1. Scope

The minimum local infrastructure needed to run the integration tests required by the [Testing Strategy](testing-strategy.md) and [Persistence Conventions §11](persistence-conventions.md#11-persistence-testing): a PostgreSQL 18 test database and a way to run golang-migrate migrations against it.

Out of scope: Redis (added when a Story needs it, for example sessions in US-002), deployment, CI, TLS, secret stores.

## 2. PostgreSQL Test Database

**Status:** `CONFIGURED` (files exist; start-up and tests not yet verified).

- File: [`compose.test.yaml`](../../compose.test.yaml), service `postgres-test`, image `postgres:18` (major version per [Persistence Conventions §2](persistence-conventions.md#environment)).
- Credentials and the database name come from `.env` (template [`.env.example`](../../.env.example); `.env` is ignored by Git). Compose refuses to start if they are not set.
- Port bound to `127.0.0.1` only (default `55432`), named volume `postgres-test-data`, health check `pg_isready`.

```
cp .env.example .env
docker compose -f compose.test.yaml --env-file .env up -d --wait postgres-test
docker compose -f compose.test.yaml --env-file .env down      # -v drops the data
```

## 3. Migrations

**Status:** `CONFIGURED`.

- The golang-migrate **CLI** runs as a separate step before the application and tests ([Persistence Conventions §7](persistence-conventions.md#7-migrations)), via the official image `migrate/migrate:v4.19.1` — the version whose driver behavior is recorded in Persistence Conventions §7 ("Driver Evidence"). The `postgres://` URL selects the `database/postgres` driver, normal mode (`x-multi-statement` not enabled).
- Migration files live in [`migrations/`](../../migrations/) (Package Map §8); the directory is empty: no tables are defined here.

```
docker compose -f compose.test.yaml --env-file .env --profile migrate run --rm migrate up
docker compose -f compose.test.yaml --env-file .env --profile migrate run --rm migrate version
```

A Story that adds migrations runs `up` from an empty database as its migration check (Persistence Conventions §11).

## 4. Integration Tests

**Status:** `CONFIGURED` (no integration test exists yet).

- Integration tests (`//go:build integration`, Package Map §8) read `TEST_DATABASE_URL`; without it they fail with a clear message rather than skip silently. They never connect to any other database.
- Order: start `postgres-test` → `migrate up` → `go test -tags=integration ./...`.
- Tests clean up the data they create.

## 5. Application Values — PROPOSED

Not approved. Values are MVP defaults for a local educational service, to be confirmed by a human; until then the deferred decisions of [API Conventions §11](api-conventions.md#11-http-server-timeout-and-limit-model) and [Persistence Conventions §3](persistence-conventions.md#3-postgresql-connection-and-pool) remain open.

| Parameter | Proposed |
|---|---|
| HTTP read header timeout | 5s |
| HTTP read timeout | 10s |
| HTTP write timeout | 15s |
| HTTP idle timeout | 60s |
| HTTP shutdown timeout | 10s |
| HTTP header size limit | 16 KiB |
| HTTP body size limit | 64 KiB |
| PostgreSQL pool: max / min connections | 10 / 0 |
| PostgreSQL connection establishment timeout | 5s |
| PostgreSQL wait for a free connection | 2s |
| PostgreSQL `statement_timeout` | 5s |
| PostgreSQL max connection lifetime / idle time / health check period | 30m / 5m / 1m |

The names of the application's environment variables are defined by `platform/config` when it is implemented.
