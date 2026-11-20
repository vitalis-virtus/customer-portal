---
name: security-review
description: Review Skill of the SECURITY_REVIEW stage. In an isolated context, assesses the applicability of SECURITY_REVIEW on entry against the current Changeset Content Identity (triggers SEC-01…SEC-11, the PLANNED decision may only be expanded), writes an ENTRY applicability-decision and, if applicable, an in-depth security review of the actual Git changes — a review-report bound to the Changeset Content Identity — and returns a result envelope. Does not change code, commit or approve risks. Started only manually with ready paths, record ids and attempt number.
argument-hint: "story=<ID> changeset=<path> design=<path> [api=<path>] [db=<path>] design_review=<path> code_review=<path> planned_decision=<path> output=<path> record_id=review-report/<NNNN> record_output=<path> decision_id=applicability-decision/<NNNN> attempt=<N> [spec=<path>] [mode=pilot] [previous_reports=<path,...>] [reason=\"<the human's decision>\"]"
disable-model-invocation: true
user-invocable: true
context: fork
agent: general-purpose
background: false
allowed-tools: Read, Grep, Glob, Write, Bash(git log:*), Bash(git status:*), Bash(git diff:*), Bash(git show:*), Bash(git rev-parse:*), Bash(git ls-files:*), Bash(git cat-file:*), Bash(git merge-base:*), Bash(date:*)
disallowed-tools: Edit, NotebookEdit, AskUserQuestion, Agent, Skill
---

# SECURITY_REVIEW — Review Skill

## 1. Purpose

An in-depth independent check of the security of the implementation (Q22): the second level of the two-level security check (OD-05); the first level is DESIGN_REVIEW. The Skill executes the SECURITY_REVIEW stage according to `docs/workflow/stages.md` (section 11), `docs/workflow/stage-map.yaml` (`optional: true`, `APPLICABILITY_SECURITY_REVIEW`, `context_isolation: required`, RR-10, RR-14) and `docs/workflow/workflow-policies.md` §4, §5, §7. These files and `docs/contracts/` are authoritative.

The Skill runs in a separate context (`context: fork`, OD-10). Evidence is the actual repository content at the Changeset Content Identity; the `implementation-changeset` description, the earlier reports and the self-assessment of the producer stages are inputs to verify, not proof.

**Boundaries:** the Skill does not change code, tests, Git history, artifacts, records, State or History; does not check out revisions (it reads them with `git show` / `git diff`); does not accept risks or grant exceptions (only a human does, `workflow-policies.md` §7); does not perform Loopbacks or Human Gates.

**Language** (`AGENTS.md`): both files in English; the human's text verbatim.

## 2. Inputs

| Argument | Required | Meaning |
|---|---|---|
| `story` | yes | Story ID |
| `changeset` | yes | The current `implementation-changeset` (`feature_branch`, `base_commit`, `changeset_commit`, changed files) |
| `design` | yes | The current `solution-design` |
| `api`, `db` | no | The current `api-design`, `db-design`, each only when its stage is applicable |
| `design_review`, `code_review` | yes | The DESIGN_REVIEW `review-report` and the CODE_REVIEW `review-report` with `PASS` for the current Changeset Content Identity |
| `planned_decision` | yes | The PLANNED `applicability-decision` of SECURITY_REVIEW from `implementation-plan` |
| `record_output`, `decision_id` | yes | A ready path and `applicability-decision/<NNNN>` of the ENTRY decision |
| `output`, `record_id` | yes | A ready path and `review-report/<NNNN>` of the review, written only if applicable |
| `attempt` | yes | The stage attempt number |
| `spec` | no | `story-spec`, as needed (`stages.md`) |
| `mode=pilot` | no | A pilot outside the registered Workflow |
| `previous_reports` | no | Previous SECURITY_REVIEW reports of this Story |
| `reason` | no | The verbatim reason of a human's decision regarding this stage (SEC-11, a decision after RESUME) |

Logical inputs — `stage-io-map.yaml` → `SECURITY_REVIEW`: `implementation-changeset`, `solution-design`, the DESIGN_REVIEW and CODE_REVIEW `review-report`, the PLANNED `applicability-decision`, `architecture-foundation` (at minimum `security-conventions.md` and `architecture.md` §8; as needed `api-conventions.md`, `persistence-conventions.md`), and `api-design`, `db-design` when applicable. As needed: `story-spec`, NFRs. The working context of the producer stages is not an input.

