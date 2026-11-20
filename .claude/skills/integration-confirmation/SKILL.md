---
name: integration-confirmation
description: Stage Skill of the INTEGRATION_CONFIRMATION stage (type check). After the human's CONFIRM_MERGED, confirms by fact — with Git and PR evidence of the pull-request repository — that exactly the approved Changeset Content Identity was integrated into the target branch regardless of the merge strategy (Integration Match), compares the target identity at the moment of the merge with the verified one (Base Comparison), performs Post-Merge Validation only if the base changed, and creates a backlog-entry for each FOLLOW_UP the human approved. Writes integration-record, post-merge-validation-record (if performed) and backlog-entry files; returns a result envelope. Never merges, reverts, pushes or fixes integrated code. Started only manually with ready paths, record ids and attempt number.
argument-hint: "story=<ID> approval=<path to state.yaml> history=<path> pull_request=<path> verification=<path> changeset=<path> output=<path> record_id=integration-record/<NNNN> attempt=<N> [pmv_output=<path> pmv_record_id=post-merge-validation-record/<NNNN> workspace=<path> plan=<path> testing_strategy=<path>] [backlog_dir=<path>] [mode=pilot]"
disable-model-invocation: true
user-invocable: true
---

# INTEGRATION_CONFIRMATION — Stage Skill

## 1. Purpose

Confirm by fact that exactly the approved changes have been integrated into the target branch (OD-12 steps 7–8); if the integration was performed on an unchecked base, perform Post-Merge Validation (OD-16); create a `backlog-entry` for approved FOLLOW_UP (OD-11). The Skill executes the stage according to `docs/workflow/stages.md` (section 12), `docs/workflow/stage-map.yaml` (`type: check`, `post_merge_validation`, `next: DONE`) and `docs/workflow/workflow-policies.md` §7, §9. These files and `docs/contracts/` are authoritative.

**How it differs:** VERIFICATION checks the result before the merge; MERGE_READINESS_CHECK re-checks the identities right before it; this stage checks the **merge result** recorded by the last CONFIRM_MERGED.

**Boundaries:** the agent does not change the target branch — no merge, revert, push or fix of integrated code (`workflow-policies.md` §9). Git use is read-only, except fetching remote objects into the local object store (`git fetch <URL> <ref>` into `FETCH_HEAD`, without updating any local branch). The Skill does not change approvals, artifacts, other records, `state.yaml` or `history.jsonl` (read-only inputs), does not approve FOLLOW_UP items, risks or findings, and does not lower finding classes.

**Language** (`AGENTS.md`): records in English; the human's text verbatim.

## 2. Inputs

| Argument | Required | Meaning |
|---|---|---|
| `story` | yes | Story ID |
| `approval`, `history` | yes | `state.yaml` and `history.jsonl`, read-only: `approvals.HUMAN_PR_APPROVAL`, the last `human_decision.applied` HUMAN_MERGE `CONFIRM_MERGED` with its merge facts |
| `pull_request` | yes | The `pull-request` record (`repository`, `number`, `source_branch`, `target_branch`, `changeset`) |
| `verification` | yes | The `verification-report` (`verified_changeset`, `verified_target_base`, `follow_up`, Findings Disposition) |
| `changeset` | yes | The approved `implementation-changeset` (`base_commit`, `changeset_commit`, files) |
| `output`, `record_id` | yes | The `integration-record` path and id |
| `pmv_output`, `pmv_record_id`, `workspace`, `plan`, `testing_strategy` | only for Post-Merge Validation | The `post-merge-validation-record` path and id; a working tree prepared by the caller with the current state of the target branch; `implementation-plan` (Story Risk Assessment, Testing Approach); the Testing Strategy (FG-01) |
| `backlog_dir` | only with approved FOLLOW_UP | The canonical Backlog directory (`docs/backlog/`, PRQ-01) |
| `attempt` | yes | The stage attempt number |
| `mode=pilot` | no | A pilot outside the registered Workflow; then no `backlog-entry` is written to the project Backlog (section 3, step 6) |

