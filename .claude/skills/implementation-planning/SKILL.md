---
name: implementation-planning
description: Stage Skill of the IMPLEMENTATION_PLANNING stage. Turns the approved story-spec and the reviewed technical design (solution-design and, if applicable, impact-analysis, api-design, db-design, with the DESIGN_REVIEW PASS report) into an implementation-plan — ordered steps linked to AC and design decisions, the Story Risk Assessment, the Testing Approach, risks and human actions — and writes the PLANNED applicability-decisions for SECURITY_REVIEW and RECONCILIATION. Returns a result envelope. Does not change the design, write code or tests. Started only manually with ready paths, versions, record ids and attempt number.
argument-hint: "story=<ID> spec=<path> design=<path> [impact=<path>] [api=<path>] [db=<path>] review=<path> testing_strategy=<path> output=<path to implementation-plan v1.md> version=1 sec_record_output=<path> sec_record_id=applicability-decision/<NNNN> rec_record_output=<path> rec_record_id=applicability-decision/<NNNN> attempt=<N> [mode=pilot] [reason=\"<the human's decision after RESUME>\"]"
disable-model-invocation: true
user-invocable: true
---

# IMPLEMENTATION_PLANNING — Stage Skill

## 1. Purpose

Plan the implementation of the approved and reviewed design. The Skill executes the IMPLEMENTATION_PLANNING stage according to `docs/workflow/stages.md` (section 9), `docs/workflow/stage-map.yaml` and `docs/workflow/workflow-policies.md` §18 (OD-15). These files and `docs/contracts/` are authoritative. If they diverge from this Skill, they apply.

The stage is mandatory (`optional: false`) and does not return `NOT_APPLICABLE`.

**Stage boundaries:**
- the plan **uses** the design; it does not change requirements, use cases, interfaces, schemas, migrations or dependency decisions. A design that is incomplete or contradictory for planning is a return to the corresponding Design Stage (section 6), not a decision of the plan;
- it does not write code, tests, SQL or migration files and does not run anything;
- it does not approve dependencies, risks or the plan: the plan is checked by PLAN_REVIEW and approved by a human at HUMAN_TECHNICAL_APPROVAL (`approval_gate`). The PLANNED applicability decisions are a forecast (AP-10).

**Language** (`AGENTS.md`): all files are written in English. The human's reason passed in `reason` is quoted verbatim in its original language.

## 2. Inputs

Arguments (`$ARGUMENTS`):

| Argument | Required | Meaning |
|---|---|---|
| `story` | yes | Story ID |
| `spec` | yes | A ready path of the current approved `story-spec` |
| `design` | yes | A ready path of the current `solution-design` |
| `impact`, `api`, `db` | no | Ready paths of the current `impact-analysis`, `api-design`, `db-design`. Each is passed only when its stage is applicable. If one is absent, do not search for or use the artifact, even if its files exist |
| `review` | yes | A ready path of the DESIGN_REVIEW `review-report` with `verdict: PASS` for the current design versions |
| `testing_strategy` | yes | A ready path of the project's Testing Strategy (`testing-strategy`, a mandatory input of `stage-io-map.yaml` → `IMPLEMENTATION_PLANNING`). It is mandatory in `mode=pilot` as well: no Workflow rule makes an exception. While FG-01 is open, the document does not exist, and the stage is `BLOCKED` |
| `output` | yes | A ready path of the `implementation-plan` file |
| `version` | yes | The version of `implementation-plan`. In this increment — only `1` |
| `sec_record_output`, `sec_record_id` | yes | A ready path and `applicability-decision/<NNNN>` of the PLANNED decision for SECURITY_REVIEW |
| `rec_record_output`, `rec_record_id` | yes | The same for RECONCILIATION |
| `attempt` | yes | The stage attempt number |
| `mode=pilot` | no | A pilot outside the registered Workflow |
| `reason` | no | The verbatim reason of the human's `RESUME` decision after a `CHANGES_REQUIRED` of this stage |

There is no human in the context: the Skill asks nothing. If a mandatory argument is missing or an input cannot be read, the verdict is `BLOCKED` (section 6). Paths, versions and record numbers are taken from the arguments, never computed. Each attempt receives new record identities; an existing output file is never overwritten.

Logical inputs — `docs/contracts/stage-io-map.yaml` → `IMPLEMENTATION_PLANNING`: `story-spec`, `solution-design`, the DESIGN_REVIEW `review-report`, `testing-strategy`, and `impact-analysis`, `api-design`, `db-design` when applicable. As needed (`stages.md`): the project structure from the Architecture Foundation (`docs/architecture/package-map.md`) and the existing code, specific paths with `commit` (D-3).

`clarification-record`, the applicability records of the TECHNICAL_DESIGN phase and chat history are not inputs. Read only the documents you need.