A missing argument or an unreadable input — `BLOCKED` before the assessment.

## 3. Procedure

1. **Check the arguments and preconditions.** In `mode=pilot`, no output in `docs/stories/`; `output` and `record_output` do not exist. The CODE_REVIEW report has `verdict: PASS`, no `OPEN` `BLOCKING`, and its `subjects` are exactly the current `{ type: implementation-changeset, version, changeset_commit }` (`stages.md`: CODE_REVIEW `PASS` for the current identity). Otherwise — `BLOCKED` before the assessment.
2. **Verify the Git identity** as CODE_REVIEW does: the commits exist; `changeset_commit` is the tip of `feature_branch`; `git merge-base --is-ancestor <base_commit> <changeset_commit>`; the actual `git diff --name-status` matches the changeset's files. A mismatch — `BLOCKED` (Integrity: the reviewed content is not identified).
3. **Assess applicability on entry** (section 3.1) and write the ENTRY `applicability-decision`.
4. **If `NOT_APPLICABLE`** — return the envelope (section 6). No `review-report`.
5. **If `APPLICABLE`** — read the actual diff and the touched files at `changeset_commit`, check according to section 4, classify the findings (section 3.2), write the `review-report` (section 5), return the envelope.

### 3.1. Applicability on Entry

- The triggers — `stage-map.yaml` → `APPLICABILITY_SECURITY_REVIEW`, SEC-01…SEC-11, verbatim; each `met` / `not_met` with a source in the **actual changeset** and the design (`not_applicable_evidence`: for each trigger — why it is not met, with a reference to the changeset and the design).
- The basis is all `basis_facts`: the Changeset Content Identity, the design artifact versions, the Story Type, the Story Risk Assessment of the plan, the findings of DESIGN_REVIEW and CODE_REVIEW (SEC-10: each finding or note marked "SEC-10" makes SEC-10 `met`), human decisions regarding this Review (SEC-11, `reason`), Recovery decisions that changed the Story's risks.
- AP-01 (one `met` → `APPLICABLE`), AP-02 (ambiguous → `met`), AP-03.
- **PLANNED vs ENTRY** (AP-06, AP-10): the PLANNED decision is a forecast. The ENTRY decision is assessed anew against the actual state and **may only expand** applicability: if the PLANNED decision is `APPLICABLE`, the ENTRY decision is `APPLICABLE`; a `NOT_APPLICABLE` PLANNED decision is not reused and may become `APPLICABLE`. The PLANNED record is not changed.
- ENTRY record — `artifact-rules.md` §2: `record_type: applicability-decision`, `record_id: <decision_id>`, `story_id`, `produced_by: { stage: SECURITY_REVIEW, attempt }`, `created_at`, `subjects` — `{ type: implementation-changeset, version, changeset_commit }`, `target_stage: SECURITY_REVIEW`, `kind: ENTRY`, `result`, `triggers`, `basis` (the facts above, including `{ record: <planned decision id> }`); no `verdict` / `findings`. Body: Subject; Triggers (with the AP-02 reasons); Relation to the PLANNED decision.

### 3.2. Findings

- `severity` — `BLOCKING` or `NON_BLOCKING` (`artifact-rules.md` §6). A violation of a mandatory Security Rule is always `BLOCKING` and is not removed by Risk Acceptance (RR-14, WF-V2-01); when in doubt — case B (`workflow-policies.md` §7).
- Each finding: `F-NNN` (stable in the Story; the same problem found by another stage keeps its ID), severity, location (file and line at `changeset_commit`), explanation, evidence, recommendation, and the class of resolution: a fix; a risk that may be accepted by a human (case A); a mandatory rule violation (case B).

## 4. Quality Criteria (if applicable)

Check the actual implementation against the Security Rules of the Foundation (`security-conventions.md`, AR- invariants) and the security decisions of `solution-design`, `api-design`, `db-design`, as far as the changes touch them:

1. Authentication, sessions and tokens (JWT, session store, revocation, cookies).
2. Authorization and "who is acting" (Actor, self-service, administrative checks, deny by default).
3. Credentials, passwords, secrets, keys, cryptography (algorithms, parameters, constant-time comparison, no secrets in code, config or commits).
4. Sensitive data in responses, errors and logs (non-disclosure, prohibited log data).
5. Trust boundaries and untrusted input (decoding, validation, limits, injection, parameterized SQL).
6. Request protection (CSRF, Origin, CORS, security headers) where the changes concern it.
7. Failure behavior (fail closed, unknown outcome, no retries of security-sensitive operations).
8. Dependencies relevant to security (SEC-06) and the CODE_REVIEW SEC-10 notes: each is resolved or turned into a finding.
9. Weakening of any mandatory Security Rule (RR-14).

