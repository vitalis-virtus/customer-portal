---
name: api-design
description: Stage Skill of the API_DESIGN stage. Assesses the applicability of the stage by triggers API-01…API-04 for the approved story-spec and the current solution-design, writes an applicability-decision and, if the stage is applicable, an api-design (operations, routes, request and response contracts, status and error codes, security requirements, AC traceability) within the API and Security Conventions, and returns a result envelope. Does not conduct an interview, does not design the data schema and does not write code or OpenAPI. Started only manually with ready paths, version, record_id and attempt number.
argument-hint: "story=<ID> spec=<path> design=<path> output=<path to api-design v1.md> version=1 record_output=<path> record_id=applicability-decision/<NNNN> attempt=<N> [mode=pilot] [reason=\"<the human's decision after RESUME>\"]"
disable-model-invocation: true
user-invocable: true
---

# API_DESIGN — Stage Skill

## 1. Purpose

Design the changes to the system's external interface. The Skill executes the API_DESIGN stage according to `docs/workflow/stages.md` (section 8) and `docs/workflow/stage-map.yaml` (`APPLICABILITY_API_DESIGN`). These files and `docs/contracts/` are authoritative. If they diverge from this Skill, they apply.

The stage is optional (`optional: true`), so the Skill first assesses its applicability and always records this assessment in an `applicability-decision` (`stage-io-map.yaml`, R4). `api-design` is created only when the stage is applicable.

**Stage boundaries:**
- it does not change requirements: product decisions live only in the approved `story-spec`;
- it does not redesign the solution: use cases, modules, ports, transactions and error categories come from `solution-design`;
- it does not design stored data: tables, columns, constraints, indexes, migrations and Redis keys belong to DB_DESIGN;
- it does not write Go code or handlers and does not create or edit `docs/api/openapi.yaml`: Implementation updates OpenAPI together with the handler (`api-conventions.md` §13);
- it does not change the Foundation and does not establish general API rules that the Foundation leaves `OPEN` or `DEFERRED` (section 3.4).

The result is finally checked by DESIGN_REVIEW: both `api-design` and the `NOT_APPLICABLE` justification (`reviewed_by`, `na_checked_by`).

**Language** (`AGENTS.md`): both files are written in English. The human's reason passed in `reason` is quoted verbatim in its original language.

## 2. Inputs

Arguments (`$ARGUMENTS`):

| Argument | Required | Meaning |
|---|---|---|
| `story` | yes | Story ID, for example `US-003` |
| `spec` | yes | A ready path of the current approved version of `story-spec`. Take its version number from the front matter |
| `design` | yes | A ready path of the current version of `solution-design`. Take its version number from the front matter |
| `output` | yes | A ready path of the `api-design` file. The Skill does not compute the path itself |
| `version` | yes | The version of `api-design`, matches `v<N>` in `output`. In this increment — only `1` |
| `record_output` | yes | A ready path of the `applicability-decision` file |
| `record_id` | yes | `applicability-decision/<NNNN>`. The number is taken from the argument |
| `attempt` | yes | The stage attempt number |
| `mode=pilot` | no | A pilot outside the registered Workflow |
| `reason` | no | The verbatim reason of the human's `RESUME` decision after a `CHANGES_REQUIRED` of this stage |

There is no human in the context: the Skill asks nothing. If a mandatory argument is missing or an input cannot be read, the verdict is `BLOCKED` (section 6).

`reason` is the human's decision regarding the previous Loopback, not a source of requirements. It may explain why the human rejected the return. It does not add new requirements.

**Each entry** is a new applicability assessment based on the current facts and a new `applicability-decision`. Previous conclusions are not carried over automatically (AP-04, AP-08). The Skill does not create a `revalidation-record`.

