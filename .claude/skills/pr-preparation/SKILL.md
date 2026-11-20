---
name: pr-preparation
description: Stage Skill of the PR_PREPARATION stage. Prepares the Story's changes for a human's approval — a Pull Request from the feature branch to the target branch with a description, references to the Story, check results and the Findings Disposition — exactly for the Changeset Content Identity and Target Base Identity that VERIFICATION checked, and writes a pull-request record. Returns a result envelope. Does not change the content of the changes, rewrite history, merge or approve. Remote push and PR creation only with the user's explicit permission. Started only manually with ready paths, record_id and attempt number.
argument-hint: "story=<ID> verification=<path> code_review=<path> [security_review=<path>] [reconciliation_report=<path>] spec=<path> changeset=<path> repository=<owner/name> feature_branch=<name> target_branch=<name> output=<path> record_id=pull-request/<NNNN> attempt=<N> remote_permission=granted|absent [mode=pilot]"
disable-model-invocation: true
user-invocable: true
---

# PR_PREPARATION — Stage Skill

## 1. Purpose

Prepare the Story's changes for approval by a human: a PR to the target branch (Q8, FR-01). The Skill executes the PR_PREPARATION stage according to `docs/workflow/stages.md` (section 12), `docs/workflow/stage-map.yaml` (RR-10, RR-12) and `docs/workflow/workflow-policies.md` §7, §9. These files and `docs/contracts/` are authoritative.

**Boundaries:**
- the stage changes only the PR metadata, never the content of the changes (RR-10, RR-12): it does not change code or tests, does not create content commits, does not rewrite, squash or rebase history, and does not change `target_branch`;
- it does not merge (only a human merges, `workflow-policies.md` §9), does not approve the PR, does not approve FOLLOW_UP items and does not lower finding classes;
- a remote push of `feature_branch` and the PR creation are performed only when the invoking command explicitly grants it (`remote_permission=granted`, `AGENTS.md`); otherwise the stage is `BLOCKED` (`stages.md`: no permission or access to the remote repository);
- it does not change artifacts, records, approvals, `state.yaml` or `history.jsonl`.

**Language** (`AGENTS.md`): the PR description and the record in English; the human's text verbatim.

## 2. Inputs

| Argument | Required | Meaning |
|---|---|---|
| `story` | yes | Story ID |
| `verification` | yes | The `verification-report` with `PASS` |
| `code_review` | yes | The CODE_REVIEW `review-report` |
| `security_review`, `reconciliation_report` | no | The SECURITY_REVIEW report and the `reconciliation-report`, if those stages were applicable |
| `spec` | yes | The current approved `story-spec` |
| `changeset` | yes | The current `implementation-changeset` |
| `repository` | yes | The remote repository (`owner/name`) |
| `feature_branch`, `target_branch` | yes | The source and target branches |
| `output`, `record_id` | yes | A ready path and `pull-request/<NNNN>` |
| `attempt` | yes | The stage attempt number |
| `remote_permission` | yes | `granted` only if the user explicitly allowed the push and the PR creation; otherwise `absent` |
| `mode=pilot` | no | A pilot outside the registered Workflow |

Logical inputs — `stage-io-map.yaml` → `PR_PREPARATION`: `verification-report`, `review-report`, `story-spec`, `implementation-changeset`, and the state of the feature branch and the target branch. A missing argument or an unreadable input — `BLOCKED`.

## 3. Procedure

1. **Check the arguments.** `output` does not exist; no output in `docs/stories/` in `mode=pilot`.
2. **Check the precondition** (`stages.md`): `verification` has `verdict: PASS`, no `OPEN` `BLOCKING` finding, and its `verified_changeset` equals the current changeset (`version`, `changeset_commit`). Otherwise — `BLOCKED` (the verification is not for this changeset).
3. **Check the identities against Git:**
   - `changeset_commit` is the tip of `feature_branch` (`git rev-parse`), and the content between `base_commit` and it matches the changeset's files;
   - the **remote** tip of `target_branch` equals `verified_target_base` of the verification report. It is read with a read-only remote query, `git ls-remote <remote> refs/heads/<target_branch>`; the local `target_branch` is not evidence. If the remote cannot be read — `BLOCKED` (External Dependency), nothing is pushed or created.

   If the target branch has moved but the content of the changes is unchanged → `CHANGES_REQUIRED`, `loopback_target: VERIFICATION` (condition `CHANGESET_CONTENT_UNCHANGED`). If the content must change (in particular a conflict with the target branch) → `CHANGES_REQUIRED`, `loopback_target: IMPLEMENTATION`.
