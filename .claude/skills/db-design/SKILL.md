---
name: db-design
description: Stage Skill of the DB_DESIGN stage. Assesses the applicability of the stage by triggers DB-01…DB-05 for the approved story-spec and the current solution-design, writes an applicability-decision and, if the stage is applicable, a db-design (PostgreSQL tables, columns, constraints and indexes, Redis keys and TTL, persistence-level mechanisms, planned migrations, traceability) within the Persistence Conventions, and returns a result envelope. Does not conduct an interview, does not write SQL files, code or migrations and does not touch databases. Started only manually with ready paths, version, record_id and attempt number.
argument-hint: "story=<ID> spec=<path> design=<path> [api=<path>] [impact=<path>] output=<path to db-design v1.md> version=1 record_output=<path> record_id=applicability-decision/<NNNN> attempt=<N> [mode=pilot] [reason=\"<the human's decision after RESUME>\"]"
disable-model-invocation: true
user-invocable: true
---

# DB_DESIGN — Stage Skill

## 1. Purpose

Design the changes to the schema and structure of stored data. The Skill executes the DB_DESIGN stage according to `docs/workflow/stages.md` (section 8) and `docs/workflow/stage-map.yaml` (`APPLICABILITY_DB_DESIGN`). These files and `docs/contracts/` are authoritative. If they diverge from this Skill, they apply.

The stage is optional (`optional: true`), so the Skill first assesses its applicability and always records this assessment in an `applicability-decision` (`stage-io-map.yaml`, R4). `db-design` is created only when the stage is applicable.

**Stage boundaries:**
- it does not change requirements: product decisions live only in the approved `story-spec`;
- it does not redesign the solution: use cases, data placement, transaction boundaries, consistency requirements and the Consistency Analysis come from `solution-design`. `solution-design` defines the required guarantees; DB_DESIGN defines the persistence-level mechanisms that satisfy them (DB-D03). If a mechanism cannot satisfy them as stated, this is a return to SOLUTION_DESIGN (section 6), not a change of the design;
- it does not change the interface contract of `api-design`;
- it does not write SQL migration files, Go code, repositories or Lua scripts, does not run migrations and does not access PostgreSQL or Redis. SQL fragments in the body are allowed only as illustrations marked "for example";
- it does not change the Foundation and does not establish general persistence rules that the Foundation leaves `OPEN` or `DEFERRED` (section 3.4).

The result is finally checked by DESIGN_REVIEW: both `db-design` and the `NOT_APPLICABLE` justification (`reviewed_by`, `na_checked_by`).

**Language** (`AGENTS.md`): both files are written in English. The human's reason passed in `reason` is quoted verbatim in its original language.

## 2. Inputs

Arguments (`$ARGUMENTS`):

| Argument | Required | Meaning |
|---|---|---|
| `story` | yes | Story ID, for example `US-001` |
| `spec` | yes | A ready path of the current approved version of `story-spec`. Take its version number from the front matter |
| `design` | yes | A ready path of the current version of `solution-design`. Take its version number from the front matter |
| `api` | no | A ready path of the current version of `api-design`. Passed only when API_DESIGN is applicable |
| `impact` | no | A ready path of the current version of `impact-analysis`. Passed only when IMPACT_ANALYSIS is applicable |
| `output` | yes | A ready path of the `db-design` file. The Skill does not compute the path itself |
| `version` | yes | The version of `db-design`, matches `v<N>` in `output`. In this increment — only `1` |
| `record_output` | yes | A ready path of the `applicability-decision` file |
| `record_id` | yes | `applicability-decision/<NNNN>`. The number is taken from the argument |
| `attempt` | yes | The stage attempt number |
| `mode=pilot` | no | A pilot outside the registered Workflow |
| `reason` | no | The verbatim reason of the human's `RESUME` decision after a `CHANGES_REQUIRED` of this stage |

There is no human in the context: the Skill asks nothing. If a mandatory argument is missing or an input cannot be read, the verdict is `BLOCKED` (section 6).

**Conditional inputs.** The applicability of API_DESIGN and IMPACT_ANALYSIS is determined by the caller, not by this Skill. If `api` or `impact` is not passed, the corresponding stage is not applicable: do not search for, require or use `api-design` or `impact-analysis`, even if their files exist in the Story directory. The existence of a file is not evidence of applicability.

`reason` is the human's decision regarding the previous Loopback, not a source of requirements. It may explain why the human rejected the return. It does not add new requirements.