**Attempts and record identity.** Each attempt, including an attempt after `CHANGES_REQUIRED` and the human's `RESUME`, receives its `record_id` and `record_output` from the caller. After `CHANGES_REQUIRED`, the previous record remains, so the next attempt must receive a new `record_id` and `record_output`. `output` of `api-design` may stay the same `v1` path, because the file is not created for `CHANGES_REQUIRED`. The Skill never computes the next record number or path and never overwrites an existing file: an existing `output` or `record_output` is `BLOCKED` (section 3, step 1). Attempts, routing and state transitions belong to the Orchestrator.

Logical inputs — `docs/contracts/stage-io-map.yaml` → `API_DESIGN`, paths — `docs/contracts/artifact-registry.yaml`:
- `story-spec` — `REQ`, `AC` with a Verification Method, scope, "Open for Technical Design";
- `solution-design` — use cases and their error categories, security decisions, "External Interface Changes", "Open for API and DB Design";
- `architecture-foundation`:
  - at minimum `docs/architecture/api-conventions.md` — only the sections the Story concerns;
  - as needed — `docs/architecture/security-conventions.md` (authentication, sessions, CSRF, non-disclosure), only the needed sections.

As needed (`stages.md`, API_DESIGN → "As needed"): the current interface contract — `docs/api/openapi.yaml`, if it exists, or specific code paths of existing routes. It is a `codebase` source with a specific path.

`impact-analysis` is not an input of this stage: do not search for, require or use it, even if its files exist in the Story directory. `clarification-record`, `story-source`, the Product Foundation and chat history are not inputs either. Read only the documents you need (Progressive Context Loading).

## 3. Procedure

1. **Check the arguments.**
   - In `mode=pilot`, `output` and `record_output` cannot lie in `docs/stories/`.
   - `version` must be `1`, otherwise section 6.
   - If the `output` or `record_output` file already exists, the verdict is `BLOCKED`. Do not overwrite the files.
2. **Read `story-spec` and `solution-design`.** Identify `REQ`, `AC`, scope, the use cases, the error categories, the security decisions, the section "External Interface Changes" and the API items of "Open for API and DB Design".
3. **Read the Foundation** (section 2) and, if it exists, the current interface contract. Record the source state (section 3.2).
4. **Evaluate the triggers API-01…API-04** (section 3.1) and determine `result`.
5. **If `result: APPLICABLE`**, design the interface changes (section 3.3) and check the open and deferred decisions (section 3.4). Until the verdict is determined, keep the design in memory, not in a file.
6. **Determine the verdict** (section 6).
7. **Write the files** in this order:
   - `applicability-decision` to `record_output` (section 5.1), if the applicability assessment is completed;
   - `api-design` to `output` (section 5.2) — **only** for `PASS`.

   For other verdicts, do not create the `api-design` file, not even partially. An unregistered Working Draft will stop the next attempt.
8. **Return the envelope** (section 6).

### 3.1. Applicability Triggers

The triggers — `stage-map.yaml` → `APPLICABILITY_API_DESIGN`, verbatim:

| ID | Trigger | What to check |
|---|---|---|
| API-01 | The Story adds, changes or removes an operation of the external interface, a request or response format, error codes or headers. | `REQ`/`AC` and the use cases that add or change operations; the current contract |
| API-02 | The Story changes the client-visible behavior of an existing operation without changing its form (validation, authorization, errors, side effects). | Existing operations in the current contract or code that the use cases affect |
| API-03 | The Story changes a client-visible mechanism of authentication, sessions or request protection. | Security decisions of `solution-design`; `REQ`/`AC` about login, sessions, cookies, CSRF |
| API-04 | solution-design references a change of the external interface. | The section "External Interface Changes" of `solution-design` |

Rules (`stage-map.yaml`, AP-01…AP-03):

- Each trigger receives `met` or `not_met` with a reference to the source: a file and section or an ID.
- The stage is applicable (`APPLICABLE`) if at least one trigger is `met` (AP-01).
- If the applicability of a trigger is ambiguous (data is missing, `solution-design` and `story-spec` diverge, an existing operation could not be identified), the trigger is considered `met` (AP-02). Record the reason for the ambiguity.
- `NOT_APPLICABLE` — only when all triggers are `not_met`, each with a source (AP-03). The evidence (`not_applicable_evidence`) is a confirmation that the external interface does not change either in form or in visible behavior, with a reference to the section "External Interface Changes" of `solution-design`.

