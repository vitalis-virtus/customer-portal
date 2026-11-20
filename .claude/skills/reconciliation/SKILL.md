---
name: reconciliation
description: Stage Skill of the RECONCILIATION stage. Assesses the applicability of RECONCILIATION on entry (triggers REC-01…REC-06, the PLANNED decision may only be expanded), writes an ENTRY applicability-decision and, if applicable, checks the full traceability between the Acceptance Criteria, the design, the actual Git changes and the tests in both directions, and writes a reconciliation-report bound to the Changeset Content Identity. Returns a result envelope. Does not run tests and does not change code or artifacts. Started only manually with ready paths, record ids and attempt number.
argument-hint: "story=<ID> spec=<path> design=<path> [api=<path>] [db=<path>] [impact=<path>] plan=<path> changeset=<path> code_review=<path> [security_review=<path>] security_decision=<path> planned_decision=<path> history=<path> output=<path> record_id=reconciliation-report/<NNNN> record_output=<path> decision_id=applicability-decision/<NNNN> attempt=<N> [clarification=<path>] [mode=pilot] [previous_reports=<path,...>]"
disable-model-invocation: true
user-invocable: true
context: fork
agent: general-purpose
background: false
allowed-tools: Read, Grep, Glob, Write, Bash(git log:*), Bash(git status:*), Bash(git diff:*), Bash(git show:*), Bash(git rev-parse:*), Bash(git ls-files:*), Bash(git cat-file:*), Bash(git merge-base:*), Bash(date:*)
disallowed-tools: Edit, NotebookEdit, AskUserQuestion, Agent, Skill
---

# RECONCILIATION — Stage Skill

## 1. Purpose

A full check of traceability between the Acceptance Criteria, the design, the implementation and the tests (Q27). The Skill executes the RECONCILIATION stage according to `docs/workflow/stages.md` (section 11), `docs/workflow/stage-map.yaml` (`type: verification`, `optional: true`, `APPLICABILITY_RECONCILIATION`) and `docs/workflow/workflow-policies.md` §5, §7. These files and `docs/contracts/` are authoritative.

**Boundaries** (`stage-map.yaml`: "Traceability only; the factual check is done by VERIFICATION", WF-05):
- the stage checks traceability only; it does not run tests and does not check whether human decisions are reflected in canonical artifacts (that is VERIFICATION);
- evidence is the actual repository content at the Changeset Content Identity, not the descriptions and self-assessments of earlier stages;
- the Skill does not change code, tests, Git history, artifacts, records, approvals, State or History; does not perform Loopbacks or Human Gates. It runs in a separate context (`context: fork`).

**Language** (`AGENTS.md`): both files in English; the human's text verbatim.

## 2. Inputs

| Argument | Required | Meaning |
|---|---|---|
| `story` | yes | Story ID |
| `spec` | yes | The current approved `story-spec` |
| `design` | yes | The current `solution-design` |
| `api`, `db`, `impact` | no | The current `api-design`, `db-design`, `impact-analysis`, each only when its stage is applicable (`impact` serves REC-03 and the traceability of the design) |
| `plan` | yes | The current approved `implementation-plan` |
| `changeset` | yes | The current `implementation-changeset` |
| `code_review` | yes | The CODE_REVIEW `review-report` with `PASS` for the current Changeset Content Identity |
| `security_review` | no | The SECURITY_REVIEW `review-report` with `PASS`, if SECURITY_REVIEW was applicable |
| `security_decision` | yes | The valid SECURITY_REVIEW ENTRY `applicability-decision` |
| `planned_decision` | yes | The PLANNED `applicability-decision` of RECONCILIATION |
| `history` | yes | The Story's `history.jsonl` (`story-history`, read-only; REC-01…REC-04) |
| `record_output`, `decision_id` | yes | The ENTRY decision path and `applicability-decision/<NNNN>` |
| `output`, `record_id` | yes | The `reconciliation-report` path and `reconciliation-report/<NNNN>`, written only if applicable |
| `attempt` | yes | The stage attempt number |
| `clarification` | no | `clarification-record`, only to find the source of a gap |
| `mode=pilot` | no | A pilot outside the registered Workflow |
| `previous_reports` | no | Previous `reconciliation-report` files of this Story |

Logical inputs — `stage-io-map.yaml` → `RECONCILIATION`. A missing argument or an unreadable input — `BLOCKED` before the assessment.

## 3. Procedure

1. **Check the arguments and preconditions.** No output in `docs/stories/` in `mode=pilot`; `output` and `record_output` do not exist. Precondition (`stages.md`): the CODE_REVIEW report has `PASS`, no `OPEN` `BLOCKING`, `subjects` = the current `{ type: implementation-changeset, version, changeset_commit }`; SECURITY_REVIEW: the ENTRY decision is `NOT_APPLICABLE` for this identity, or `security_review` has `PASS` for it. Otherwise — `BLOCKED` before the assessment.
2. **Verify the Git identity**: the commits exist; `changeset_commit` is the tip of `feature_branch`; `git merge-base --is-ancestor <base_commit> <changeset_commit>`; the actual `git diff --name-status` matches the changeset's files. A mismatch — `BLOCKED` (the content is not identified).
3. **Assess applicability on entry** (section 3.1) and write the ENTRY record.
4. **If `NOT_APPLICABLE`** — return the envelope.
5. **If `APPLICABLE`** — build and check the traceability (section 4), classify the findings, write the `reconciliation-report` (section 5), return the envelope.

### 3.1. Applicability on Entry