**Each entry** is a new applicability assessment based on the current facts and a new `applicability-decision`. Previous conclusions are not carried over automatically (AP-04, AP-08). The Skill does not create a `revalidation-record`.

**Attempts and record identity.** Each attempt, including an attempt after `CHANGES_REQUIRED` and the human's `RESUME`, receives its `record_id` and `record_output` from the caller. After `CHANGES_REQUIRED`, the previous record remains, so the next attempt must receive a new `record_id` and `record_output`. `output` of `db-design` may stay the same `v1` path, because the file is not created for `CHANGES_REQUIRED`. The Skill never computes the next record number or path and never overwrites an existing file: an existing `output` or `record_output` is `BLOCKED` (section 3, step 1). Attempts, routing and state transitions belong to the Orchestrator.

Logical inputs — `docs/contracts/stage-io-map.yaml` → `DB_DESIGN`, paths — `docs/contracts/artifact-registry.yaml`:
- `story-spec` — `REQ`, `AC`, scope, "Open for Technical Design";
- `solution-design` — data placement, use cases, transaction boundaries, consistency requirements and the Consistency Analysis, security decisions, "Persisted Data Changes", "Open for API and DB Design";
- `api-design` (only if passed) — operations and DTO fields that read or write stored data, uniqueness and conflict errors;
- `impact-analysis` (only if passed) — affected existing data structures, migrations and regression risks;
- `architecture-foundation`:
  - at minimum `docs/architecture/persistence-conventions.md` — only the sections the Story concerns (§6 schema, §7 migrations, §9 Redis, §10 consistency);
  - as needed — `package-map.md` (module ownership, §8 migrations), `architecture.md` (AR- invariants), `security-conventions.md` (§4 sessions in Redis, secrets and personal data), only the needed sections.

As needed (`stages.md`, DB_DESIGN → "As needed"): the current data schema — the files under `migrations/`, if they exist. It is a `codebase` source with specific paths.

`clarification-record`, `story-source`, the Product Foundation and chat history are not inputs. Read only the documents you need (Progressive Context Loading).

## 3. Procedure

1. **Check the arguments.**
   - In `mode=pilot`, `output` and `record_output` cannot lie in `docs/stories/`.
   - `version` must be `1`, otherwise section 6.
   - If the `output` or `record_output` file already exists, the verdict is `BLOCKED`. Do not overwrite the files.
2. **Read `story-spec` and `solution-design`**, and also `api-design` and `impact-analysis`, if they are passed. Identify `REQ`, `AC`, the data the use cases store, change or read, the required guarantees, the section "Persisted Data Changes" and the DB items of "Open for API and DB Design".
3. **Read the Foundation** (section 2) and, if it exists, the current schema. Record the source state (section 3.2).
4. **Evaluate the triggers DB-01…DB-05** (section 3.1) and determine `result`.
5. **If `result: APPLICABLE`**, design the persistence changes (section 3.3) and check the open and deferred decisions (section 3.4). Until the verdict is determined, keep the design in memory, not in a file.
6. **Determine the verdict** (section 6).
7. **Write the files** in this order:
   - `applicability-decision` to `record_output` (section 5.1), if the applicability assessment is completed;
   - `db-design` to `output` (section 5.2) — **only** for `PASS`.

   For other verdicts, do not create the `db-design` file, not even partially. An unregistered Working Draft will stop the next attempt.
8. **Return the envelope** (section 6).

### 3.1. Applicability Triggers

The triggers — `stage-map.yaml` → `APPLICABILITY_DB_DESIGN`, verbatim:

| ID | Trigger | What to check |
|---|---|---|
| DB-01 | The Story creates, changes or removes structures of stored data (tables, columns, indexes, constraints, structured keys). | `REQ`/`AC` and the data of the use cases; the current schema |
| DB-02 | The Story requires a schema or data migration. | New or changed structures; existing data that must be transformed |
| DB-03 | The Story changes the format, lifetime or method of storing data in any store. | Value representation, TTL, the store in which the data lives |
| DB-04 | The Story changes integrity or concurrent access guarantees at the store level. | The required guarantees of `solution-design`: uniqueness, locks, atomic operations |
| DB-05 | solution-design references a change of stored data. | The section "Persisted Data Changes" of `solution-design` |

Rules (`stage-map.yaml`, AP-01…AP-03):