### 3.2. Source State (D-3)

- Reference only specific paths that were read, never the repository root `.`.
- The source version: `git log -1 --format=%h -- <path>` (`artifact-rules.md` §3, item 4).
- Check uncommitted changes with `git status --porcelain -- <path>` for each path read. If there are changes, rely on the actual content of the working tree and describe the limitation in the body.
- A path without a history in Git cannot be an `inputs` element, because there is no `commit`. It is recorded in "Limitations".

### 3.3. Interface Design

For each operation that the Story adds, changes or removes — only to the extent the Story needs:

1. **Operation** — method, route, route group (`api-conventions.md` §2–3), idempotency (§4), the use case of `solution-design` it serves.
2. **Request** — Content-Type, body size, JSON decoding, query and path parameters, the DTO fields with JSON type, mandatory status and validation rules (§6–7). Data in URLs — §2.
3. **Response** — success status code (§5), the response DTO, relevant headers and caching (§10).
4. **Errors** — for each failure: the error category of `solution-design` → HTTP code → code from the catalog, Problem Details and validation errors (§8), the 401 / 403 / 404 order.
5. **Security** — authentication, authorization, CSRF and Origin check, cookies in responses (§9, `security-conventions.md`), non-disclosure of sensitive data in responses. Only references to the rules; do not weaken them (RR-14).
6. **Client-visible behavior** — for an existing operation (API-02): what changes for the client and what stays the same.
7. **AC** — each operation, error or behavior change is linked to at least one `AC-NN`, and each referenced `AC-NN` exists in `acceptance_criteria` of `story-spec`. Not every AC must appear in `api-design`: AC without an interface aspect are not listed.

Reference Foundation rules by section. Do not rewrite them. Names of DTO and fields follow the JSON conventions (§7). Each API item of "Open for API and DB Design" gets a decision, an escalation (section 3.4) or the justification that it is not an API question.

### 3.4. Open and Deferred Foundation Decisions (API-D02)

The statuses are from `api-conventions.md` §1 and §15 and `security-conventions.md` §12.

| Situation | Handling |
|---|---|
| An `APPROVED` Foundation rule applies | Reference it. Do not rewrite or weaken it |
| A `DEFERRED` decision whose deferral point is the Technical Design of the current Story, or "the first Story / module that needs it", when this Story is that one | Story-scoped: resolve it within the approved Foundation, record it in "Story-Scoped Decisions" with the Foundation item, the rationale and the `REQ`/`AC` or convention it serves. It does not become a general rule |
| A `DEFERRED` decision that requires separate approval, or whose deferral point is another moment (Infrastructure Bootstrap, a separate decision, when the need appears), and the contract depends on it | `BLOCKED` (Escalation Required). If the contract does not depend on it, only reference the deferred item |
| An `OPEN` product question (P1–P7, Profile composition and others) on which the contract depends, and the approved `story-spec` contains no human decision on it with a source | `CHANGES_REQUIRED`, `loopback_target: CLARIFICATION` |
| An `OPEN` product question that the approved `story-spec` decides with a source: a `REQ`/`AC` traced to a `CQ` decided by the human. This is the product owner's explicit approval for this Story (`security-conventions.md` §12, `api-conventions.md` §15) | Apply the decision within the approved Foundation. Reference the question ID, the `CQ`, the `REQ`/`AC` and the Foundation item. The decision is Story-scoped and does not become a general Foundation rule |
| An `OPEN` question for which the Foundation names another authority than the Story: a change of the non-disclosure of failed login reasons (P5, only through approval of the Security Conventions, `api-conventions.md` §15); the EPIC-003 product dependencies (P6, Target User Identification), which the Product Foundation requires to be resolved before Clarification (`epic-map.md`, "Open Product Dependencies"; not an input of this stage), while `api-conventions.md` §15 still lists them as `OPEN` | `BLOCKED` (Foundation Change Required), even if `story-spec` contains a decision |
| An `OPEN` architectural question (for example A2 — the CSRF token delivery method) on which the contract depends | `BLOCKED` (Escalation Required) |
| The contract requires changing an `APPROVED` Foundation rule | `BLOCKED` (Foundation Change Required) |
| The contract is impossible without weakening a mandatory rule, in particular the Security Rules | `BLOCKED` (Foundation Rule Violation) |

