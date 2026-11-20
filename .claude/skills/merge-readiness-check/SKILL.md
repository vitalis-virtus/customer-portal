---
name: merge-readiness-check
description: Stage Skill of the MERGE_READINESS_CHECK stage (type check). Immediately before the merge, checks with actual Git and PR evidence of the repository of the pull-request record that the approved changes can still be integrated without a new check — the feature branch and the PR contain the approved Changeset Content Identity, the target branch has not moved since VERIFICATION — and determines the Merge Protection Status. Writes no files; returns a result envelope. Never merges, pushes or changes protection settings. Started only manually with ready paths and attempt number.
argument-hint: "story=<ID> approval=<path to state.yaml> pull_request=<path> verification=<path> changeset=<path> attempt=<N> [merge_protection_policy=<path>] [mode=pilot]"
disable-model-invocation: true
user-invocable: true
---

# MERGE_READINESS_CHECK — Stage Skill

## 1. Purpose

Immediately before the merge, check that the approved changes can still be integrated without a new check (OD-12 steps 2–5, OD-16, FR-01, FR-02). The Skill executes the stage according to `docs/workflow/stages.md` (section 12), `docs/workflow/stage-map.yaml` (`type: check`, `merge_protection_statuses`, RR-13) and `docs/workflow/workflow-policies.md` §9. These files and `docs/contracts/` are authoritative.

**How it differs:** VERIFICATION checks the result; PR_PREPARATION creates the PR; HUMAN_PR_APPROVAL is the human's approval; HUMAN_MERGE is the human's merge. This stage only re-checks the identities right before the merge.

**Boundaries:** the Skill writes no files (`stage-io-map.yaml`: `outputs: []`; the result and the Merge Protection Status are recorded by the Orchestrator in `state.yaml` → `git` and History). It does not merge, push, rewrite history, change code, artifacts, approvals, protection settings, State or History; it uses only read-only Git and platform queries.

**Language** (`AGENTS.md`): the envelope in English.

## 2. Inputs

| Argument | Required | Meaning |
|---|---|---|
| `story` | yes | Story ID |
| `approval` | yes | `state.yaml` of the Story, read-only: `approvals.HUMAN_PR_APPROVAL`, `git` |
| `pull_request` | yes | The `pull-request` record (`repository`, `number`, `source_branch`, `target_branch`, `changeset`) |
| `verification` | yes | The `verification-report` (`verified_changeset`, `verified_target_base`) |
| `changeset` | yes | The current `implementation-changeset` |
| `merge_protection_policy` | no | The project's Merge Protection Policy (`merge-protection-policy`). While PRQ-02 is open it does not exist: then no mechanism is mandatory, and the `NOT_SUPPORTED` / `UNAVAILABLE` cases apply (`workflow-policies.md` §15) |
| `attempt` | yes | The stage attempt number |
| `mode=pilot` | no | A pilot outside the registered Workflow |

A missing argument or an unreadable input — `BLOCKED` before the check.

## 3. Procedure

1. **Approval.** `approvals.HUMAN_PR_APPROVAL` exists with `decision: APPROVE` and `approval_state: VALID`, and its `scope` contains the `pull-request` and `verification-report` records and the changeset identity passed here. Otherwise — `BLOCKED` (the precondition is not met).
2. **Repository identity.** Every remote query targets `pull_request.repository` explicitly: `gh ... --repo <repository>`, `git ls-remote <URL of repository> ...`. If the repository cannot be reached or identified — `BLOCKED` (External Dependency).
3. **The three conditions** (`stages.md`), with actual evidence:
   1. the remote feature branch contains the approved Changeset Content Identity: `git ls-remote <URL> refs/heads/<source_branch>` equals `changeset.changeset_commit`;
   2. the PR corresponds to the approved identity: `gh pr view <number> --repo <repository> --json state,headRefOid,baseRefName,headRefName` — open, `headRefOid` = `changeset_commit`, `baseRefName` = `target_branch`, `headRefName` = `source_branch`;
   3. the target branch has not changed: `git ls-remote <URL> refs/heads/<target_branch>` equals `verification.verified_target_base`.