- Each trigger receives `met` or `not_met` with a reference to the source: a file and section or an ID.
- The stage is applicable (`APPLICABLE`) if at least one trigger is `met` (AP-01).
- If the applicability of a trigger is ambiguous (data is missing, `solution-design` and `story-spec` diverge, the current schema could not be identified), the trigger is considered `met` (AP-02). Record the reason for the ambiguity.
- `NOT_APPLICABLE` — only when all triggers are `not_met`, each with a source (AP-03). The evidence (`not_applicable_evidence`) is a confirmation that the structure, format and guarantees of stored data do not change, with a reference to the section "Persisted Data Changes" of `solution-design`.

### 3.2. Source State (D-3)

- Reference only specific paths that were read, never the repository root `.`.
- The source version: `git log -1 --format=%h -- <path>` (`artifact-rules.md` §3, item 4).
- Check uncommitted changes with `git status --porcelain -- <path>` for each path read. If there are changes, rely on the actual content of the working tree and describe the limitation in the body.
- A path without a history in Git cannot be an `inputs` element, because there is no `commit`. It is recorded in "Limitations".
- The current schema is known only from the repository: the files under `migrations/`. If `migrations/` does not exist, the repository contains no migration files. This does not prove that no database schema or stored data exists in any environment: the Skill does not inspect databases. Design against the repository state and record in "Limitations" that the actual state of databases was not checked.

### 3.3. Persistence Design

Only to the extent the Story needs. Reference Foundation rules by section. Do not rewrite them.

1. **PostgreSQL** (`persistence-conventions.md` §6) — for each table: the owner module (AR-004), columns with types and nullability, primary key, foreign keys, `UNIQUE`, `CHECK`, `NOT NULL`, explicit constraint and index names by the templates of §6, indexes with the access pattern of `solution-design` or `api-design` that justifies each one. Identifiers, time and enumerations follow §6.
2. **Persistence-level mechanisms** (DB-D03) — for each guarantee required by `solution-design` (uniqueness under concurrent writes, row locks and per-user serialization, atomic Redis changes): the mechanism at the store level that satisfies it (§5, §6, §9) and a reference to the requirement in `solution-design`. Do not add guarantees that `solution-design` does not require and do not change its transaction boundaries.
3. **Redis** (§9, `security-conventions.md` §4) — only Story-specific keys: the key name by the template, the value structure and representation, the TTL and how it is set atomically, the atomic operations. Session keys are only referenced; their model does not change.
4. **Migrations** (§7, `package-map.md` §8) — the planned file names (6 digits, the owner module), what `up` does, whether a correct `down` exists, reversibility, the handling of existing data, and the transactional requirement of each migration: transactional, or a documented non-transactional exception (for example, `CREATE INDEX CONCURRENTLY`) with the possibility of a partial result. Do not assume the behavior of the golang-migrate driver. If this Story has the first migration, the check of how the driver executes an SQL file and whether it wraps it in a transaction belongs to this Technical Design (§7, §13), not to Implementation (section 3.4).
5. **Data security** — sensitive data and secrets in tables and keys, the absence of personal data and secrets in key names (§9), references to `security-conventions.md`. No new security policy.
6. **Traceability** — each schema change, key and migration is justified by a `REQ`/`AC` or a decision of `solution-design`, and by an `api-design` operation where it is relevant. Each referenced `AC-NN` exists in `acceptance_criteria` of `story-spec`. Not every AC must appear in `db-design`.

Each DB item of "Open for API and DB Design" gets a decision, an escalation (section 3.4) or the justification that it is not a DB question.

### 3.4. Open and Deferred Foundation Decisions

The statuses are from `persistence-conventions.md` §1 and §13. The handling is the same as in API_DESIGN.

