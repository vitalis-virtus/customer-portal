---
name: code-review
description: Review Skill of the CODE_REVIEW stage. Independently, in an isolated context, checks the actual Git changes of the Story (implementation-changeset bound to its Changeset Content Identity) against the implementation-plan, the applicable design artifacts and the Architecture Foundation, writes a review-report with findings and a verdict, and returns a result envelope. Does not change code, commit, merge or push. Started only manually with ready paths, record_id and attempt number.
argument-hint: "story=<ID> changeset=<path> plan=<path> design=<path> [api=<path>] [db=<path>] output=<path> record_id=review-report/<NNNN> attempt=<N> [mode=pilot] [previous_reports=<path,...>]"
disable-model-invocation: true
user-invocable: true
context: fork
agent: general-purpose
background: false
allowed-tools: Read, Grep, Glob, Write, Bash(git log:*), Bash(git status:*), Bash(git diff:*), Bash(git show:*), Bash(git rev-parse:*), Bash(git ls-files:*), Bash(git cat-file:*), Bash(git merge-base:*), Bash(date:*)
disallowed-tools: Edit, NotebookEdit, AskUserQuestion, Agent, Skill
---

# CODE_REVIEW — Review Skill

## 1. Purpose

Independently check the implementation. The Skill executes the CODE_REVIEW stage according to `docs/workflow/stages.md` (section 10), `docs/workflow/stage-map.yaml` (`reviews: [IMPLEMENTATION]`, `context_isolation: required`, RR-10) and `docs/workflow/workflow-policies.md` §4, §7, §9. These files and `docs/contracts/` are authoritative.

The Skill runs in a separate context (`context: fork`, OD-10). Evidence is the **actual repository content** at the Changeset Content Identity, not the `implementation-changeset` description and not the working context or self-assessment of IMPLEMENTATION. A `PASS` of IMPLEMENTATION is not evidence. The reviewer does not fix the code.

**Boundaries:** the Skill does not change code, tests, build files, artifacts, records, State or History; does not create commits, branches, merges or pushes; does not check out other revisions in the working tree (it reads them with `git show` / `git diff`); does not approve anything; does not perform Loopbacks. The review report is valid only for the reviewed Changeset Content Identity (RR-10, `workflow-policies.md` §4).

**Language** (`AGENTS.md`): `review-report` in English.

## 2. Inputs

| Argument | Required | Meaning |
|---|---|---|
| `story` | yes | Story ID |
| `changeset` | yes | The current `implementation-changeset` (its front matter and body give `feature_branch`, `target_branch`, `base_commit`, `changeset_commit`) |
| `plan` | yes | The current approved `implementation-plan` |
| `design` | yes | The current `solution-design` |
| `api`, `db` | no | The current `api-design`, `db-design`, each only when its stage is applicable |
| `output`, `record_id` | yes | A ready path and `review-report/<NNNN>` |
| `attempt` | yes | The stage attempt number |
| `mode=pilot` | no | A pilot outside the registered Workflow |
| `previous_reports` | no | Previous CODE_REVIEW reports of this Story |

Logical inputs — `stage-io-map.yaml` → `CODE_REVIEW`: `implementation-changeset`, `implementation-plan`, `solution-design`, `architecture-foundation` (at minimum the Architecture Rules, `architecture.md` §8; as needed — `package-map.md`, `api-conventions.md`, `persistence-conventions.md`, `security-conventions.md`), `codebase`, and `api-design`, `db-design` when applicable. `story-spec` is reached only through the `REQ`/`AC` references of the plan; it is not an input.

A missing argument or an unreadable input — `BLOCKED` before the check.

## 3. Procedure

1. **Check the arguments.** In `mode=pilot`, `output` cannot lie in `docs/stories/`. If `output` exists — `BLOCKED`.
2. **Verify the Git identity** of the changeset:
   - `base_commit` and `changeset_commit` exist (`git cat-file -t`); `changeset_commit` is the tip of `feature_branch` (`git rev-parse <feature_branch>`); `base_commit` is an ancestor of `changeset_commit`, checked with `git merge-base --is-ancestor <base_commit> <changeset_commit>` (exit code 0); a non-ancestor result is a Git identity mismatch;
   - the actual changed files (`git diff --name-status <base_commit> <changeset_commit>`) match the "Changed Files" of the changeset exactly.

   A mismatch is a `BLOCKING` finding (the description does not identify the reviewed content). If the commits do not exist — `BLOCKED` before the check.
3. **Read the actual diff** (`git diff <base_commit> <changeset_commit>`) and, where needed, whole files at `changeset_commit` (`git show <changeset_commit>:<path>`). Then the plan, the design and the needed Foundation sections.
4. **Check the previous reports**, if passed (the same `F-NNN` for the same problem; each `OPEN` finding receives a new state).
5. **Check** according to section 4, classify the findings (section 3.1), determine the verdict (section 6), write the report (section 5), return the envelope.

### 3.1. Findings

- Only material problems. `severity` — `BLOCKING` or `NON_BLOCKING` (`artifact-rules.md` §6).
  - `BLOCKING`: a plan step not implemented or an `AC` without its planned test; a change of behavior, interface or schema not in the design; an out-of-scope change; a violation of an AR- invariant, the Package Map rules or a mandatory Security Rule (RR-14); incorrect error handling that breaks the design's error categories; a concurrency or persistence defect against the design's required guarantees; missing or wrong tests for the changes; an unclassified or misclassified dependency (OD-09); a Git identity mismatch; secrets in the changes.
  - `NON_BLOCKING`: improvements that violate no requirement or rule.
  - Do not downgrade an AC or Foundation rule violation; when in doubt — `BLOCKING`.