Do not choose an option for the human. Name the exact Foundation item in `blocker.needed` or in `summary`.

### 3.5. Impact Analysis Gap or Solution Design Gap

`IMPACT_ANALYSIS` is proposed only for a concrete gap: a specific **existing** operation or its client-visible behavior that the interface change changes or removes. The operation is identified in the current interface contract or in the code by a specific path (section 3.2). If there are no existing operations (no current contract and no route code), there is no such gap.

The absence of `impact-analysis` in the `inputs` of `solution-design` is not a gap by itself: it only shows that `solution-design` was designed without it. It is used only to classify a gap that has already been found. `impact-analysis` itself is not read.

| What is found about the existing operation | `inputs` of `solution-design` | Target |
|---|---|---|
| `solution-design` identifies it as affected, but the design for it (use case, error category, security decision) is missing or contradictory | any | `SOLUTION_DESIGN` |
| `solution-design` does not identify it as affected | without `impact-analysis` | `IMPACT_ANALYSIS`: the impact on an existing contract (IA-01) was not assessed |
| `solution-design` does not identify it as affected, and its section "Handling of `impact-analysis`" does not mention it | with `impact-analysis` | Without reading `impact-analysis`, it is not possible to tell whether the gap is in `impact-analysis` or in `solution-design`. Apply the upstream priority (section 6): `IMPACT_ANALYSIS`, and state this ambiguity in `summary` |
| The section "Handling of `impact-analysis`" mentions it, but the design does not cover it | with `impact-analysis` | `SOLUTION_DESIGN` |

In `summary`, name the existing operation, its path and the `REQ`/`AC` of the change.

## 4. Quality Criteria

- Each trigger has a conclusion and a source. The conclusions do not contradict `solution-design`.
- Each interface change is described according to the Foundation conventions and is linked to at least one existing `AC-NN` (`stages.md`, Completion Conditions).
- The error codes, statuses and headers are taken from the Foundation catalog or are Story-scoped decisions allowed by section 3.4.
- Security requirements reference the Foundation and do not weaken it.
- Each API item of "Open for API and DB Design" is considered.
- There are no requirements without a source in `story-spec`, no redesign of `solution-design`, no DB_DESIGN details and no code.
- No `OPEN` or `DEFERRED` decision is silently resolved.
- Each source in `inputs` is a specific path with a current `commit`.

## 5. Output

Both files are written in English. In `mode=pilot`, the first line of the body of both files: "Pilot artifact outside the official registered Workflow. Not officially approved and not an input of stages of registered Stories; may be an input of subsequent stages of the same pilot." There are no statuses, Currency, approval or pilot marker in the front matter.

### 5.1. `applicability-decision`

Front matter — the common record header and the fields of the type from `docs/contracts/artifact-rules.md` §2, and only them:

```yaml
---
record_type: applicability-decision
record_id: applicability-decision/0002       # from the argument
story_id: US-003
produced_by: { stage: API_DESIGN, attempt: 1 }
created_at: 2026-11-20T12:00:00Z             # date -u +%Y-%m-%dT%H:%M:%SZ
subjects:                                    # the versions against which applicability was assessed
  - { type: story-spec, version: 2 }
  - { type: solution-design, version: 1 }
target_stage: API_DESIGN
kind: ENTRY
result: APPLICABLE                           # APPLICABLE | NOT_APPLICABLE
triggers:
  API-01: { status: met, ref: "story-spec v2 REQ-01, AC-01; solution-design v1 §External Interface Changes" }
  API-02: { status: not_met, ref: "docs/api/openapi.yaml does not exist; no existing operations" }
  API-03: { status: not_met, ref: "..." }
  API-04: { status: met, ref: "solution-design v1 §External Interface Changes: yes" }
basis:                                       # basis_facts of the stage
  - { type: story-spec, version: 2 }
  - { type: solution-design, version: 1 }
---
```