| Situation | Handling |
|---|---|
| An `APPROVED` Foundation rule applies | Reference it. Do not rewrite or weaken it |
| A `DEFERRED` decision whose deferral point applies to this Story (for example, the compare-and-set mechanism — "Technical Design of the corresponding Story") | Story-scoped: resolve it within the approved Foundation, record it in "Story-Scoped Decisions" with the Foundation item, the rationale and the `REQ`/`AC` or design decision it serves. It does not become a general rule |
| The transactionality of SQL file execution by the golang-migrate driver, when this Story has the first migration ("Technical Design of the first migration", §7, §13) | A mandatory Technical Design check, not an Implementation detail. If the inputs and the sources read establish the driver's behavior with a source, record the result in "Story-Scoped Decisions". Otherwise do not assume it and do not move it to "Open for Implementation": `BLOCKED` (Missing Information), with the needed verified driver behavior in `blocker.needed` |
| A `DEFERRED` decision whose deferral point is another moment (Infrastructure Bootstrap, Deployment, a second module, a separate decision), and the design depends on it | `BLOCKED` (Escalation Required). If the design does not depend on it, only reference the deferred item |
| A decision that the Foundation allows only by separate approval, and the design actually requires it: an irreversible migration (§7), `ON DELETE CASCADE` (§6), an FK between modules or separate schemas and roles (§6), a key without TTL (§9), a new serialization mechanism (§10) | `BLOCKED` (Escalation Required) |
| The design requires creating administrator accounts in a migration while P6 Administrator Provisioning is not closed (§7; P6 is `OPEN` in `security-conventions.md` §12). This is a prohibition until a product dependency is resolved, not a decision by separate approval | `BLOCKED` (Foundation Change Required): P6 is closed only in the Product Foundation |
| An `OPEN` product question on which the design depends (for example, email case and normalization for BR-003 — "Clarification of US-001"), and the approved `story-spec` contains no human decision on it with a source | `CHANGES_REQUIRED`, `loopback_target: CLARIFICATION` |
| An `OPEN` product question that the approved `story-spec` decides with a source: a `REQ`/`AC` traced to a `CQ` decided by the human, while the Foundation names the Story Clarification as the place of the decision | Apply the decision within the approved Foundation. Reference the question, the `CQ`, the `REQ`/`AC` and the Foundation item. The decision is Story-scoped and does not become a general Foundation rule |
| An `OPEN` architectural or infrastructure question on which the design depends (for example, Redis persistence and failover) | `BLOCKED` (Escalation Required) |
| The design requires changing an `APPROVED` Foundation rule | `BLOCKED` (Foundation Change Required) |
| The design is impossible without weakening a mandatory rule, in particular the Security Rules | `BLOCKED` (Foundation Rule Violation) |

The list is not a reason to block by itself: a decision blocks the stage only when the current design actually requires it. Do not choose an option for the human. Name the exact Foundation item in `blocker.needed` or in `summary`.

## 4. Quality Criteria

- Each trigger has a conclusion and a source. The conclusions do not contradict `solution-design`.
- Each schema change, key and migration is justified by a requirement or the design and is described according to the Foundation conventions (`stages.md`, Completion Conditions).
- Each guarantee required by `solution-design` has a persistence-level mechanism, and no mechanism changes the design.
- Constraint and index names follow the templates. Migrations follow §7.
- Each DB item of "Open for API and DB Design" is considered. If `api-design` or `impact-analysis` is passed, its data-related content is considered.
- There are no requirements without a source in `story-spec`, no redesign of `solution-design` or `api-design`, no code and no SQL files.
- No `OPEN` or `DEFERRED` decision is silently resolved, and no unverified driver behavior is assumed.
- Each source in `inputs` is a specific path with a current `commit`.

## 5. Output

Both files are written in English. In `mode=pilot`, the first line of the body of both files: "Pilot artifact outside the official registered Workflow. Not officially approved and not an input of stages of registered Stories; may be an input of subsequent stages of the same pilot." There are no statuses, Currency, approval or pilot marker in the front matter.

### 5.1. `applicability-decision`

Front matter — the common record header and the fields of the type from `docs/contracts/artifact-rules.md` §2, and only them:

```yaml
---
record_type: applicability-decision
record_id: applicability-decision/0003       # from the argument
story_id: US-001
produced_by: { stage: DB_DESIGN, attempt: 1 }
created_at: 2026-11-20T12:00:00Z             # date -u +%Y-%m-%dT%H:%M:%SZ
subjects:                                    # the versions against which applicability was assessed
  - { type: story-spec, version: 1 }
  - { type: solution-design, version: 1 }
target_stage: DB_DESIGN
kind: ENTRY
result: APPLICABLE                           # APPLICABLE | NOT_APPLICABLE
triggers:
  DB-01: { status: met, ref: "story-spec v1 REQ-01; solution-design v1 §Persisted Data Changes" }
  DB-02: { status: met, ref: "migrations/ does not exist; a new table requires the first migration" }
  DB-03: { status: not_met, ref: "..." }
  DB-04: { status: met, ref: "solution-design v1 §Application Design: email uniqueness under concurrent writes" }
  DB-05: { status: met, ref: "solution-design v1 §Persisted Data Changes: yes" }
basis:                                       # basis_facts of the stage
  - { type: story-spec, version: 1 }
  - { type: solution-design, version: 1 }
---
```