Logical inputs — `stage-io-map.yaml` → `INTEGRATION_CONFIRMATION`: `pull-request`, `verification-report`, `implementation-changeset`, `codebase` (the target branch after the merge), and `implementation-plan`, `testing-strategy` for Post-Merge Validation. A missing argument or an unreadable input — `BLOCKED`.

## 3. Procedure

1. **Preconditions.** `output` does not exist. The last routing event is `human_decision.applied` HUMAN_MERGE `CONFIRM_MERGED` (its `merge_commit`, `pr_number`, `repository`, `target_branch` are the merge to evaluate, `merge_binding`); `approvals.HUMAN_PR_APPROVAL` is `VALID` for the Changeset Content Identity of `changeset`. Changes of the target branch after the merge do not affect the approval (`workflow-state-model.md` §7). Otherwise — `BLOCKED`.
2. **The merge was performed.** With every remote query against `pull_request.repository` (`gh ... --repo`, `git ls-remote <URL>`): the PR is `MERGED`, its `mergeCommit` equals the recorded `merge_commit`, and `merge_commit` is reachable from the remote `target_branch`. The PR is not merged, or the merge commit is not on the target branch → `CHANGES_REQUIRED`, `loopback_target: HUMAN_MERGE`. Fetch the needed objects (`git fetch <URL> <target_branch>`, `git fetch <URL> pull/<number>/head` or the source branch).
3. **INTEGRATION_MATCH** (`check_order` 1). Determine the strategy and the integrated range from Git, not from assumptions:
   - **merge commit** — `merge_commit` has two parents, and the second parent is `changeset_commit` (or contains it as the PR head): `target_before_merge` = the first parent; the integrated change = `diff(first parent, merge_commit)`;
   - **squash** — one parent and a single new commit: `target_before_merge` = its parent; the integrated change = `diff(parent, merge_commit)`;
   - **rebase** — the PR's commits were replayed: `N` = the number of the PR's commits (`gh pr view --repo ... --json commits`); the replayed range is `merge_commit~N..merge_commit`, each commit with one parent and authored as the corresponding PR commit; `target_before_merge` = `merge_commit~N`; the integrated change = the cumulative `diff(merge_commit~N, merge_commit)`.

   If the strategy or the range cannot be established reliably, do not guess: `BLOCKED` (Missing Information) with what is missing; nothing about the target identity is recorded.

   Then compare the content, without relying only on SHA: the set of changed paths of the integrated change equals the changeset's files; the patch identity (`git diff ... | git patch-id --stable`) of the integrated change equals that of `diff(base_commit, changeset_commit)`; if the patch identities differ, compare per path — the blob of each changeset path at `merge_commit` equals its blob at `changeset_commit`, and no other path changed. Any unapproved content or a differing approved path → `BLOCKED` (Integration Mismatch); stop, Post-Merge Validation is not performed.
4. **BASE_COMPARISON** (`check_order` 2). `target_before_merge` = `verified_target_base` → Post-Merge Validation is not needed.
5. **POST_MERGE_VALIDATION** (`check_order` 3), only if the base changed. Requires `workspace` (the current remote tip of `target_branch`, which contains `merge_commit`), `plan` and `testing_strategy`; missing — `BLOCKED` (Missing Information: FG-01 or the workspace). Perform the relevant integration checks and tests according to the Testing Strategy, the Story Risk Assessment and the Testing Approach, each with a state `passed` / `failed` / `not_executed` / `blocked` and evidence (as VERIFICATION, section 4.1). Write `post-merge-validation-record` with the result and the checked target state. A `failed` check → `BLOCKED` (Post-Merge Validation Failed). A technical failure to run the checks is not a verdict: report it, so that the Orchestrator records `FAILED`.
6. **FOLLOW_UP_BACKLOG** (`check_order` 4). Only the FOLLOW_UP items that the human approved at HUMAN_PR_APPROVAL (the `follow_up` and the Findings Disposition of the approved `verification-report`). For each, one `backlog-entry` in `backlog_dir` with a reference to the Story and the Finding Identity (`artifact-registry.yaml`, OD-11). If there are approved items and the canonical Backlog is not established (PRQ-01, `workflow-policies.md` §15) or `mode=pilot` — create nothing and return `BLOCKED` (Missing Information: canonical Backlog). No other item is created; a `BLOCKING` finding is never a FOLLOW_UP.
7. **Write `integration-record`** (section 4) and return the envelope (section 5).

