---
name: solution-design
description: Stage Skill of the SOLUTION_DESIGN stage. Turns the approved story-spec (and impact-analysis, if IMPACT_ANALYSIS is applicable) into a solution-design — Domain, Application, Integration and Infrastructure Design within the Architecture Foundation, with AC traceability, security decisions, dependency classification and the intent of interface and data changes — and returns a result envelope. Does not conduct an interview and does not design the API or the data schema. Started only manually with ready paths, version and attempt number.
argument-hint: "story=<ID> spec=<path> [impact=<path>] output=<path to solution-design v1.md> version=1 attempt=<N> [mode=pilot] [reason=\"<the human's decision after RESUME>\"]"
disable-model-invocation: true
user-invocable: true
---

# SOLUTION_DESIGN — Stage Skill

## 1. Purpose

Define the overall technical structure of the Story's solution to the extent it needs. The Skill executes the SOLUTION_DESIGN stage according to `docs/workflow/stages.md` (section 8) and `docs/workflow/stage-map.yaml`. These files and `docs/contracts/` are authoritative. If they diverge from this Skill, they apply.

The stage is mandatory (`optional: false`, OD-01). The scope of the design may be minimal, but it does not return `NOT_APPLICABLE`.

**Stage boundaries:**
- **API_DESIGN:** specific routes, methods, request and response formats, status and error codes, headers. SOLUTION_DESIGN records only the intent of external interface changes (section 5, item 8).
- **DB_DESIGN:** tables, columns, types, constraints, indexes, Redis keys, migrations. SOLUTION_DESIGN records only the intent of persisted data changes (section 5, item 9).
- The applicability of API_DESIGN and DB_DESIGN is assessed by these stages themselves, not by SOLUTION_DESIGN.
- No code is written. Names of types and files are only illustrative, marked with "for example".
- Requirements do not change: product decisions live in the approved `story-spec`.

The result is checked by DESIGN_REVIEW (`reviewed_by`).

**Language** (`AGENTS.md`): `solution-design` is written in English. The human's reason passed in `reason` is quoted verbatim in its original language.

## 2. Inputs

Arguments (`$ARGUMENTS`):

| Argument | Required | Meaning |
|---|---|---|
| `story` | yes | Story ID, for example `US-003` |
| `spec` | yes | A ready path of the current approved version of `story-spec`. Take its version number from the front matter |
| `impact` | no | A ready path of the current version of `impact-analysis`. The Orchestrator passes it only when IMPACT_ANALYSIS is applicable. If the argument is absent, the stage is not applicable: do not search for, require or use `impact-analysis`, even if its files exist in the Story directory (for example, a historical version after a repeated `NOT_APPLICABLE` assessment) |
| `output` | yes | A ready path of the `solution-design` file. The Skill does not compute the path itself |
| `version` | yes | The version of `solution-design`, matches `v<N>` in `output`. In this increment — only `1` |
| `attempt` | yes | The stage attempt number |
| `mode=pilot` | no | A pilot outside the registered Workflow |
| `reason` | no | The verbatim reason of the human's `RESUME` decision after a `CHANGES_REQUIRED` of this stage (Loopback confirmation) or after a `BLOCKED` of this stage (Recovery) |

There is no human in the context: the Skill asks nothing. If a mandatory argument is missing or an input cannot be read, the verdict is `BLOCKED` (section 6).

`reason` is the human's decision regarding the previous Loopback or blocking, not a source of requirements. It may explain why the human rejected the return or how the blocker was resolved. It does not add new requirements and does not by itself approve a dependency: only the Foundation record does (section 3.3).

Logical inputs — `docs/contracts/stage-io-map.yaml` → `SOLUTION_DESIGN`, paths — `docs/contracts/artifact-registry.yaml`:
- `story-spec` — `REQ`, `AC` with a Verification Method, scope, Story Type, "Open for Technical Design";
- `impact-analysis` (only if passed) — affected areas, impact, regression risks, "Open for Solution Design";
- `architecture-foundation`:
  - at minimum `docs/architecture/architecture.md` (style, modules, cross-cutting concerns, AR- invariants) and `docs/architecture/package-map.md` (modules, layers, ports, imports);
  - as needed — `api-conventions.md`, `persistence-conventions.md` (in particular §10 Consistency Analysis), `security-conventions.md`, only the sections the Story concerns.