Do not trust earlier reports: verify each security claim against the code at `changeset_commit`.

## 5. Output (only `APPLICABLE`)

`review-report` — `artifact-rules.md` §2: `record_type: review-report`, `record_id`, `story_id`, `produced_by: { stage: SECURITY_REVIEW, attempt }`, `created_at`, `subjects` — exactly `{ type: implementation-changeset, version, changeset_commit }`, `verdict`, `findings` (each new and each previous `OPEN` one). Body: 1. Subject of the Check; 2. Summary; 3. Findings; 4. Previous Findings; 5. SEC-10 Handling; 6. Checked Criteria (items 1–9).

In `mode=pilot`, the first line of the body of each file: "Pilot artifact outside the official registered Workflow. Not officially approved and not an input of stages of registered Stories; may be an input of subsequent stages of the same pilot."

## 6. Verdict and Return

SECURITY_REVIEW has `optional: true`. Loopbacks: `IMPLEMENTATION` (without `crosses_gates`); `IMPLEMENTATION_PLANNING`, `DB_DESIGN`, `API_DESIGN`, `SOLUTION_DESIGN`, `IMPACT_ANALYSIS` (through HUMAN_TECHNICAL_APPROVAL); `SPECIFICATION`, `CLARIFICATION` (through HUMAN_TECHNICAL_APPROVAL and HUMAN_SPEC_APPROVAL). Allowed: **`PASS`**, **`NOT_APPLICABLE`**, **`CHANGES_REQUIRED`**, **`BLOCKED`**.

| Situation | Verdict | `outputs` |
|---|---|---|
| An argument or precondition is not met, an input cannot be read, an output exists, the Git identity does not match | `BLOCKED` before the assessment | `[]` |
| All triggers `not_met` (and the PLANNED decision is not `APPLICABLE`) | `NOT_APPLICABLE` | `[{ record: <decision_id> }]` |
| `APPLICABLE`, no `OPEN` `BLOCKING` findings | `PASS` | `[{ record: <decision_id> }, { record: <record_id> }]` |
| `APPLICABLE`, `BLOCKING` findings to fix in the code | `CHANGES_REQUIRED`, `loopback_target: IMPLEMENTATION` | the same |
| `APPLICABLE`, findings that need a plan, design or specification change | `CHANGES_REQUIRED`, the highest upstream fitting target | the same |
| A security risk that does not violate a mandatory Security Rule | `BLOCKED` (Escalation Required) — for possible Risk Acceptance by a human | the same |
| A violation of a mandatory Security Rule that the Story cannot fix within the Foundation | `BLOCKED` (Foundation Rule Violation) — a fix, a Foundation Change Request or an allowed exception | the same |

- `CHANGES_REQUIRED` and the review-based `BLOCKED` are possible only with `result: APPLICABLE`.
- `loopback_target` is only a proposal; routing and gate confirmation belong to the Orchestrator.
- In `mode=pilot`, add "pilot" to `summary`.

```yaml
result:
  story_id: US-001
  stage: SECURITY_REVIEW
  attempt: 1
  verdict: PASS                      # PASS | NOT_APPLICABLE | CHANGES_REQUIRED | BLOCKED
  outputs:
    - { record: applicability-decision/0006 }
    - { record: review-report/0006 }
  loopback_target: null
  blocker: null                      # for BLOCKED: { reason, details, needed }
  summary: "APPLICABLE (SEC-01, SEC-02, SEC-04); 0 BLOCKING, 1 NON_BLOCKING; changeset_commit 1a2b3c4; pilot"
```

## 7. Constraints

- Write only `record_output` and, if applicable, `output`. Do not modify code, tests, build files, artifacts, the PLANNED record, the Foundation, `docs/contracts/` or other files.
- Git — only the read commands of `allowed-tools`; no `checkout`, `commit`, `merge`, `rebase`, `reset`, `push` or other writing operations.
- Do not run the application, tests, scanners or migrations; do not access secrets or external systems.
- Do not change `state.yaml`, `history.jsonl`, statuses or approvals. Do not ask the human, do not accept risks, do not perform a Loopback or a Human Gate, do not start other stages, Skills or subagents.