`api-design` and `impact-analysis` are not in `subjects` and `basis`: they are not `basis_facts` of the stage. The record does not contain `verdict` and `findings`: it is not a review report (`artifact-rules.md` §2).

Body sections:
1. **Subject** — the versions of `story-spec` and `solution-design`, the conditional inputs passed, the Foundation and code paths read.
2. **Triggers** — for each DB-xx, the conclusion, the justification and the reason for the ambiguity if AP-02 was applied. For `NOT_APPLICABLE` — the evidence of section 3.1.
3. **Entry Reason** — only if `reason` is passed: a verbatim quotation and how it was taken into account.

### 5.2. `db-design` (only `PASS`)

Front matter — the fields of `docs/contracts/artifact-rules.md` §1, and only them:
- `artifact_type: db-design`, `story_id`, `version`;
- `produced_by: { stage: DB_DESIGN, attempt }`;
- `created_at` — the actual UTC from `date -u +%Y-%m-%dT%H:%M:%SZ`;
- `inputs`:
  - `{ type: story-spec, version }`;
  - `{ type: solution-design, version }`;
  - `{ type: api-design, version }` — only if `api` is passed;
  - `{ type: impact-analysis, version }` — only if `impact` is passed;
  - each Foundation document and code path read as `{ source: architecture-foundation | codebase, path, commit }` (section 3.2).

Body sections, brief and proportional to the Story:

1. **Subject** — the Story, the input versions, the `record_id` of the applicability decision, the `met` triggers, the sources read.
2. **PostgreSQL Changes** — tables, columns, types, nullability, keys, constraints, indexes and persistence-level integrity mechanisms with their justification, or "No PostgreSQL changes".
3. **Redis Changes** — keys, value representation, TTL, atomic operations, or "None".
4. **Migrations** — the planned files, the owner module, `up` / `down`, reversibility, transactionality, existing data, or "None".
5. **Data Security** — references to the rules for sensitive data and secrets.
6. **Traceability** — a table: change → `REQ`/`AC` → `solution-design` decision → `api-design` operation (where relevant).
7. **Story-Scoped Decisions** — decisions of section 3.4 with the Foundation item and the rationale, or "none".
8. **Handling of Inputs** — how "Persisted Data Changes", the DB items of "Open for API and DB Design" and, if passed, `api-design` and `impact-analysis` were taken into account.
9. **Open for Implementation** — details deliberately left to Implementation (SQL text, Go types). Not blockers and not mandatory Technical Design checks.
10. **Limitations** — uncommitted or untracked paths, the absence of migration files, the unchecked actual state of databases, ambiguities (AP-02).
11. **Human Decision** — only if `reason` is passed: a verbatim quotation and how it affected the design. A human's decision is recorded in the canonical artifact, not only in History (`workflow-policies.md` §2).

## 6. Verdict and Return

DB_DESIGN has `optional: true` and the following loopbacks:
- `API_DESIGN`, `SOLUTION_DESIGN` and `IMPACT_ANALYSIS` — without `crosses_gates`;
- `SPECIFICATION` and `CLARIFICATION` — with `crosses_gates: [HUMAN_SPEC_APPROVAL]`.

Therefore **`PASS`**, **`NOT_APPLICABLE`**, **`CHANGES_REQUIRED`** and **`BLOCKED`** are allowed.