## 4. Output

- **`integration-record`** — the record header of `artifact-rules.md` §2 and the fields of the type: `record_type: integration-record`, `record_id`, `story_id`, `produced_by: { stage: INTEGRATION_CONFIRMATION, attempt }`, `created_at`, `subjects` — `{ type: implementation-changeset, version, changeset_commit }` and `{ record: <pull-request id> }`, `verdict`, `merge_commit`, `target_before_merge` (only when established by step 3), `checks` — INTEGRATION_MATCH, BASE_COMPARISON, POST_MERGE_VALIDATION (`performed` / `not_needed`), FOLLOW_UP_BACKLOG, each with its result and evidence (strategy, range, patch identities, the current target tip). The record records facts; it does not by itself prove integration. Written for every verdict reached after step 2.
- **`post-merge-validation-record`** — only if performed: the header, `subjects`, `verdict`, the checked target state (commit), the checks with states and evidence, diagnostics for a failure. Not part of any Approval Scope.
- **`backlog-entry`** — one per approved FOLLOW_UP (step 6).

In `mode=pilot`, the first line of the body of each record: "Pilot artifact outside the official registered Workflow. Not officially approved and not an input of stages of registered Stories; may be an input of subsequent stages of the same pilot."

## 5. Verdict and Return

The stage is mandatory (`optional: false`), `next: DONE`. Loopback: `HUMAN_MERGE` (the merge has not been performed yet). Allowed: **`PASS`**, **`CHANGES_REQUIRED`**, **`BLOCKED`**.

| Situation | Verdict | `outputs` |
|---|---|---|
| A precondition or an argument is not met, an input or the remote cannot be read | `BLOCKED` | `[]` |
| The PR is not merged / the merge commit is not on the target branch | `CHANGES_REQUIRED`, `loopback_target: HUMAN_MERGE` | `[]` |
| Strategy or range not reliably established | `BLOCKED` (Missing Information) | `[{ record: <record_id> }]` |
| Unapproved or differing content | `BLOCKED` (Integration Mismatch) | `[{ record: <record_id> }]` |
| Post-Merge Validation `failed` | `BLOCKED` (Post-Merge Validation Failed) | `[{ record: <record_id> }, { record: <pmv_record_id> }]` |
| Post-Merge Validation needed but impossible; approved FOLLOW_UP without a canonical Backlog | `BLOCKED` (Missing Information) | `[{ record: <record_id> }]` (and the PMV record, if written) |
| Integration Match passed, PMV passed or not needed, every approved FOLLOW_UP has a `backlog-entry` | `PASS` | `[{ record: <record_id> }]`, the PMV record if performed, and `{ type: backlog-entry, id }` for each entry |

- There is no automatic fix of an already performed merge; a `BLOCKED` is resolved by a human through Recovery (`resume_rules`).
- In `mode=pilot`, add "pilot" to `summary`.

```yaml
result:
  story_id: US-001
  stage: INTEGRATION_CONFIRMATION
  attempt: 1
  verdict: PASS                      # PASS | CHANGES_REQUIRED | BLOCKED
  outputs:
    - { record: integration-record/0001 }
  loopback_target: null              # HUMAN_MERGE only for CHANGES_REQUIRED
  blocker: null
  summary: "squash merge 5e6f7a8; INTEGRATION_MATCH pass; base 9f8e7d6 = verified, PMV not needed; no FOLLOW_UP; pilot"
```

## 6. Constraints

- Write only `output`, `pmv_output` (if performed) and the `backlog-entry` files of step 6. Do not modify code, the target branch, other branches, approvals, artifacts, other records, `state.yaml`, `history.jsonl`, the Foundation or `docs/contracts/`.
- Git: read-only commands and `git fetch` into `FETCH_HEAD` only; no `checkout` of the project tree, `merge`, `revert`, `reset`, `push`, `rebase`; platform queries read-only, always against `pull_request.repository`.
- Do not approve FOLLOW_UP items, risks or findings; do not ask the human; do not perform a Loopback, a Human Gate or Recovery; do not start other stages, Skills or subagents.