The record does not contain `verdict` and `findings`: it is not a review report (`artifact-rules.md` §2).

Body sections:
1. **Subject** — the versions of `story-spec` and `solution-design`, the Foundation and code paths read.
2. **Triggers** — for each API-xx, the conclusion, the justification and the reason for the ambiguity if AP-02 was applied. For `NOT_APPLICABLE` — the evidence of section 3.1.
3. **Entry Reason** — only if `reason` is passed: a verbatim quotation and how it was taken into account.

### 5.2. `api-design` (only `PASS`)

Front matter — the fields of `docs/contracts/artifact-rules.md` §1, and only them:
- `artifact_type: api-design`, `story_id`, `version`;
- `produced_by: { stage: API_DESIGN, attempt }`;
- `created_at` — the actual UTC from `date -u +%Y-%m-%dT%H:%M:%SZ`;
- `inputs`:
  - `{ type: story-spec, version }`;
  - `{ type: solution-design, version }`;
  - each Foundation document and code path read as `{ source: architecture-foundation | codebase, path, commit }` (section 3.2).

Body sections, brief and proportional to the Story:

1. **Subject** — the versions of `story-spec` and `solution-design`, the `record_id` of the applicability decision, the `met` triggers, the sources read.
2. **Operations** — a table: operation, method, route, route group, authentication, success code, idempotency, use case, `AC`.
3. **Request and Response Contracts** — for each operation: parameters, request DTO with validation, response DTO, headers and caching.
4. **Errors** — for each operation: condition → category → HTTP code → error code, validation errors, `AC`.
5. **Security** — authentication, authorization, CSRF, cookies, non-disclosure; references to the rules.
6. **Changed Client-Visible Behavior** — for existing operations (API-02), or "no existing operations are changed".
7. **AC Traceability** — a table: interface change → `AC-NN` → `REQ`.
8. **Story-Scoped Decisions** — decisions of section 3.4 with the Foundation item and the rationale, or "none".
9. **Handling of `solution-design`** — how "External Interface Changes" and each API item of "Open for API and DB Design" were taken into account.
10. **Open for DB Design and Implementation** — questions that the stage deliberately does not resolve.
11. **Limitations** — uncommitted or untracked paths, ambiguities (AP-02).
12. **Human Decision** — only if `reason` is passed: a verbatim quotation and how it affected the design. A human's decision is recorded in the canonical artifact, not only in History (`workflow-policies.md` §2).

## 6. Verdict and Return

API_DESIGN has `optional: true` and the following loopbacks:
- `SOLUTION_DESIGN` and `IMPACT_ANALYSIS` — without `crosses_gates`;
- `SPECIFICATION` and `CLARIFICATION` — with `crosses_gates: [HUMAN_SPEC_APPROVAL]`.

Therefore **`PASS`**, **`NOT_APPLICABLE`**, **`CHANGES_REQUIRED`** and **`BLOCKED`** are allowed.