## 3. Procedure

1. **Check the arguments.** In `mode=pilot`, no output may lie in `docs/stories/`. `version` must be `1`. If `output`, `sec_record_output` or `rec_record_output` already exists — `BLOCKED`, do not overwrite.
2. **Check the review precondition** (`stages.md` §9; `design-review/SKILL.md` §5). The `review` report has `record_type: review-report`, `story_id` of this Story, `produced_by.stage: DESIGN_REVIEW`, `verdict: PASS` and no `OPEN` `BLOCKING` finding. Its `subjects` are exactly: the passed design artifact versions (`solution-design` and each passed `impact-analysis`, `api-design`, `db-design`) and three `{ record: applicability-decision/<NNNN> }` elements; `story-spec` is not among them. Otherwise — `BLOCKED` before planning.
3. **Check the Testing Strategy.** `testing_strategy` is passed and the file can be read. Otherwise — `BLOCKED` (Missing Information: FG-01), no files.
4. **Read the inputs** and the `NON_BLOCKING` findings of the review: each is planned for or explicitly left with a reason.
5. **Plan** according to section 3.1. Until the verdict is determined, keep the plan in memory.
6. **Determine the verdict** (section 6).
7. **Write the files** — only for `PASS`: `implementation-plan` to `output`, then the two PLANNED `applicability-decision` records. For other verdicts, write nothing.
8. **Return the envelope.**

### 3.1. Plan Content

- **Steps.** Ordered implementation steps by module and layer of the design (Package Map), each with: what is implemented, the design element it realizes (`solution-design` / `api-design` / `db-design` section), the `REQ`/`AC` it serves, its dependencies on other steps. Order respects dependencies (for example, the migration before the adapter that uses it). Steps realize the design; they do not add behavior.
- **Story Risk Assessment** (OD-15, `workflow-policies.md` §18). The risk categories the Story touches — at least those of §18 (Authentication and authorization, Persistence, Concurrency, Infrastructure) — each `touched` or `not touched` with a reference to `solution-design`, `impact-analysis` or the Story Type.
- **Testing Approach.** For each `AC` — the step that implements it and the check that verifies it (from its Verification Method); for each touched risk category — the additional check (§18 examples); where Test-First is appropriate (Q3). Tools, test kinds and scope come from the Testing Strategy.
- **New dependencies** — only those already classified in `solution-design` (OD-09), with the step that introduces them. Do not reclassify or approve them.
- **Risks and human actions** — risks of the implementation and actions that require a human's decision (for example, a dependency approval, an Infrastructure Bootstrap prerequisite).

### 3.2. Planned Applicability (SECURITY_REVIEW, RECONCILIATION)

- Use the triggers of `stage-map.yaml` → `APPLICABILITY_SECURITY_REVIEW` (SEC-01…SEC-11) and `APPLICABILITY_RECONCILIATION` (REC-01…REC-06), verbatim, each `met` / `not_met` with a source. AP-01, AP-02 (ambiguous → `met`), AP-03 apply.
- This is a forecast based only on the plan's artifacts (AP-10). For facts that only exist later (the changeset, events of the Story), assess what the plan's artifacts show and say so in the `ref`.
- Record: `kind: PLANNED`, `target_stage: SECURITY_REVIEW` / `RECONCILIATION`, `result`, `triggers`, `subjects` and `basis` — the versions of `story-spec`, `solution-design`, the passed conditional artifacts and `implementation-plan` v1, plus the `story_type` fact.

## 4. Quality Criteria

- Each `AC` is covered by an implementation step and a check; each touched risk category has a check (`stages.md`, Completion Conditions).
- Each step references a design element and a `REQ`/`AC`. There are no steps without a source and no changes of the design.
- Each `NON_BLOCKING` finding of the review is handled or explicitly left with a reason.
- Each open technical question is either delegated by the design to Implementation or returned (section 6). Nothing is silently decided.
- Each source in `inputs` is a specific path with a current `commit`.

## 5. Output

All files are written in English. In `mode=pilot`, the first line of the body of each file: "Pilot artifact outside the official registered Workflow. Not officially approved and not an input of stages of registered Stories; may be an input of subsequent stages of the same pilot." No statuses, Currency or approval in the front matter.

### 5.1. `implementation-plan` (only `PASS`)

Front matter — `docs/contracts/artifact-rules.md` §1, and only it: `artifact_type: implementation-plan`, `story_id`, `version`, `produced_by: { stage: IMPLEMENTATION_PLANNING, attempt }`, `created_at` (`date -u +%Y-%m-%dT%H:%M:%SZ`), `inputs`: `story-spec`, `solution-design`, the passed conditional artifacts, `{ record: <review record_id> }`, the Testing Strategy as `{ source: testing-strategy, path, commit }`, and each source read as `{ source, path, commit }`.

