---
name: plan-review
description: Review Skill of the PLAN_REVIEW stage. Independently, in an isolated context, checks the implementation-plan and its PLANNED applicability-decisions (SECURITY_REVIEW, RECONCILIATION) against story-spec, the applicable design artifacts and the project's Testing Strategy, writes a review-report with findings and a verdict, and returns a result envelope. Does not change the checked artifacts and does not approve the plan. Started only manually with ready paths, record_id and attempt number.
argument-hint: "story=<ID> plan=<path> spec=<path> design=<path> [impact=<path>] [api=<path>] [db=<path>] sec_decision=<path> rec_decision=<path> testing_strategy=<path> output=<path> record_id=review-report/<NNNN> attempt=<N> [mode=pilot] [previous_reports=<path,...>]"
disable-model-invocation: true
user-invocable: true
context: fork
agent: general-purpose
background: false
allowed-tools: Read, Grep, Glob, Write, Bash(git log:*), Bash(git status:*), Bash(date:*)
disallowed-tools: Edit, NotebookEdit, AskUserQuestion, Agent, Skill
---

# PLAN_REVIEW — Review Skill

## 1. Purpose

Independently check the `implementation-plan` (OD-04). The Skill executes the PLAN_REVIEW stage according to `docs/workflow/stages.md` (section 9) and `docs/workflow/stage-map.yaml` (`reviews: [IMPLEMENTATION_PLANNING]`). These files and `docs/contracts/` are authoritative.

Context isolation is risk-triggered for PLAN_REVIEW (OD-10, `review_context_isolation`). This Skill always runs isolated (`context: fork`), which satisfies every trigger. Inputs are only the files at the given paths and the Foundation sources; the working context and self-assessment of IMPLEMENTATION_PLANNING are not evidence. The reviewer does not fix the plan (`workflow-policies.md` §4).

**Boundaries:** the Skill does not change the plan, the PLANNED records, the design, `story-spec`, the Testing Strategy, the Foundation or the code; does not write code or tests; does not approve the plan, dependencies or risks. `PASS` is not a Human Approval: the plan and the design are approved only by a human at HUMAN_TECHNICAL_APPROVAL (`artifact-rules.md` §5).

**Language** (`AGENTS.md`): `review-report` is written in English. Quotations of the human's decisions stay verbatim.

## 2. Inputs

| Argument | Required | Meaning |
|---|---|---|
| `story` | yes | Story ID |
| `plan` | yes | A ready path of the current `implementation-plan` |
| `spec`, `design` | yes | Ready paths of the current approved `story-spec` and the current `solution-design` |
| `impact`, `api`, `db` | no | Ready paths of the current `impact-analysis`, `api-design`, `db-design`, each only when its stage is applicable. If one is absent, do not search for or use the artifact |
| `sec_decision`, `rec_decision` | yes | Ready paths of the PLANNED `applicability-decision` records of SECURITY_REVIEW and RECONCILIATION |
| `testing_strategy` | yes | A ready path of the project's Testing Strategy (a mandatory input, `stage-io-map.yaml` → `PLAN_REVIEW`). There is no exception in `mode=pilot` |
| `output` | yes | A ready path of the `review-report` file |
| `record_id` | yes | `review-report/<NNNN>`, taken from the argument |
| `attempt` | yes | The stage attempt number |
| `mode=pilot` | no | A pilot outside the registered Workflow |
| `previous_reports` | no | Previous PLAN_REVIEW `review-report` files of this Story |

There is no human: the Skill does not ask. If a mandatory argument is missing or an input cannot be read (in particular the Testing Strategy, FG-01) — `BLOCKED` before the check (section 6).

As needed: the project structure (`docs/architecture/package-map.md`) and the existing code, specific paths. `clarification-record`, the review reports of earlier stages and chat history are not inputs.

## 3. Procedure

1. **Check the arguments.** In `mode=pilot`, `output` cannot lie in `docs/stories/`. If `output` already exists — `BLOCKED`, do not overwrite.
2. **Read the artifacts directly** from the given paths, then the Testing Strategy and the needed Foundation sections. Check summaries and tables against the primary text.
3. **Check the previous reports**, if passed: each `OPEN` finding receives a new state; the same problem keeps its `F-NNN`; new IDs continue the highest number (otherwise from `F-001`).
4. **Check** according to section 4, classify the findings (section 3.1), determine the verdict (section 6).
5. **Write `review-report`** to `output` (section 5) and return the envelope.

### 3.1. Findings

- Only material problems. `severity` — `BLOCKING` or `NON_BLOCKING` (`artifact-rules.md` §6).
  - `BLOCKING`: an `AC` without a step or a check; a step that changes or contradicts the design; a missing or wrong risk category; a Testing Approach inconsistent with the Testing Strategy or a touched risk category without a check; a wrong or unjustified PLANNED decision (in particular a `not_met` without a source, AP-02 not applied to an ambiguous trigger); an open decision silently made by the plan; a dependency reclassified or presented as approved; a weakened mandatory Foundation rule (RR-14); contract metadata violations.
  - `NON_BLOCKING`: an improvement that does not violate requirements and rules.
  - Do not downgrade an AC or Foundation rule violation; when in doubt — `BLOCKING` (`workflow-policies.md` §7).
- Each finding: `F-NNN`; severity; location; explanation; evidence; a recommendation and the stage that must fix it.

## 4. Quality Criteria

