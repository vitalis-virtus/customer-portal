---
name: implementation
description: Stage Skill of the IMPLEMENTATION stage. Implements the approved implementation-plan within the approved technical design — code and tests in the Story's feature branch — and writes an implementation-changeset that describes the changes (feature branch, base_commit, changeset_commit, changed files, unplanned dependencies, test results). Returns a result envelope. Does not change requirements, the design, the plan or Workflow State. Started only manually with ready paths, the feature branch and the attempt number.
argument-hint: "story=<ID> plan=<path> spec=<path> design=<path> [api=<path>] [db=<path>] feature_branch=<name> target_branch=<name> output=<path to implementation-changeset v<N>.md> version=<N> attempt=<N> [commit=allowed] [mode=pilot] [findings=<path,...>] [reason=\"<the human's decision>\"]"
disable-model-invocation: true
user-invocable: true
---

# IMPLEMENTATION — Stage Skill

## 1. Purpose

Implement the approved plan: code and tests (Q3). The Skill executes the IMPLEMENTATION stage according to `docs/workflow/stages.md` (section 10), `docs/workflow/stage-map.yaml` (RR-10) and `docs/workflow/workflow-policies.md` §2, §9. These files and `docs/contracts/` are authoritative.

IMPLEMENTATION is the only code-modifying stage and the only stage that creates a new Changeset Content Identity (RR-10). Its document output describes the change; the change itself lives in Git (`artifact-registry.yaml` → `implementation-changeset`).

**Stage boundaries:**
- implement only what the approved `implementation-plan` and design define; technical decisions inside these boundaries are the agent's (Controlled Autonomy, `workflow-policies.md` §2);
- do not expand the scope, do not add requirements, do not silently change the design or the plan: a deviation is a return or an escalation (section 6);
- do not change `story-spec`, the design artifacts, the plan, the applicability records, the review reports, the approvals, the Foundation, the Workflow Definition, `docs/contracts/`, `docs/stories/`, `docs/pilots/` (except `output`), `state.yaml` or `history.jsonl`;
- the result is checked by CODE_REVIEW (`reviewed_by`).

**Language** (`AGENTS.md`): code comments, commit messages and `implementation-changeset` in English; the human's text verbatim.

## 2. Inputs

| Argument | Required | Meaning |
|---|---|---|
| `story` | yes | Story ID |
| `plan` | yes | The current `implementation-plan`, approved at HUMAN_TECHNICAL_APPROVAL |
| `spec`, `design` | yes | The current approved `story-spec` and `solution-design` |
| `api`, `db` | no | The current `api-design`, `db-design`, each only when its stage is applicable. If absent, do not use the artifact |
| `feature_branch` | yes | The Story's feature branch, created and checked out by the caller |
| `target_branch` | yes | The target branch. It is never modified by this stage |
| `output` | yes | A ready path of the `implementation-changeset` file |
| `version` | yes | Its version, matches `output` |
| `attempt` | yes | The stage attempt number |
| `commit=allowed` | no | The user's explicit permission for `git add <paths>` and `git commit` in `feature_branch` for this invocation only (`AGENTS.md`). Without it, no commit is made (section 3, step 7). It never permits push, merge, rebase, reset or branch operations |
| `mode=pilot` | no | A pilot outside the registered Workflow. The pilot root `docs/pilots/orchestrator/<story>/` holds Orchestrator-owned State, History, artifacts and records: they are not code changes |
| `findings` | no | On re-entry: the review reports whose findings caused the return (`stages.md`, "On re-entry") |
| `reason` | no | The verbatim reason of the human's decision that returned or resumed the stage |

Logical inputs — `stage-io-map.yaml` → `IMPLEMENTATION`: `implementation-plan`, `story-spec`, `solution-design`, `architecture-foundation` (the rules the changes concern — at minimum `architecture.md` §8, `package-map.md`, and the conventions of the touched areas), `codebase`, and `api-design`, `db-design` when applicable. `impact-analysis`, `clarification-record` and chat history are not inputs.

There is no human in the context: the Skill asks nothing. A missing argument or an unreadable input — `BLOCKED` (section 6).

## 3. Procedure

1. **Check the preconditions.** The current branch is `feature_branch` (`git rev-parse --abbrev-ref HEAD`), not `target_branch`; the working tree has no changes outside this Story's scope (`git status --porcelain -- . ':(exclude)<story root>'`, where the story root is `docs/stories/<story>/`, or the pilot root in `mode=pilot`); `output` does not exist. Otherwise — `BLOCKED`, change nothing.
2. **Record `base_commit`** — for `version: 1`, `git rev-parse HEAD` before any change; for `version > 1`, the `base_commit` of the previous version, so that the changeset always describes the Story's cumulative change.
3. **Read the inputs** and, on re-entry, each finding of `findings`: each is fixed or explained.
4. **Implement the plan steps in their order.** For each step: the code within the module, layer and import rules of the Package Map and the AR- invariants; the tests defined by the plan's Testing Approach, Test-First where the plan says so. Names, files and structure follow the Foundation conventions.
5. **Dependencies.** A dependency that the design and the plan already contain is introduced as planned. An unplanned dependency is recorded in the changeset; a significant or ambiguous one (OD-09) — stop: `BLOCKED` (Escalation Required).
6. **Run the tests related to the changes locally.** All must pass (`stages.md`, Completion Conditions). A failure that cannot be fixed within the design is a return (section 6).
7. **Create the commits in `feature_branch` only.** Never commit to `target_branch`, never merge, never push, never rewrite history that is not this Story's (`workflow-policies.md` §9). `AGENTS.md` requires an explicit permission of the user for `git commit`: commit only with `commit=allowed`; stage only explicit code, test and build paths — never the story or pilot root, never `git add -A`, `git add .` or `git commit -a`; otherwise stop before the commit and return `BLOCKED` (Missing Information: commit permission), leaving the changes uncommitted.
8. **Record `changeset_commit`** — `git rev-parse HEAD` after the last commit.
9. **Write `implementation-changeset`** (section 5) and return the envelope. For a verdict other than `PASS`, write no `output`.