4. **Compose the PR description** (section 4) from the registered artifacts and the Git evidence, not from the implementation summary.
5. **If `remote_permission` is not `granted`** — write nothing remote, return `BLOCKED` (Missing Information: permission for the remote repository) with the prepared description in `blocker.details`; no record.
6. **If granted:** repeat the remote check of step 3 immediately before the push (a moved target branch — the routes of step 3, nothing is pushed); push `feature_branch` without force (`git push <remote> <feature_branch>`; a rejected push is reported, never forced); create the PR from `feature_branch` to `target_branch` with the description (for example, `gh pr create`); read its number and head commit back. The PR head commit must equal `changeset_commit` (Completion Conditions); otherwise `BLOCKED` (Integration Mismatch), the PR is left for the human.
7. **Write the `pull-request` record** (section 5) and return the envelope.

## 4. PR Description

- **Story** — ID, title, a link to `story-spec` (version) and the Story Type.
- **Changes** — the scope from the changeset: files and plan steps; `base_commit`, `changeset_commit`, `feature_branch`, `target_branch`.
- **Requirements** — the `AC-NN` and the evidence references of the verification report.
- **Checks** — the verification results (`verified_changeset`, `verified_target_base`, the check states) and the review outcomes (CODE_REVIEW; SECURITY_REVIEW and RECONCILIATION — the result or `NOT_APPLICABLE`).
- **Findings Disposition** — from the verification report, unchanged: every finding with its state; the `NON_BLOCKING` and FOLLOW_UP dispositions. `BLOCKING` findings are never turned into FOLLOW_UP; an unresolved `BLOCKING` finding means the precondition is not met.
- **Proposed FOLLOW_UP** — as proposals that require the human's approval at HUMAN_PR_APPROVAL (OD-11); not approved here.
- **Known limitations and risks** — from the verification report and the review reports.
- **Human decisions required** — HUMAN_PR_APPROVAL, the FOLLOW_UP approvals, and the human merge.

## 5. Output (only after the PR is created)

`pull-request` record — the header of `artifact-rules.md` §2 and the fields of the type: `record_type: pull-request`, `record_id`, `story_id`, `produced_by: { stage: PR_PREPARATION, attempt }`, `created_at`, `subjects` — `{ type: implementation-changeset, version, changeset_commit }` and `{ record: <verification record_id> }`, `repository`, `number`, `source_branch`, `target_branch`, `changeset: { version, changeset_commit }`. The PR metadata (title, description, labels) is not part of the identity (RR-12). Body: 1. Subject; 2. PR (URL, number, head commit); 3. The description as submitted.

In `mode=pilot`, the first line of the body: "Pilot artifact outside the official registered Workflow. Not officially approved and not an input of stages of registered Stories; may be an input of subsequent stages of the same pilot."

## 6. Verdict and Return

PR_PREPARATION is mandatory (`optional: false`). Loopbacks (`stage-map.yaml`): `IMPLEMENTATION` (any need to change the content of the changes) and `VERIFICATION` with the condition `CHANGESET_CONTENT_UNCHANGED` (only the target branch has changed); neither crosses a gate. Allowed: **`PASS`**, **`CHANGES_REQUIRED`**, **`BLOCKED`**.

| Situation | Verdict | `outputs` |
|---|---|---|
| An argument or the VERIFICATION precondition is not met, an input cannot be read | `BLOCKED` | `[]` |
| The target branch moved, the content is unchanged | `CHANGES_REQUIRED`, `loopback_target: VERIFICATION` | `[]` |
| The content must change (a conflict, a defect) | `CHANGES_REQUIRED`, `loopback_target: IMPLEMENTATION` | `[]` |
| No permission or no access to the remote repository; a rejected push; the PR head differs from `changeset_commit` | `BLOCKED` (Missing Information / External Dependency / Integration Mismatch) | `[]` |
| The PR is open and contains exactly the verified Changeset Content Identity | `PASS` | `[{ record: <record_id> }]` |

In `mode=pilot`, add "pilot" to `summary`.

```yaml
result:
  story_id: US-001
  stage: PR_PREPARATION
  attempt: 1
  verdict: PASS                      # PASS | CHANGES_REQUIRED | BLOCKED
  outputs:
    - { record: pull-request/0001 }
  loopback_target: null              # IMPLEMENTATION | VERIFICATION (CHANGESET_CONTENT_UNCHANGED)
  blocker: null
  summary: "PR #12 open; head 1a2b3c4 = verified changeset; target base 9f8e7d6; pilot"
```

## 7. Constraints

- Write only `output`. Do not change code, tests, artifacts, records, approvals, `state.yaml`, `history.jsonl`, the Foundation or `docs/contracts/`.
- Git: no content commits, no history rewriting, no `force`, no change of `target_branch`, no merge; push and PR creation only with `remote_permission=granted`.
- Do not approve the PR or FOLLOW_UP items, do not lower finding classes, do not perform a Loopback or a Human Gate, do not start other stages, Skills or subagents.