| Situation | Verdict | `outputs` |
|---|---|---|
| An argument is missing, `version` ≠ `1`, an input cannot be read, `output` or `record_output` already exists | `BLOCKED` before the assessment. No files | `[]` |
| All triggers are `not_met` | `NOT_APPLICABLE` | `[{ record: <record_id> }]` |
| `APPLICABLE`, the design is completed, the criteria of section 4 are met | `PASS` | `[{ type: db-design, version }, { record: <record_id> }]` |
| `APPLICABLE`, `api-design` requires stored data or a guarantee that contradicts `solution-design` or cannot be provided under the Persistence Conventions, and the fix is in the interface contract | `CHANGES_REQUIRED`, `loopback_target: API_DESIGN` | `[{ record: <record_id> }]` |
| `APPLICABLE`, `solution-design` is incomplete or contradictory for the data (data placement, a transaction boundary, a required guarantee or the Consistency Analysis is missing), or no persistence-level mechanism can satisfy a guarantee as `solution-design` states it (DB-D03) | `CHANGES_REQUIRED`, `loopback_target: SOLUTION_DESIGN` | `[{ record: <record_id> }]` |
| `APPLICABLE`, the design changes, removes or migrates an existing data structure or existing data (identified in the current schema by a specific path), and `impact` is not passed or `impact-analysis` does not cover this area | `CHANGES_REQUIRED`, `loopback_target: IMPACT_ANALYSIS` | `[{ record: <record_id> }]` |
| `APPLICABLE`, `story-spec` is infeasible, contradictory or incomplete for the data, and this can be eliminated in `story-spec` without a new decision of the human | `CHANGES_REQUIRED`, `loopback_target: SPECIFICATION` | `[{ record: <record_id> }]` |
| `APPLICABLE`, the design requires a new or different product decision of the human (section 3.4) | `CHANGES_REQUIRED`, `loopback_target: CLARIFICATION` | `[{ record: <record_id> }]` |
| The design requires changing an `APPROVED` Foundation rule | `BLOCKED` (Foundation Change Required) | `[{ record }]`, if the assessment is completed, otherwise `[]` |
| A decision from the list of escalations is needed (`workflow-policies.md` §2) or section 3.4 requires escalation | `BLOCKED` (Escalation Required) | the same |
| The design is impossible without weakening a mandatory Foundation rule | `BLOCKED` (Foundation Rule Violation) | the same |
| Information that only a human can provide is missing | `BLOCKED` (Missing Information) | the same |

Rules:

- `CHANGES_REQUIRED` is possible only with `result: APPLICABLE`. If the stage is not applicable, the verdict is `NOT_APPLICABLE`.
- If several targets fit, choose the highest upstream one: CLARIFICATION, then SPECIFICATION, then IMPACT_ANALYSIS, then SOLUTION_DESIGN, then API_DESIGN. List all detected problems in `summary`.
- `loopback_target` is only a proposal. The route, the human's confirmation for targets with `crosses_gates` (RR-04) and the re-entry of the target belong to the Orchestrator. The Skill does not perform or simulate them and does not change the upstream artifacts. In the current Orchestrator MVP, re-entry into the targets without `crosses_gates` after a return from DB_DESIGN is not supported (DB-D01); this does not change the proposed target.
- For `CHANGES_REQUIRED`, findings are passed in `summary`: briefly each problem with a reference to `REQ`/`AC`, a `solution-design` or `api-design` section, an `impact-analysis` area or a Foundation rule. Do not assign `F-NNN` IDs; that is done by review.
- For `BLOCKED`, describe what is needed in `blocker.needed`: the exact Foundation item or decision and who must make it. The Skill does not create a `foundation-change-request`.
- In `mode=pilot`, add the marker "pilot" to `summary`.

Envelope — the format of `docs/contracts/workflow-state.md` §6, unchanged and without additional fields:

```yaml
result:
  story_id: US-001
  stage: DB_DESIGN
  attempt: 1
  verdict: PASS                      # PASS | NOT_APPLICABLE | CHANGES_REQUIRED | BLOCKED
  outputs:
    - { type: db-design, version: 1 }
    - { record: applicability-decision/0003 }
  loopback_target: null              # API_DESIGN | SOLUTION_DESIGN | IMPACT_ANALYSIS | SPECIFICATION | CLARIFICATION only for CHANGES_REQUIRED
  blocker: null                      # for BLOCKED: { reason, details, needed }
  summary: "APPLICABLE (DB-01, DB-02, DB-04, DB-05); 1 table, 1 migration; no Redis changes; pilot"
```

## 7. Constraints

- Write only the `record_output` file and, for `PASS`, the `output` file.
- Do not change `story-spec`, `solution-design`, `api-design`, `impact-analysis`, `migrations/`, the code, the Foundation, the Workflow Definition, `docs/contracts/` or other artifacts.
- Do not create SQL, migration or Lua files, do not run migrations and do not access PostgreSQL or Redis.
- Do not change the `state.yaml` and `history.jsonl` of any Story, as well as statuses, Currency or approvals.
- Do not compute the path, version or record number: take them from the arguments.
- Bash — only `git log`, `git status`, `git ls-files` and `date`.
- Do not conduct an interview, do not ask the human and do not make product or architectural decisions instead of the human or the corresponding stages.
- Do not perform a Loopback, do not pass a Human Gate, do not simulate the Orchestrator, DESIGN_REVIEW or the human's decisions. Do not start other stages, Skills or subagents.
- Do not run `git commit`, `git push` or destructive operations.