## 4. Quality Criteria

- Each plan step is implemented; each change is traceable to a step, a design element and a `REQ`/`AC`.
- Each `AC` has the test or check that the plan assigns to it; the tests related to the changes pass locally.
- No change outside the plan, no new requirement, no change of an approved technical decision.
- AR- invariants, Package Map rules and the Security Rules are not violated or weakened (RR-14).
- Unplanned dependencies are recorded; significant ones are escalated.
- No secrets in code, tests or commits.

## 5. Output

`implementation-changeset` at `output`, in English. Front matter — `artifact-rules.md` §1 only: `artifact_type: implementation-changeset`, `story_id`, `version`, `produced_by: { stage: IMPLEMENTATION, attempt }`, `created_at`, `inputs` (`implementation-plan`, `story-spec`, `solution-design`, the passed `api-design` / `db-design`, the Foundation documents read as `{ source, path, commit }`; for `version > 1` — the previous version and `revision_reason`).

Body sections:
1. **Subject** — the inputs and versions.
2. **Changeset** — `feature_branch`, `target_branch`, `base_commit`, `changeset_commit`, the commits.
3. **Changed Files** — each file: added / changed / removed, the plan step.
4. **Traceability** — plan step → files → `REQ`/`AC` → tests.
5. **Tests** — the tests written and the local run result (command and outcome).
6. **Dependencies** — planned ones introduced; unplanned ones with their OD-09 classification.
7. **Handling of Findings** — only on re-entry.
8. **Deviations and Limitations** — what could not be done as planned and why (without such a deviation, the verdict is not `PASS`).
9. **Human Decision** — only if `reason` is passed, verbatim.

In `mode=pilot`, the first line of the body is the pilot line of the Stage Skills: "Pilot artifact outside the official registered Workflow. Not officially approved and not an input of stages of registered Stories; may be an input of subsequent stages of the same pilot."

## 6. Verdict and Return

IMPLEMENTATION has `optional: false`. All its loopbacks (`IMPLEMENTATION_PLANNING`, `DB_DESIGN`, `API_DESIGN`, `SOLUTION_DESIGN`, `IMPACT_ANALYSIS`, `SPECIFICATION`, `CLARIFICATION`) cross HUMAN_TECHNICAL_APPROVAL (SPECIFICATION and CLARIFICATION also HUMAN_SPEC_APPROVAL). Allowed: **`PASS`**, **`CHANGES_REQUIRED`**, **`BLOCKED`**.

| Situation | Verdict | `outputs` |
|---|---|---|
| A precondition or an argument is not met, an input cannot be read | `BLOCKED` before any change | `[]` |
| The plan is implemented, the tests pass, the commits are in `feature_branch` | `PASS` | `[{ type: implementation-changeset, version }]` |
| The approved plan or design cannot be implemented as approved | `CHANGES_REQUIRED`, the stage that must change it (the highest upstream fitting one) | `[]` |
| A significant dependency, a Foundation change or another escalation is needed; no commit permission | `BLOCKED` (Escalation Required / Foundation Change Required / Missing Information) | `[]` |

- For `CHANGES_REQUIRED` and `BLOCKED`, describe in `summary` / `blocker` what was changed in the working tree and what remains uncommitted; do not discard work silently and do not commit partial work.
- `loopback_target` is only a proposal; every target crosses a Human Gate, so the Orchestrator asks the human.
- In `mode=pilot`, add "pilot" to `summary`.

Envelope — `docs/contracts/workflow-state.md` §6, unchanged:

```yaml
result:
  story_id: US-001
  stage: IMPLEMENTATION
  attempt: 1
  verdict: PASS                      # PASS | CHANGES_REQUIRED | BLOCKED
  outputs:
    - { type: implementation-changeset, version: 1 }
  loopback_target: null
  blocker: null                      # for BLOCKED: { reason, details, needed }
  summary: "9/9 steps; 41/41 AC covered by tests; tests pass locally; changeset_commit 1a2b3c4; pilot"
```

## 7. Constraints

- Modify only the code, tests and build files that the plan's steps require, and write only `output`. Do not touch the files listed in section 1 as forbidden.
- Git: work only in `feature_branch`; commits only with the user's explicit permission (`AGENTS.md`); no push, no merge, no `target_branch` change, no `force`, no destructive operations.
- Do not run migrations against shared databases, do not access production systems or secrets.
- Do not change `state.yaml`, `history.jsonl`, statuses or approvals. Do not ask the human, do not perform a Loopback or a Human Gate, do not start other stages, Skills or subagents.