As needed (`stages.md`, SOLUTION_DESIGN → "As needed"):
- the existing code of the affected modules — specific paths;
- Product Foundation documents whose rules (BR-, NFR-) `story-spec` directly references — only to understand the content of these rules and to place them in the design. This is not a new source of requirements.

Chat history and `clarification-record` are not inputs. Read only the documents you need (Progressive Context Loading).

## 3. Procedure

1. **Check the arguments.**
   - In `mode=pilot`, `output` cannot lie in `docs/stories/`.
   - `version` must be `1`, otherwise section 6.
   - If the `output` file already exists, the verdict is `BLOCKED`. Do not overwrite the file.
2. **Read `story-spec`.** Identify `REQ`, `AC`, scope, Story Type, the section "Open for Technical Design" and the Foundation rules it references.
3. **If `impact` is passed, read `impact-analysis`.** Each affected area, regression risk and item of "Open for Solution Design" must get a place in the design or an explicit justification of why the design does not touch it.
4. **Read the Foundation** (section 2): the minimum, then only the needed sections of the conventions.
5. **Design the solution** according to section 3.1.
6. **Check the boundaries** (section 3.2) and the quality criteria (section 4).
7. **Determine the verdict** (section 6). Until the verdict is determined, keep the design in memory, not in a file.
8. **Only for `PASS`, write `solution-design`** to `output` (section 5). For other verdicts, do not create the file, not even partially: an unregistered Working Draft will stop the next attempt.
9. **Return the envelope** (section 6).

### 3.1. Design Levels

For each level — decisions to the extent of the Story or the marker "not affected" with a short justification (`stages.md`, OD-01):

- **Domain** — entities, value objects, invariants; in which module and layer each business rule lives (`architecture.md` §4, "Business Rule Placement").
- **Application** — use cases with input, result and error categories; ports; transaction boundaries; data consistency between stores; who is acting and how the permission to act is checked.
- **Integration** — interaction of modules through public contracts (`package-map.md` §6), external systems, import directions (§7).
- **Infrastructure** — configuration, environments, runtime components, the composition root, startup and rollback, if the Story changes them.

Reference Foundation rules by ID or section. Do not rewrite them.

### 3.2. Boundaries with API_DESIGN and DB_DESIGN

| SOLUTION_DESIGN | Not SOLUTION_DESIGN |
|---|---|
| which use cases change client-visible behavior; error categories of the use case | routes, methods, DTOs, status codes, Problem Details, headers → API_DESIGN |
| which data is stored, changed or read and in which store; transaction boundaries; consistency | tables, columns, types, constraints, indexes, Redis keys, TTL, migrations → DB_DESIGN |
| placement in modules, layers and ports | code, names of files and types, except illustrative ones |

If a decision cannot be made without API or schema details, move the question to "Open for API and DB Design". Do not resolve it here.

### 3.3. Security and Dependencies

- **Security at the architecture level** — for every Story, regardless of whether SECURITY_REVIEW will be applicable (OD-05). Describe authentication, authorization and "who is acting", trust boundaries, sensitive data in logs and responses, sessions and credentials, if the Story touches them. The basis — references to `security-conventions.md` and AR-. Do not weaken mandatory rules (RR-14).
- **Dependencies** (`workflow-policies.md` §2, Risk-Based Dependency Approval, OD-09). Each new or changed dependency receives a classification according to the OD-09 criteria:
  - a simple auxiliary dependency that does not meet any criterion — proposed in the design with a justification;
  - a significant dependency, as well as any whose classification is ambiguous — `BLOCKED` (section 6). Exception: a dependency listed in `architecture.md` §6 "Approved Library Dependencies", used for exactly the listed purpose and within the listed scope (Story), is classified "significant — approved by the project owner" with a reference to that entry; it does not block. Any other package, purpose or Story still gives `BLOCKED`;
  - for Customer Portal, AR-012 applies: a new technology or a replacement of an entry of `architecture.md` §6 requires a Foundation update — `BLOCKED` (Foundation Change Required);
  - if there are no new dependencies, say so.
