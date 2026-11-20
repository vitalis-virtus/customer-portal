---
name: verification
description: Stage Skill of the VERIFICATION stage. Performs the actual check of the Story result bound to the Changeset Content Identity and the Target Base Identity — the checks of the Testing Strategy, the Story Risk Assessment and the Testing Approach, actual evidence for each Acceptance Criterion, basic security checks, the applicability decisions of SECURITY_REVIEW and RECONCILIATION, Currency and review binding, the reflection of human decisions — and writes a verification-report with the Findings Disposition and the DoD state. Returns a result envelope. Does not repair code, commit, merge or push. Started only manually with ready paths, a prepared workspace, record_id and attempt number.
argument-hint: "story=<ID> spec=<path> changeset=<path> plan=<path> testing_strategy=<path> code_review=<path> [security_review=<path>] security_decision=<path> reconciliation_decision=<path> [reconciliation_report=<path>] state=<path> history=<path> workspace=<path> target_branch=<name> output=<path> record_id=verification-report/<NNNN> attempt=<N> [design=<path>] [api=<path>] [db=<path>] [mode=pilot] [previous_reports=<path,...>]"
disable-model-invocation: true
user-invocable: true
---

# VERIFICATION — Stage Skill

## 1. Purpose

An actual check of the Story result: behavior, test results, basic security checks and the Definition of Done. The Skill executes the VERIFICATION stage according to `docs/workflow/stages.md` (section 11), `docs/workflow/stage-map.yaml` (`optional: false`, OD-12, OD-15) and `docs/workflow/workflow-policies.md` §7, §8, §18. These files and `docs/contracts/` are authoritative.

**How it differs:** CODE_REVIEW reviews the code without running it; RECONCILIATION checks traceability without running tests; VERIFICATION **executes** the checks and confirms each Acceptance Criterion with actual evidence, checks the DoD and is the only stage that checks the reflection of human decisions (DoD 7).

**Boundaries:**
- the checks run only in `workspace` — a working tree prepared by the caller that contains the integration result of `changeset_commit` with the current Target Base Identity (OD-12). The Skill does not create, switch, merge, rebase, commit or push branches and does not modify the repository at the project root;
- the Skill does not repair code or tests and does not change requirements, the design, the plan, artifacts, records, approvals, `state.yaml` or `history.jsonl` (both are read-only inputs);
- checks that would touch shared infrastructure, production data, secrets or external systems are not executed without an authorization recorded in the canonical artifacts; they are reported as "not executed";
- the Skill does not accept risks, grant exceptions or lower finding classes (`workflow-policies.md` §7).

**Language** (`AGENTS.md`): `verification-report` in English; the human's text verbatim.

## 2. Inputs

| Argument | Required | Meaning |
|---|---|---|
| `story` | yes | Story ID |
| `spec` | yes | The current approved `story-spec` (AC and Verification Methods) |
| `changeset` | yes | The current `implementation-changeset` |
| `plan` | yes | The approved `implementation-plan` (Story Risk Assessment, Testing Approach) |
| `testing_strategy` | yes | The project's Testing Strategy (`testing-strategy`, FG-01). Mandatory in `mode=pilot` as well |
| `code_review` | yes | The CODE_REVIEW `review-report` with `PASS` for the current identity |
| `security_review` | no | The SECURITY_REVIEW `review-report`, if SECURITY_REVIEW was applicable |
| `security_decision`, `reconciliation_decision` | yes | The valid ENTRY `applicability-decision` records of SECURITY_REVIEW and RECONCILIATION |
| `reconciliation_report` | no | The `reconciliation-report`, if RECONCILIATION was applicable |
| `state`, `history` | yes | The Story's `state.yaml` and `history.jsonl`, read-only (Currency, review binding, approvals, human decisions) |
| `workspace` | yes | The prepared working tree with the integration result (section 1) |
| `target_branch` | yes | The target branch; its current tip is the Target Base Identity |
| `output`, `record_id` | yes | A ready path and `verification-report/<NNNN>` |
| `attempt` | yes | The stage attempt number |
| `design`, `api`, `db` | no | Design artifacts, as needed |
| `mode=pilot` | no | A pilot outside the registered Workflow |
| `previous_reports` | no | Previous `verification-report` files |