Body sections:
1. **Subject** — inputs and versions, the review record.
2. **Implementation Steps** — section 3.1.
3. **Story Risk Assessment**.
4. **Testing Approach** — a table `AC-NN` → step → check; then risk category → additional check.
5. **Dependencies** — from `solution-design`, with the introducing step.
6. **Planned Optional Stages** — SECURITY_REVIEW and RECONCILIATION: `result` and the record ids.
7. **Handling of Review Findings** — each `NON_BLOCKING` finding.
8. **Risks and Human Actions**.
9. **Limitations** — untracked or uncommitted paths, ambiguities.
10. **Human Decision** — only if `reason` is passed: a verbatim quotation and how it affected the plan.

### 5.2. PLANNED `applicability-decision` (two records, only `PASS`)

The common record header and the fields of the type from `artifact-rules.md` §2 (`record_type`, `record_id`, `story_id`, `produced_by`, `created_at`, `subjects`, `target_stage`, `kind: PLANNED`, `result`, `triggers`, `basis`), no `verdict` and `findings`. Body: Subject, Triggers (with the AP-02 reasons, if any).

## 6. Verdict and Return

IMPLEMENTATION_PLANNING has `optional: false` and loopbacks `DB_DESIGN`, `API_DESIGN`, `SOLUTION_DESIGN`, `IMPACT_ANALYSIS` (without `crosses_gates`) and `SPECIFICATION`, `CLARIFICATION` (with `crosses_gates: [HUMAN_SPEC_APPROVAL]`). Therefore **`PASS`**, **`CHANGES_REQUIRED`** and **`BLOCKED`** are allowed.

| Situation | Verdict | `outputs` |
|---|---|---|
| An argument is missing (in particular `testing_strategy`, FG-01), `version` ≠ `1`, an input cannot be read, an output already exists, or the review precondition is not met | `BLOCKED` before planning | `[]` |
| The plan is complete, the criteria of section 4 are met | `PASS` | `[{ type: implementation-plan, version: 1 }, { record: <sec_record_id> }, { record: <rec_record_id> }]` |
| The design is incomplete or contradictory for planning | `CHANGES_REQUIRED`, `loopback_target` — the Design Stage that must fix it (DB_DESIGN, API_DESIGN, SOLUTION_DESIGN, IMPACT_ANALYSIS) | `[]` |
| `story-spec` is incomplete for planning and can be fixed without a new human decision | `CHANGES_REQUIRED`, `loopback_target: SPECIFICATION` | `[]` |
| A new or different product decision of the human is needed | `CHANGES_REQUIRED`, `loopback_target: CLARIFICATION` | `[]` |
| A Foundation change is required | `BLOCKED` (Foundation Change Required) | `[]` |
| A decision from the list of escalations is needed (`workflow-policies.md` §2) | `BLOCKED` (Escalation Required) | `[]` |

Rules:

- If several targets fit, choose the highest upstream one: CLARIFICATION, SPECIFICATION, IMPACT_ANALYSIS, SOLUTION_DESIGN, API_DESIGN, DB_DESIGN.
- `loopback_target` is only a proposal; the route and the human's confirmation for targets with `crosses_gates` belong to the Orchestrator.
- For `CHANGES_REQUIRED`, list the problems in `summary` with references to `REQ`/`AC` and design sections. For `BLOCKED`, describe what is needed in `blocker.needed`.
- In `mode=pilot`, add the marker "pilot" to `summary`.

Envelope — `docs/contracts/workflow-state.md` §6, unchanged:

```yaml
result:
  story_id: US-001
  stage: IMPLEMENTATION_PLANNING
  attempt: 1
  verdict: PASS                      # PASS | CHANGES_REQUIRED | BLOCKED
  outputs:
    - { type: implementation-plan, version: 1 }
    - { record: applicability-decision/0004 }
    - { record: applicability-decision/0005 }
  loopback_target: null
  blocker: null                      # for BLOCKED: { reason, details, needed }
  summary: "9 steps, 41/41 AC covered; risks: persistence, concurrency, auth; SECURITY_REVIEW APPLICABLE, RECONCILIATION APPLICABLE; pilot"
```

## 7. Constraints

- Write only `output`, `sec_record_output` and `rec_record_output`, and only for `PASS`.
- Do not change the upstream artifacts, the review, the Foundation, `docs/contracts/`, the Workflow Definition, the code or other files. Do not write code, tests, SQL or migrations.
- Do not change `state.yaml`, `history.jsonl`, statuses, Currency or approvals.
- Bash — only `git log`, `git status`, `git ls-files` and `date`.
- Do not conduct an interview, do not approve dependencies, risks or the plan, do not perform a Loopback or a Human Gate, do not start other stages, Skills or subagents.
- Do not run `git commit`, `git push` or destructive operations.