- **Analyses required by the Foundation.** For example, a Consistency Analysis for each use case that changes both PostgreSQL and Redis (`persistence-conventions.md` §10), covering all items of its table. If there are no such use cases, say so.

## 4. Quality Criteria

- Each `AC` has a place in the design: a use case, a rule or a component that implements it (`stages.md`, Completion Conditions).
- If `impact-analysis` is passed, each of its affected areas and each item of "Open for Solution Design" is considered.
- Each level (Domain, Application, Integration, Infrastructure) has decisions or a justified "not affected".
- The decisions do not violate the Architecture Rules and AR- invariants, the import directions and the business rule placement rules.
- Security decisions are described. Each dependency is classified. The analyses required by the Foundation are performed.
- The intent of external interface and persisted data changes is formulated explicitly: "yes" or "no", with a reference to `REQ`/`AC`.
- There are no API_DESIGN and DB_DESIGN details.
- There are no requirements without a source in `story-spec`. An open product question is not silently resolved.
- Each source in `inputs` is a specific path with a current `commit`.

## 5. Output

One `solution-design` file at the `output` path, in English.

Front matter — the fields of `docs/contracts/artifact-rules.md` §1, and only them:
- `artifact_type: solution-design`, `story_id`, `version`;
- `produced_by: { stage: SOLUTION_DESIGN, attempt }`;
- `created_at` — the actual UTC from `date -u +%Y-%m-%dT%H:%M:%SZ`;
- `inputs`:
  - `{ type: story-spec, version }`;
  - `{ type: impact-analysis, version }` — only if `impact` is passed;
  - each Foundation document and code path read as `{ source: architecture-foundation | product-foundation | codebase, path, commit }`, where `commit` is `git log -1 --format=%h -- <path>`.

**Source reference rules (D-3):**
- only specific paths, never the repository root `.`;
- a path without a history in Git does not get into `inputs`, because there is no `commit`. It is recorded in "Limitations";
- for a path read that has uncommitted changes (`git status --porcelain -- <path>`), the design relies on the actual content of the working tree, and the limitation is described in the body.

There are no statuses, Currency, approval or pilot marker in the front matter.

Body sections, brief and proportional to the Story:

1. **Subject** — the versions of `story-spec` and `impact-analysis` (if any), the sources read.
2. **Domain Design** — section 3.1.
3. **Application Design** — use cases, ports, transaction boundaries, consistency, error categories.
4. **Integration Design** — modules, public contracts, imports, external systems.
5. **Infrastructure Design** — or "not affected" with a justification.
6. **AC Traceability** — a table `AC-NN` → `REQ` → design element.
7. **Security** — section 3.3.
8. **External Interface Changes** — yes or no; which use cases add or change operations or visible behavior, with `REQ`/`AC`. Without routes and formats.
9. **Persisted Data Changes** — yes or no; which data, in which store, creation, change or only reading, with `REQ`/`AC`. Without a schema and migrations.
10. **Dependencies** — the classification according to OD-09 or "there are no new dependencies".
11. **Analyses Required by the Foundation** — for example Consistency Analysis, or "not needed" with a reason.
12. **Handling of `impact-analysis`** — only if it is passed: how each area and each item of "Open for Solution Design" was taken into account.
13. **Open for API and DB Design** — questions that the stage deliberately does not resolve.
14. **Limitations** — uncommitted or untracked paths, visibility boundaries.
15. **Human Decision** — only if `reason` is passed: a verbatim quotation and how it affected the design. A human's decision is recorded in the canonical artifact, not only in History (`workflow-policies.md` §2).

In `mode=pilot`, the first line of the body: "Pilot artifact outside the official registered Workflow. Not officially approved and not an input of stages of registered Stories; may be an input of subsequent stages of the same pilot."

## 6. Verdict and Return

SOLUTION_DESIGN has `optional: false` and the following loopbacks:
- `IMPACT_ANALYSIS` — without `crosses_gates`;
- `SPECIFICATION` and `CLARIFICATION` — with `crosses_gates: [HUMAN_SPEC_APPROVAL]`.