Logical inputs — `stage-io-map.yaml` → `VERIFICATION`: `story-spec`, `implementation-changeset`, `implementation-plan`, `testing-strategy`, the review reports, the `applicability-decision` records (`na_checked_by`), `codebase` (in particular the current state of the target branch), `reconciliation-report` when applicable. A missing argument or an unreadable input (in particular the Testing Strategy) — `BLOCKED` before the checks.

## 3. Procedure

1. **Check the arguments and preconditions** (`stages.md`): `output` does not exist; CODE_REVIEW `PASS`, and SECURITY_REVIEW and RECONCILIATION `PASS` or `NOT_APPLICABLE`, each for the current Changeset Content Identity. Otherwise — `BLOCKED`.
2. **Verify the identities.** Changeset: the commits exist, `changeset_commit` is the tip of the feature branch, `git merge-base --is-ancestor <base_commit> <changeset_commit>`, the actual changed files match the changeset. Target Base Identity: `verified_target_base` = `git rev-parse <target_branch>`. Workspace: its `HEAD` contains `changeset_commit` and `verified_target_base` (`git -C <workspace> merge-base --is-ancestor`), and it has no other changes (`git -C <workspace> status --porcelain`). A mismatch — `BLOCKED` (Integrity Error).
3. **Execute the checks** (section 4) in `workspace`, recording for each the command, the outcome and the evidence.
4. **Check the DoD conditions 1–7** (section 4.4) and the findings, produce the Findings Disposition.
5. **Write `verification-report`** (section 5) and return the envelope (section 6).

## 4. Checks

### 4.1. Checks from three sources (OD-15, `workflow-policies.md` §18)

The checks required by the Testing Strategy, by each touched category of the Story Risk Assessment (additional checks for risky changes) and by the plan's Testing Approach. The tools, kinds and scope come from the Testing Strategy; for Go, for example, the test and vet commands it prescribes, run in `workspace`.

Each check has exactly one state, with evidence:
- `passed` — executed, the command and the output show success;
- `failed` — executed, the output shows a failure;
- `not_executed` — not run, with the reason;
- `blocked` — could not run because of an environment limitation, with the limitation.

A summary of an earlier stage is never evidence. A check without evidence is not `passed`.

### 4.2. Acceptance Criteria (basic compliance check, always mandatory, Q27)

For each `AC-NN` — actual evidence of fulfillment according to its Verification Method (a passed check, an observed response or stored state). If `reconciliation_report` exists, use its traceability and do not build it anew; if RECONCILIATION is `NOT_APPLICABLE`, a direct link "criterion → evidence" is sufficient. An AC without evidence is a `BLOCKING` finding; it cannot become FOLLOW_UP.

### 4.3. Other checks

- **Basic security checks** (Q22), even when SECURITY_REVIEW is not applicable — at least: no secrets in the changes, no passwords or sensitive data in responses and logs in the executed scenarios, the authentication and authorization behavior of the AC.
- **Integration with the Target Base Identity** (OD-12): the checks run on the integration result in `workspace`.
- **Applicability decisions** (`na_checked_by: VERIFICATION`): for each `NOT_APPLICABLE` of SECURITY_REVIEW and RECONCILIATION — the triggers and the evidence are correct, and the Applicability Basis has not changed since the decision (AP-08).
- **Foundation rules** (WF-V2-01): no violation of a mandatory rule is resolved by a `risk-acceptance-record`.
- **Currency and binding** (from `state`): all controlled artifacts `CURRENT`; the review reports are bound to the current versions and the current Changeset Content Identity (RR-10).
- **Human decisions** (DoD 7): each human decision recorded in `history` (`human_decision.applied`) is reflected in the canonical artifact it concerns.

### 4.4. DoD and Findings Disposition