- Triggers — `stage-map.yaml` → `APPLICABILITY_RECONCILIATION`, REC-01…REC-06, verbatim, each `met` / `not_met` with a source: REC-01, REC-02, REC-04, REC-06 — the events and human decisions of `history`; REC-03 — the ENTRY decisions of API_DESIGN, DB_DESIGN, IMPACT_ANALYSIS (the passed conditional artifacts and the history events); REC-05 — the Story Type.
- `basis` — all `basis_facts` of the condition, including the Changeset Content Identity and `{ record: <planned decision id> }`. AP-01, AP-02 (ambiguous → `met`), AP-03. `not_applicable_evidence` — for each trigger why it is not met; the basic requirements check of VERIFICATION is performed anyway.
- **PLANNED vs ENTRY** (AP-06, AP-10): the ENTRY decision may only expand applicability; a PLANNED `APPLICABLE` makes the ENTRY `APPLICABLE`. The PLANNED record is not changed; a previous `NOT_APPLICABLE` with a changed basis is not used (AP-08).
- ENTRY record — `artifact-rules.md` §2: `record_type: applicability-decision`, `record_id: <decision_id>`, `story_id`, `produced_by: { stage: RECONCILIATION, attempt }`, `created_at`, `subjects` — `{ type: implementation-changeset, version, changeset_commit }`, `target_stage: RECONCILIATION`, `kind: ENTRY`, `result`, `triggers`, `basis`; no `verdict` / `findings`.

## 4. Traceability Check (if applicable)

1. **AC → design → changes → checks.** For each `AC-NN` of `story-spec`: its place in the design (`solution-design`, `api-design`, `db-design`), the plan step, the changes in the changeset that implement it (files and lines at `changeset_commit`), and the tests or other checks that cover it (actual test code, not the description). A missing link is a `BLOCKING` finding (`stages.md`, Completion Conditions).
2. **Change → basis.** For each changed file and each material change of the diff: a basis in a `REQ`/`AC`, a design decision or a plan step. A change without a basis is a finding.
3. **Review consistency.** The `OPEN` findings of CODE_REVIEW and SECURITY_REVIEW are not contradicted by the traceability; a gap they already report keeps its `F-NNN`.
4. **Source of a gap** — the stage where it arose (as needed, `clarification-record` and `history`). The Skill does not run tests and does not evaluate the factual correctness of the code.

Findings — `artifact-rules.md` §6: `F-NNN` (stable in the Story), `BLOCKING` / `NON_BLOCKING`, location, explanation, evidence, recommendation and the stage where the gap arose. Previous `OPEN` findings receive a new state.

## 5. Output (only `APPLICABLE`)

`reconciliation-report` — the record header of `artifact-rules.md` §2 (`record_type: reconciliation-report`, `record_id`, `story_id`, `produced_by: { stage: RECONCILIATION, attempt }`, `created_at`, `subjects` — the Changeset Content Identity `{ type: implementation-changeset, version, changeset_commit }` and the versions of `story-spec`, the design artifacts and `implementation-plan` that were traced, `verdict`, `findings`). Body: 1. Subject; 2. Summary; 3. Traceability Matrix (AC → design → step → changes → checks); 4. Changes Without a Basis; 5. Findings; 6. Previous Findings.

In `mode=pilot`, the first line of the body of each file: "Pilot artifact outside the official registered Workflow. Not officially approved and not an input of stages of registered Stories; may be an input of subsequent stages of the same pilot."

## 6. Verdict and Return

RECONCILIATION has `optional: true`. Loopbacks: `IMPLEMENTATION` (without `crosses_gates`); `IMPLEMENTATION_PLANNING`, `DB_DESIGN`, `API_DESIGN`, `SOLUTION_DESIGN`, `IMPACT_ANALYSIS` (through HUMAN_TECHNICAL_APPROVAL); `SPECIFICATION`, `CLARIFICATION` (through HUMAN_TECHNICAL_APPROVAL and HUMAN_SPEC_APPROVAL). Allowed: **`PASS`**, **`NOT_APPLICABLE`**, **`CHANGES_REQUIRED`**, **`BLOCKED`**.

| Situation | Verdict | `outputs` |
|---|---|---|
| An argument or precondition is not met, an input cannot be read, an output exists, the Git identity does not match | `BLOCKED` before the assessment | `[]` |
| All triggers `not_met` and the PLANNED decision is not `APPLICABLE` | `NOT_APPLICABLE` | `[{ record: <decision_id> }]` |
| `APPLICABLE`, each AC has a chain and each change has a basis (no `OPEN` `BLOCKING`) | `PASS` | `[{ record: <decision_id> }, { record: <record_id> }]` |
| `APPLICABLE`, a traceability gap | `CHANGES_REQUIRED`, `loopback_target` — the stage where the gap arose (the highest upstream one if several) | the same |
| A Foundation change or an escalation is required | `BLOCKED` (Foundation Change Required / Escalation Required) | the same |

- `loopback_target` is only a proposal; routing and gate confirmation belong to the Orchestrator.
- In `mode=pilot`, add "pilot" to `summary`.

```yaml
result:
  story_id: US-001
  stage: RECONCILIATION
  attempt: 1
  verdict: NOT_APPLICABLE            # PASS | NOT_APPLICABLE | CHANGES_REQUIRED | BLOCKED
  outputs:
    - { record: applicability-decision/0007 }
  loopback_target: null
  blocker: null
  summary: "NOT_APPLICABLE (REC-01…REC-06 not_met); pilot"
```

## 7. Constraints

- Write only `record_output` and, if applicable, `output`. Do not modify code, tests, artifacts, records, `history.jsonl`, the Foundation, `docs/contracts/` or other files.
- Git — only the read commands of `allowed-tools`; no writing operations.
- Do not run tests or the application. Do not change `state.yaml`, statuses or approvals. Do not ask the human, do not perform a Loopback or a Human Gate, do not start other stages, Skills or subagents.