4. **Merge Protection Status** — `ACTIVE`, `NOT_SUPPORTED`, `UNAVAILABLE` or `DISABLED` (`merge_protection_statuses`), from read-only platform queries (for example, `gh api repos/<repository>/branches/<target_branch>/protection`) and the Merge Protection Policy. A mechanism available but not mandatory and disabled is `UNAVAILABLE`. If the status cannot be determined, it is not assumed `ACTIVE`.
5. **Determine the verdict** (section 4) and return the envelope.

## 4. Verdict and Return

The stage is mandatory (`optional: false`). Loopbacks (all cross `HUMAN_PR_APPROVAL`): `VERIFICATION` with the condition `CHANGESET_CONTENT_UNCHANGED`, `PR_PREPARATION`, `IMPLEMENTATION`. Allowed: **`PASS`**, **`CHANGES_REQUIRED`**, **`BLOCKED`**. `outputs` is always `[]`.

| Situation | Verdict |
|---|---|
| The approval is not `VALID`, an input or the remote cannot be read | `BLOCKED` |
| Conditions 1–3 met and the status is `ACTIVE`, or `NOT_SUPPORTED` (FR-02), or `UNAVAILABLE` with an integration order approved by a human (recorded in State and History) | `PASS` |
| The target branch moved, the content of the changes is the same | `CHANGES_REQUIRED`, `loopback_target: VERIFICATION` (`CHANGESET_CONTENT_UNCHANGED`) |
| The PR does not correspond to the approved identity (another head, base or branch; a closed PR) | `CHANGES_REQUIRED`, `loopback_target: PR_PREPARATION` |
| Integration requires changing the content of the changes (for example, the feature branch contains another commit) | `CHANGES_REQUIRED`, `loopback_target: IMPLEMENTATION` |
| `UNAVAILABLE` without a human-approved integration order | `BLOCKED` (Merge Protection Unavailable) |
| `DISABLED` under a mandatory policy, without a valid `foundation-exception-record` | `BLOCKED` (Merge Protection Disabled) |

- `summary` begins exactly with `merge_protection_status: <STATUS>;`, where `<STATUS>` is one of `ACTIVE`, `NOT_SUPPORTED`, `UNAVAILABLE`, `DISABLED` (uppercase, no other value); if the status could not be determined, the verdict is not `PASS`. The Orchestrator accepts the result only if this prefix parses to a valid value. Then `summary` lists the three conditions with their evidence (commits, PR number). For `NOT_SUPPORTED`, it states the limitation for the HUMAN_MERGE package. This is the only channel for the status, because the envelope has no other fields (`workflow-state.md` §6).
- `loopback_target` is only a proposal; every return requires a human's decision (LOOPBACK_CONFIRMATION).
- In `mode=pilot`, add "pilot" to `summary`.

```yaml
result:
  story_id: US-001
  stage: MERGE_READINESS_CHECK
  attempt: 1
  verdict: PASS                      # PASS | CHANGES_REQUIRED | BLOCKED
  outputs: []
  loopback_target: null              # VERIFICATION (CHANGESET_CONTENT_UNCHANGED) | PR_PREPARATION | IMPLEMENTATION
  blocker: null                      # for BLOCKED: { reason, details, needed }
  summary: "merge_protection_status: ACTIVE; feature head 1a2b3c4 = changeset; PR #12 open, head 1a2b3c4, base main; target main 9f8e7d6 = verified; pilot"
```

## 5. Constraints

- Write no files. Read-only Git and platform queries only, always against `pull_request.repository`.
- No merge, push, rebase, history rewriting, PR edits or protection-setting changes (`workflow-policies.md` §9: the agent does not change protection settings without a human's permission).
- Do not change `state.yaml`, `history.jsonl`, artifacts, records or approvals; do not ask the human, approve anything, perform a Loopback or a Human Gate, or start other stages, Skills or subagents.