| Situation | Verdict | `outputs` |
|---|---|---|
| An argument is missing, `version` ≠ `1`, an input cannot be read, `output` or `record_output` already exists | `BLOCKED` before the assessment. No files | `[]` |
| All triggers are `not_met` | `NOT_APPLICABLE` | `[{ record: <record_id> }]` |
| `APPLICABLE`, the design is completed, the criteria of section 4 are met | `PASS` | `[{ type: api-design, version }, { record: <record_id> }]` |
| `APPLICABLE`, `solution-design` is incomplete, contradictory or infeasible for the interface (a use case, an error category or a security decision is missing or contradicts `story-spec` or the Foundation) | `CHANGES_REQUIRED`, `loopback_target: SOLUTION_DESIGN` | `[{ record: <record_id> }]` |
| `APPLICABLE`, a concrete impact-analysis gap according to section 3.5 | `CHANGES_REQUIRED`, `loopback_target: IMPACT_ANALYSIS` | `[{ record: <record_id> }]` |
| `APPLICABLE`, `story-spec` is infeasible, contradictory or incomplete for the interface, and this can be eliminated in `story-spec` without a new decision of the human | `CHANGES_REQUIRED`, `loopback_target: SPECIFICATION` | `[{ record: <record_id> }]` |
| `APPLICABLE`, the interface requires a new or different product decision of the human (section 3.4) | `CHANGES_REQUIRED`, `loopback_target: CLARIFICATION` | `[{ record: <record_id> }]` |
| The contract requires changing an `APPROVED` Foundation rule | `BLOCKED` (Foundation Change Required) | `[{ record }]`, if the assessment is completed, otherwise `[]` |
| A decision from the list of escalations is needed (`workflow-policies.md` §2) or section 3.4 requires escalation | `BLOCKED` (Escalation Required) | the same |
| The contract is impossible without weakening a mandatory Foundation rule | `BLOCKED` (Foundation Rule Violation) | the same |
| Information that only a human can provide is missing | `BLOCKED` (Missing Information) | the same |

Rules:

- `CHANGES_REQUIRED` is possible only with `result: APPLICABLE`. If the stage is not applicable, the verdict is `NOT_APPLICABLE`.
- If several targets fit, choose the highest upstream one: CLARIFICATION, then SPECIFICATION, then IMPACT_ANALYSIS, then SOLUTION_DESIGN. List all detected problems in `summary`.
- `loopback_target` is only a proposal. The route, the human's confirmation for targets with `crosses_gates` (RR-04) and the re-entry of the target belong to the Orchestrator. The Skill does not perform or simulate them and does not change the upstream artifacts.
- For `CHANGES_REQUIRED`, findings are passed in `summary`: briefly each problem with a reference to `REQ`/`AC`, a `solution-design` section or a Foundation rule. Do not assign `F-NNN` IDs; that is done by review.
- For `BLOCKED`, describe what is needed in `blocker.needed`: the exact Foundation item or decision and who must make it. The Skill does not create a `foundation-change-request`.
- In `mode=pilot`, add the marker "pilot" to `summary`.

Envelope — the format of `docs/contracts/workflow-state.md` §6, unchanged and without additional fields:

```yaml
result:
  story_id: US-003
  stage: API_DESIGN
  attempt: 1
  verdict: PASS                      # PASS | NOT_APPLICABLE | CHANGES_REQUIRED | BLOCKED
  outputs:
    - { type: api-design, version: 1 }
    - { record: applicability-decision/0002 }
  loopback_target: null              # SOLUTION_DESIGN | IMPACT_ANALYSIS | SPECIFICATION | CLARIFICATION only for CHANGES_REQUIRED
  blocker: null                      # for BLOCKED: { reason, details, needed }
  summary: "APPLICABLE (API-01, API-04); 2 operations, 5 AC linked; no story-scoped decisions; pilot"
```

## 7. Constraints

- Write only the `record_output` file and, for `PASS`, the `output` file.
- Do not change `story-spec`, `solution-design`, `docs/api/`, the code, the Foundation, the Workflow Definition, `docs/contracts/` or other artifacts.
- Do not change the `state.yaml` and `history.jsonl` of any Story, as well as statuses, Currency or approvals.
- Do not compute the path, version or record number: take them from the arguments.
- Bash — only `git log`, `git status`, `git ls-files` and `date`.
- Do not conduct an interview, do not ask the human and do not make product or architectural decisions instead of the human or the corresponding stages.
- Do not perform a Loopback, do not pass a Human Gate, do not simulate the Orchestrator, DESIGN_REVIEW or the human's decisions. Do not start other stages, Skills or subagents.
- Do not run `git commit`, `git push` or destructive operations.