Therefore **`PASS`**, **`CHANGES_REQUIRED`** and **`BLOCKED`** are allowed. `NOT_APPLICABLE` is not allowed.

| Situation | Verdict | `outputs` |
|---|---|---|
| An argument is missing, `version` ≠ `1`, an input cannot be read or `output` already exists | `BLOCKED` before the design. No file | `[]` |
| The design is completed, the criteria of section 4 are met | `PASS` | `[{ type: solution-design, version }]` |
| `impact-analysis` is incomplete or wrong (an affected area, risk or migration is missed), or `impact` is not passed, but the design shows that it touches existing behavior, a contract, data or shared components (trigger IA-01…IA-04) | `CHANGES_REQUIRED`, `loopback_target: IMPACT_ANALYSIS` | `[]` |
| `story-spec` is infeasible, contradictory or incomplete for the design, and this can be eliminated in `story-spec` without a new decision of the human | `CHANGES_REQUIRED`, `loopback_target: SPECIFICATION` | `[]` |
| The design requires a new or different product decision of the human | `CHANGES_REQUIRED`, `loopback_target: CLARIFICATION` | `[]` |
| The Story requires a change to the Architecture or Product Foundation, in particular a new technology (AR-012) | `BLOCKED` (Foundation Change Required) | `[]` |
| A significant or ambiguous new dependency (OD-09) not covered by an approved entry (section 3.3), or another decision from the list of escalations (`workflow-policies.md` §2) | `BLOCKED` (Escalation Required) | `[]` |
| A decision is impossible without weakening a mandatory Foundation rule, in particular the Security Rules | `BLOCKED` (Foundation Rule Violation) | `[]` |

Rules:

- If several targets fit, choose the highest upstream one: CLARIFICATION, then SPECIFICATION, then IMPACT_ANALYSIS. After the return, the Story will go through the subsequent stages anyway (RR-06). List all detected problems in `summary`.
- `loopback_target` is only a proposal. The route, the human's confirmation for targets with `crosses_gates` (RR-04) and the re-entry of the target belong to the Orchestrator. The Skill does not perform or simulate them.
- For `CHANGES_REQUIRED`, findings are passed in `summary`: briefly each problem with a reference to `REQ`/`AC`, an `impact-analysis` area or a Foundation rule. Do not assign `F-NNN` IDs; that is done by review.
- For `BLOCKED`, describe what is needed in `blocker.needed`. The Skill does not create a `foundation-change-request`.
- In `mode=pilot`, add the marker "pilot" to `summary`.

Envelope — the format of `docs/contracts/workflow-state.md` §6, unchanged and without additional fields:

```yaml
result:
  story_id: US-003
  stage: SOLUTION_DESIGN
  attempt: 1
  verdict: PASS                      # PASS | CHANGES_REQUIRED | BLOCKED
  outputs:
    - { type: solution-design, version: 1 }
  loopback_target: null              # IMPACT_ANALYSIS | SPECIFICATION | CLARIFICATION only for CHANGES_REQUIRED
  blocker: null                      # for BLOCKED: { reason, details, needed }
  summary: "2 use cases, 13/13 AC; interface: yes; data: read only; no new dependencies; pilot"
```

## 7. Constraints

- Write only the `output` file and only for `PASS`.
- Do not change `story-spec`, `impact-analysis`, `clarification-record`, the code, the Foundation, the Workflow Definition, `docs/contracts/` or other artifacts.
- Do not change the `state.yaml` and `history.jsonl` of any Story, as well as statuses, Currency or approvals.
- Do not compute the path or version: take them from the arguments.
- Bash — only `git log`, `git status`, `git ls-files` and `date`.
- Do not conduct an interview, do not ask the human and do not make product decisions.
- Do not design the API and the data schema, do not write code.
- Do not perform a Loopback, do not pass a Human Gate, do not simulate the Orchestrator, DESIGN_REVIEW or the human's decisions. Do not start other stages, Skills or subagents.
- Do not run `git commit`, `git push` or destructive operations.