- DoD conditions 1–7 (`workflow-policies.md` §8) — each `met` / `not_met` with evidence; conditions 8–13 are out of this stage.
- Findings — `artifact-rules.md` §6: `F-NNN`, `BLOCKING` / `NON_BLOCKING`, previous `OPEN` findings with a new state. Findings Disposition: for each finding of this Story — `OPEN`, `RESOLVED` (confirmed by this verification), or `DISPOSITIONED` with the human decision or record that supports it; proposed FOLLOW_UP items (they require a human's approval at HUMAN_PR_APPROVAL, OD-11).

## 5. Output

`verification-report` — the record header of `artifact-rules.md` §2 and the fields of the type: `record_type: verification-report`, `record_id`, `story_id`, `produced_by: { stage: VERIFICATION, attempt }`, `created_at`, `subjects` — `{ type: implementation-changeset, version, changeset_commit }`, `verdict`, `findings`, `verified_changeset: { version, changeset_commit }`, `verified_target_base: <commit>`, `dod` (conditions 1–7 with their state), `follow_up` (the proposed items, or `[]`).

Body: 1. Subject and Identities; 2. Summary; 3. Checks (a table: source, check, command, state, evidence); 4. Acceptance Criteria Evidence (`AC-NN` → evidence); 5. Basic Security Checks; 6. Applicability Decisions; 7. Currency, Binding and Human Decisions; 8. Findings; 9. Findings Disposition; 10. DoD State.

In `mode=pilot`, the first line of the body: "Pilot artifact outside the official registered Workflow. Not officially approved and not an input of stages of registered Stories; may be an input of subsequent stages of the same pilot."

## 6. Verdict and Return

VERIFICATION is mandatory (`optional: false`). Loopbacks: `IMPLEMENTATION` (without `crosses_gates`); `IMPLEMENTATION_PLANNING`, `DB_DESIGN`, `API_DESIGN`, `SOLUTION_DESIGN`, `IMPACT_ANALYSIS` (through HUMAN_TECHNICAL_APPROVAL); `SPECIFICATION`, `CLARIFICATION` (through HUMAN_TECHNICAL_APPROVAL and HUMAN_SPEC_APPROVAL). Allowed: **`PASS`**, **`CHANGES_REQUIRED`**, **`BLOCKED`**.

| Situation | Verdict |
|---|---|
| A precondition or an argument is not met, an input cannot be read, an identity does not match, `output` exists | `BLOCKED` before the checks, no file |
| All required checks `passed`, each AC has evidence, DoD 1–7 `met`, no unresolved `BLOCKING` finding | `PASS` |
| A test or check `failed`, an AC without evidence, or a defect in the changes | `CHANGES_REQUIRED`, `loopback_target: IMPLEMENTATION` (or the highest upstream stage where the cause lies) |
| A required check `not_executed` / `blocked` by the environment, a Foundation change or an escalation needed, a mandatory rule violation | `BLOCKED` (Missing Information / Foundation Change Required / Escalation Required / Foundation Rule Violation) |

- `PASS` is impossible while any required check is not `passed` (`stages.md`, Completion Conditions).
- The report is written for every verdict found during the checks: `outputs: [{ record: <record_id> }]`; `outputs: []` only when there were no checks.
- `loopback_target` is only a proposal; gate confirmation belongs to the Orchestrator. In `mode=pilot`, add "pilot" to `summary`.

```yaml
result:
  story_id: US-001
  stage: VERIFICATION
  attempt: 1
  verdict: PASS                      # PASS | CHANGES_REQUIRED | BLOCKED
  outputs:
    - { record: verification-report/0001 }
  loopback_target: null
  blocker: null
  summary: "14/14 checks passed; 41/41 AC with evidence; DoD 1–7 met; target base 9f8e7d6; pilot"
```

## 7. Constraints

- Write only `output`. Run the checks only in `workspace`; do not modify the project repository, branches or history; do not commit, merge, rebase, push or `force`.
- Do not repair code or tests; do not change artifacts, records, approvals, `state.yaml` or `history.jsonl`.
- Do not access production systems, shared databases or secrets; do not create infrastructure or security exceptions.
- Do not ask the human, do not accept risks, do not perform a Loopback or a Human Gate, do not start other stages, Skills or subagents.