1. **AC and design coverage.** Each `AC` of `story-spec` has an implementation step and a check. Each step realizes a design element and references a `REQ`/`AC`; there are no steps without a source and no changes of the design (`stages.md` §9).
2. **Ordering and dependencies.** The step order respects the dependencies between steps and the design (for example, a migration before its adapter). Dependencies come from `solution-design` with its OD-09 classification, unchanged.
3. **Story Risk Assessment** (OD-15, `workflow-policies.md` §18). Each risk category the design touches is identified with a source; none is missing.
4. **Testing Approach.** Consistent with the Testing Strategy; each `AC` and each touched risk category has a check; additional checks for risky changes (§18); Test-First where appropriate (Q3).
5. **PLANNED applicability decisions.** `kind: PLANNED`; `target_stage` SECURITY_REVIEW / RECONCILIATION; the triggers of `stage-map.yaml` (`APPLICABILITY_SECURITY_REVIEW`, `APPLICABILITY_RECONCILIATION`) verbatim, each with a status and a source; AP-01, AP-02, AP-03; consistency with the Story Risk Assessment and the design (for example, credentials or untrusted input ⇒ the SEC triggers).
6. **Risks and human actions.** The plan identifies the risks and the external actions that need a human's decision; nothing is silently decided or approved.
7. **Foundation and security.** Mandatory Architecture and Security rules are not weakened (RR-14).
8. **Metadata** (`artifact-rules.md` §1, §2): front matter fields; `inputs` versions match the passed artifacts; conditional artifacts only when passed; `commit` of each source matches `git log -1 --format=%h -- <path>` (otherwise `STALE`, `BLOCKING`); no `path: .`; the pilot line in `mode=pilot`.

## 5. Output

One `review-report` file, in English. Front matter — `artifact-rules.md` §2 only: `record_type: review-report`; `record_id`; `story_id`; `produced_by: { stage: PLAN_REVIEW, attempt }`; `created_at` (`date -u +%Y-%m-%dT%H:%M:%SZ`); `subjects` — exactly `{ type: implementation-plan, version }` and `{ record: <id> }` of the two PLANNED records (`story-spec` and the design artifacts are references of the check, not its subjects); `verdict`; `findings` — `{ id, severity, summary, status }` for each new and each previous `OPEN` finding.

Body: 1. Subject of the Check; 2. Summary; 3. Findings; 4. Previous Findings; 5. Checked Criteria (items 1–8).

In `mode=pilot`, the first line of the body: "Pilot artifact outside the official registered Workflow. Not officially approved and not an input of stages of registered Stories; may be an input of subsequent stages of the same pilot."

## 6. Verdict and Return

PLAN_REVIEW is mandatory (`optional: false`) with loopbacks `IMPLEMENTATION_PLANNING`, `DB_DESIGN`, `API_DESIGN`, `SOLUTION_DESIGN`, `IMPACT_ANALYSIS` (without `crosses_gates`) and `SPECIFICATION`, `CLARIFICATION` (with `crosses_gates: [HUMAN_SPEC_APPROVAL]`). Allowed: **`PASS`**, **`CHANGES_REQUIRED`**, **`BLOCKED`**; not `NOT_APPLICABLE`.

| Situation | Verdict |
|---|---|
| No `OPEN` `BLOCKING` findings | `PASS` |
| `BLOCKING` findings in the plan or the PLANNED records | `CHANGES_REQUIRED`, `loopback_target: IMPLEMENTATION_PLANNING` |
| `BLOCKING` findings that require a design or specification change | `CHANGES_REQUIRED`, the corresponding Design Stage, `SPECIFICATION` or `CLARIFICATION` |
| A Foundation change is required | `BLOCKED` (Foundation Change Required) |
| A decision from the list of escalations is needed (`workflow-policies.md` §2) | `BLOCKED` (Escalation Required) |
| An argument is missing, an input cannot be read (including the Testing Strategy) or `output` exists | `BLOCKED` before the check, no file |

- Several targets — the highest upstream one: CLARIFICATION, SPECIFICATION, IMPACT_ANALYSIS, SOLUTION_DESIGN, API_DESIGN, DB_DESIGN, IMPLEMENTATION_PLANNING.
- `loopback_target` is only a proposal; routing and gate confirmation belong to the Orchestrator.
- The report is written for `PASS`, `CHANGES_REQUIRED` and a `BLOCKED` found during the check: `outputs: [{ record: <record_id> }]`; `outputs: []` only when there was no check.
- In `mode=pilot`, add "pilot" to `summary`.

Envelope — `docs/contracts/workflow-state.md` §6, unchanged:

```yaml
result:
  story_id: US-001
  stage: PLAN_REVIEW
  attempt: 1
  verdict: PASS                      # PASS | CHANGES_REQUIRED | BLOCKED
  outputs:
    - { record: review-report/0004 }
  loopback_target: null              # only for CHANGES_REQUIRED
  blocker: null                      # for BLOCKED: { reason, details, needed }
  summary: "0 BLOCKING, 1 NON_BLOCKING; pilot"
```

## 7. Constraints

- Write only the `output` file. Do not change the checked artifacts, the records, the Testing Strategy, the Foundation, `docs/contracts/`, the code or other files.
- Do not change `state.yaml`, `history.jsonl`, statuses, Currency or approvals.
- Do not compute paths or record numbers. Do not ask the human, do not make product or architectural decisions, do not approve anything, do not perform a Loopback or a Human Gate, do not start other stages, Skills or subagents.
- Bash — only `git log`, `git status` and `date`. No `git commit`, `git push` or destructive operations.