- Each finding: `F-NNN`, severity, location (file and line at `changeset_commit`, or the artifact section), explanation, evidence, recommendation and the stage that must fix it.
- **SEC-10.** A security decision or a finding that needs an in-depth check of the implementation is explicitly marked "SEC-10" in the report (it is input for the SECURITY_REVIEW applicability).

## 4. Quality Criteria

1. **Plan and design.** Each plan step is implemented; the changes realize the design and nothing else; each change is traceable to a step and a `REQ`/`AC`.
2. **Architecture.** AR- invariants, module boundaries, layers, import directions (Package Map), no new technology outside Architecture §6 (AR-012).
3. **Go correctness and error handling.** Correct logic; errors are translated by the design's categories and the Foundation rules (no adapter error crosses the boundary, AR-008).
4. **Security.** The Security Rules applicable to the changes (authentication, sessions, credentials, sensitive data in logs and responses, input parsing); no secrets in the changes.
5. **Persistence and concurrency**, where the changes touch them: the mechanisms of `db-design` and the guarantees of `solution-design` are implemented as designed.
6. **Tests.** The tests of the plan's Testing Approach exist for the changes and each `AC`; they test the behavior, not only the code path.
7. **Dependencies.** Changes of `go.mod` / `go.sum` match the planned dependencies; unplanned ones are recorded and classified (OD-09).
8. **Scope and identity.** No change outside the Story's scope; the Git identity is consistent (section 3, step 2); the changeset metadata matches `artifact-rules.md` §1.

The Skill does not run the tests or the code; it reviews them. The execution of checks belongs to VERIFICATION.

## 5. Output

One `review-report` file. Front matter — `artifact-rules.md` §2 only: `record_type: review-report`, `record_id`, `story_id`, `produced_by: { stage: CODE_REVIEW, attempt }`, `created_at`, `subjects` — exactly `{ type: implementation-changeset, version, changeset_commit }` (`workflow-state.md` §4: the identity of a changeset), `verdict`, `findings` — `{ id, severity, summary, status }` for each new and each previous `OPEN` finding.

Body: 1. Subject of the Check (branches, `base_commit`, `changeset_commit`, the files); 2. Summary; 3. Findings; 4. Previous Findings; 5. SEC-10 Notes; 6. Checked Criteria (items 1–8).

In `mode=pilot`, the first line of the body is the pilot line: "Pilot artifact outside the official registered Workflow. Not officially approved and not an input of stages of registered Stories; may be an input of subsequent stages of the same pilot."

## 6. Verdict and Return

CODE_REVIEW is mandatory (`optional: false`). Loopbacks: `IMPLEMENTATION` (without `crosses_gates`); `IMPLEMENTATION_PLANNING`, `DB_DESIGN`, `API_DESIGN`, `SOLUTION_DESIGN`, `IMPACT_ANALYSIS` (with `crosses_gates: [HUMAN_TECHNICAL_APPROVAL]`); `SPECIFICATION`, `CLARIFICATION` (with `crosses_gates: [HUMAN_TECHNICAL_APPROVAL, HUMAN_SPEC_APPROVAL]`). Allowed: **`PASS`**, **`CHANGES_REQUIRED`**, **`BLOCKED`**; not `NOT_APPLICABLE`.

| Situation | Verdict |
|---|---|
| No `OPEN` `BLOCKING` findings | `PASS` |
| `BLOCKING` findings in the code or tests | `CHANGES_REQUIRED`, `loopback_target: IMPLEMENTATION` |
| `BLOCKING` findings that need a plan, design or specification change | `CHANGES_REQUIRED`, the highest upstream fitting target |
| A Foundation change or an escalation is required | `BLOCKED` (Foundation Change Required / Escalation Required) |
| An argument is missing, an input cannot be read, the commits do not exist, `output` exists | `BLOCKED` before the check, no file |

- The report is written for `PASS`, `CHANGES_REQUIRED` and a `BLOCKED` found during the check: `outputs: [{ record: <record_id> }]`; otherwise `[]`.
- `loopback_target` is only a proposal; routing and gate confirmation belong to the Orchestrator.
- In `mode=pilot`, add "pilot" to `summary`.

```yaml
result:
  story_id: US-001
  stage: CODE_REVIEW
  attempt: 1
  verdict: CHANGES_REQUIRED          # PASS | CHANGES_REQUIRED | BLOCKED
  outputs:
    - { record: review-report/0005 }
  loopback_target: IMPLEMENTATION
  blocker: null
  summary: "1 BLOCKING (F-001), 2 NON_BLOCKING; changeset_commit 1a2b3c4; pilot"
```

## 7. Constraints

- Write only the `output` file. Do not modify code, tests, build files, artifacts, records, the Foundation, `docs/contracts/` or other files.
- Git — only the read commands listed in `allowed-tools`; no `checkout`, `switch`, `commit`, `merge`, `rebase`, `reset`, `push` or other writing or destructive operations.
- Do not run the application, tests or migrations. Do not change `state.yaml`, `history.jsonl`, statuses or approvals. Do not ask the human, do not approve anything, do not perform a Loopback, do not start other stages, Skills or subagents.
