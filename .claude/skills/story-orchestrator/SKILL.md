---
name: story-orchestrator
description: Story Orchestrator MVP (Phase 6, increments 6.4.1–6.4.3; Phase 7, increments 7.4.1–7.4.2; Phase 8, increments 8.3.2, 8.4 and the API_DESIGN, DB_DESIGN, DESIGN_REVIEW, IMPLEMENTATION_PLANNING and PLAN_REVIEW integration; IMP-D01 with IMPLEMENTATION and CODE_REVIEW; the Quality Pipeline SECURITY_REVIEW, RECONCILIATION, VERIFICATION; the Delivery Pipeline PR_PREPARATION, MERGE_READINESS_CHECK, INTEGRATION_CONFIRMATION → DONE). Initializes a pilot Story outside the registered Workflow, shows its status with a State and History check, executes one stage per run (CLARIFICATION inline, SPECIFICATION, SPEC_REVIEW, IMPACT_ANALYSIS, SOLUTION_DESIGN, API_DESIGN, DB_DESIGN, DESIGN_REVIEW, IMPLEMENTATION_PLANNING, PLAN_REVIEW, IMPLEMENTATION and CODE_REVIEW in a subagent, then SECURITY_REVIEW, RECONCILIATION, VERIFICATION, PR_PREPARATION, MERGE_READINESS_CHECK and INTEGRATION_CONFIRMATION) in Step Mode up to DONE; remote push and PR creation only with allow_remote=PR_PREPARATION, the merge only by the human; IMPLEMENTATION runs only in a dedicated pilot worktree and commits only with allow_commit=IMPLEMENTATION; supports re-entry into IMPACT_ANALYSIS after a Loopback from SOLUTION_DESIGN, applies the human's decisions at HUMAN_SPEC_APPROVAL (APPROVE, REQUEST_CHANGES, CANCEL), at HUMAN_TECHNICAL_APPROVAL and HUMAN_PR_APPROVAL (APPROVE, CANCEL) and at the Loopback confirmation from IMPACT_ANALYSIS, SOLUTION_DESIGN, API_DESIGN, DB_DESIGN, DESIGN_REVIEW, IMPLEMENTATION_PLANNING and PLAN_REVIEW (RESUME, CANCEL), and the minimal Recovery RESUME of a Story BLOCKED at SOLUTION_DESIGN with Escalation Required (section 8.11). Started only manually with mode=pilot.
argument-hint: "init|status|next story=<ID> mode=pilot [allow_commit=IMPLEMENTATION] [allow_remote=PR_PREPARATION repository=<owner/name>] [workspace=<path>] | decide story=<ID> mode=pilot decision=APPROVE|REQUEST_CHANGES|CANCEL|RESUME|ROUTE_BACK|CONFIRM_MERGED by=<id> [target=<stage>] [reason=\"...\"] [comment=\"...\"] [evidence=<path>]"
disable-model-invocation: true
user-invocable: true
---

# Story Orchestrator — MVP (increments 6.4.1–6.4.3, 7.4.1–7.4.2, 8.3.2, 8.4, API_DESIGN, DB_DESIGN, DESIGN_REVIEW, IMPLEMENTATION_PLANNING, PLAN_REVIEW, IMP-D01: IMPLEMENTATION, CODE_REVIEW; SECURITY_REVIEW, RECONCILIATION, VERIFICATION; PR_PREPARATION, MERGE_READINESS_CHECK, INTEGRATION_CONFIRMATION)

## 1. Purpose

The Orchestrator maintains the Workflow State and History of one Story: `state.yaml` and `history.jsonl` are changed **only** by it (`docs/contracts/workflow-state.md` §1). Routes, verdicts, loopbacks and gates are defined only by `docs/workflow/stage-map.yaml`; the state format — by `docs/contracts/workflow-state.md`; paths — by `docs/contracts/artifact-registry.yaml`. These files are authoritative; in case of a discrepancy, they apply, not this Skill.

The Skill works in the main session.

**Language** (`AGENTS.md`):
- everything addressed to the human — explanations, status output, requests for decisions, reports — is in Ukrainian;
- everything written to files — `state.yaml`, `history.jsonl`, `cancellation-record` and the generated Story artifacts — is in English;
- text provided by the human (`reason`, `comment`, `by`, interview answers) is recorded verbatim in its original language (section 7.10).

**Boundaries.** Commands: `init`, `status`, `next`, `decide`.

- `next` executes one stage per run (Step Mode): CLARIFICATION, SPECIFICATION, SPEC_REVIEW, IMPACT_ANALYSIS, SOLUTION_DESIGN, API_DESIGN, DB_DESIGN, DESIGN_REVIEW, IMPLEMENTATION_PLANNING, PLAN_REVIEW, IMPLEMENTATION, CODE_REVIEW, SECURITY_REVIEW, RECONCILIATION, VERIFICATION, PR_PREPARATION, MERGE_READINESS_CHECK or INTEGRATION_CONFIRMATION.
- After `PASS` of SPEC_REVIEW, the Story stops at HUMAN_SPEC_APPROVAL and waits for the human's decision.
- After `PASS` or `NOT_APPLICABLE` of IMPACT_ANALYSIS, the Story moves to SOLUTION_DESIGN.
- After `PASS` of SOLUTION_DESIGN, the Story moves to API_DESIGN.
- After `PASS` or `NOT_APPLICABLE` of API_DESIGN, the Story moves to DB_DESIGN.
- After `PASS` or `NOT_APPLICABLE` of DB_DESIGN, the Story moves to DESIGN_REVIEW.
- After `PASS` of DESIGN_REVIEW, the Story moves to IMPLEMENTATION_PLANNING.
- After `PASS` of IMPLEMENTATION_PLANNING, the Story moves to PLAN_REVIEW. IMPLEMENTATION_PLANNING and PLAN_REVIEW require the Testing Strategy: while FG-01 is open, `next` stops at them (section 7.2, steps 10 and 11).
- After `PASS` of PLAN_REVIEW, the Story stops at HUMAN_TECHNICAL_APPROVAL and waits for the human's decision (`decide`, section 8.8). After `APPROVE`, the Story moves to IMPLEMENTATION.
- IMPLEMENTATION and CODE_REVIEW run only under the pilot execution model IMP-D01 (section 7.11): a dedicated linked worktree on `pilot/<story_id>`, commits only with `next ... allow_commit=IMPLEMENTATION`. After `PASS` of IMPLEMENTATION, the Story moves to CODE_REVIEW (a fresh isolated reviewer); after `PASS` of CODE_REVIEW, to SECURITY_REVIEW → RECONCILIATION → VERIFICATION (section 7.12); after `PASS` of VERIFICATION, to PR_PREPARATION → HUMAN_PR_APPROVAL → MERGE_READINESS_CHECK → HUMAN_MERGE → INTEGRATION_CONFIRMATION → `DONE` (section 7.13).
- **IMPLEMENTATION and CODE_REVIEW are executable under IMP-D01** (section 7.11).
- **SECURITY_REVIEW, RECONCILIATION and VERIFICATION are executable** (section 7.12). VERIFICATION needs a human-prepared verification workspace (section 7.12.3).
- **PR_PREPARATION, MERGE_READINESS_CHECK and INTEGRATION_CONFIRMATION are executable** (section 7.13). PR_PREPARATION needs the explicit remote permission `allow_remote=PR_PREPARATION repository=<owner/name>`; the merge is performed only by the human (section 8.10); INTEGRATION_CONFIRMATION needs a human-prepared workspace of the current target tip and ends the Story in `DONE` (section 7.13.4).
- `decide` applies the human's decisions:
  - `APPROVE`, `REQUEST_CHANGES` or `CANCEL` at HUMAN_SPEC_APPROVAL (section 8);
  - `APPROVE` or `CANCEL` at HUMAN_TECHNICAL_APPROVAL; `REQUEST_CHANGES` there is rejected without changes, because re-entry into its targets is not implemented (section 8.8);
  - `APPROVE` or `CANCEL` at HUMAN_PR_APPROVAL, with the same rejection of `REQUEST_CHANGES` (section 8.9). The gate is reached after PR_PREPARATION `PASS` (section 7.13.1). `APPROVE` does not merge: the merge is HUMAN_MERGE;
  - `RESUME` or `CANCEL` at the Loopback confirmation (`LOOPBACK_CONFIRMATION`), when IMPACT_ANALYSIS, SOLUTION_DESIGN, API_DESIGN, DB_DESIGN, DESIGN_REVIEW, IMPLEMENTATION_PLANNING or PLAN_REVIEW returned `CHANGES_REQUIRED` with a target through a gate (section 8.7). `ROUTE_BACK` is rejected without changes (section 8.7.2);
  - the minimal Recovery `RESUME` of a Story `BLOCKED` at SOLUTION_DESIGN with the reason Escalation Required, after the escalation decision is recorded in the committed Foundation (section 8.11).
- Re-entry after a return is supported:
  - for SPECIFICATION — after a loopback from SPEC_REVIEW or REQUEST_CHANGES at the gate (section 7.3.1);
  - for IMPACT_ANALYSIS — after a loopback from SOLUTION_DESIGN (section 7.3.2).
- `REQUEST_CHANGES` with `target=CLARIFICATION` is rejected without changes (section 8.4.1).
- A Loopback without a gate from API_DESIGN (to SOLUTION_DESIGN or IMPACT_ANALYSIS) or from DB_DESIGN, DESIGN_REVIEW, IMPLEMENTATION_PLANNING or PLAN_REVIEW (to IMPLEMENTATION_PLANNING, DB_DESIGN, API_DESIGN, SOLUTION_DESIGN or IMPACT_ANALYSIS) is routed, but the target is not executed (section 7.8, API-D01, DB-D01).
- Not implemented: execution of other stages, re-entry into CLARIFICATION and SOLUTION_DESIGN with a new version, re-entry into IMPACT_ANALYSIS after a Loopback from API_DESIGN or DB_DESIGN, re-entry into API_DESIGN after `PASS` (only `api-design` v1), re-entry into DB_DESIGN after `PASS` (only `db-design` v1), other Human Gates, RECONFIRM, a Loopback through a gate (`ROUTE_BACK`), Recovery other than the minimal Recovery `RESUME` of section 8.11 (SOLUTION_DESIGN, Escalation Required), and Auto Mode. If the human asks for something else, explain that it does not exist yet, and stop.

## 2. Inputs

Arguments (`$ARGUMENTS`):

| Argument | Required | Meaning |
|---|---|---|
| command | yes | `init`, `status`, `next` or `decide` |
| `story` | yes | Story ID, format `US-NNN` |
| `mode=pilot` | yes | An explicit pilot marker. Without it, the Skill does not work: registration of real Stories is not supported |
| `decision` | only for `decide` | The human's decision at a gate or at a Loopback confirmation (section 8.1) |
| `by` | only for `decide` | The identifier of the human who made the decision (section 8.1) |
| `target` | only for `decide` with `REQUEST_CHANGES` | The target stage of the return (section 8.1) |
| `reason` | only for `decide` with `REQUEST_CHANGES`, `CANCEL` or `RESUME` | The reason for the decision (section 8.1) |
| `comment` | no, only for `decide` with `APPROVE` | The human's comment on the decision (section 8.1) |
| `evidence=<path>` | only for `decide` with Recovery `RESUME`, mandatory there | The repository path of the committed canonical Foundation document that records the escalation decision (section 8.11) |
| `workspace=<path>` | only for `next` at VERIFICATION and INTEGRATION_CONFIRMATION, mandatory there | The human-prepared workspace (sections 7.12.3, 7.13.4). Not stored in `state.yaml` |
| `allow_remote=PR_PREPARATION`, `repository=<owner/name>` | only for `next` at PR_PREPARATION, both mandatory there | The human's explicit permission to push `pilot/<story_id>` (without force) and create one PR in exactly this repository, for that one attempt (section 7.13.1). Not stored, not inferred from `git remote`, not granted by `allow_commit` |
| `allow_commit=IMPLEMENTATION` | only for `next` at IMPLEMENTATION | The human's explicit permission for the commits of that one IMPLEMENTATION attempt (section 7.11.2). Not allowed with another command or stage |

If an argument is missing or different, stop and explain. Do not guess.

## 3. Pilot Storage

The pilot lies **outside** `docs/stories/` and is never presented as a registered or approved Story.

- The pilot root: `docs/pilots/orchestrator/<story_id>/`.
- The structure under the root is the same as in the registry under `storage_root`; only the prefix changes `docs/stories/<story_id>/` → `docs/pilots/orchestrator/<story_id>/`:

  ```
  docs/pilots/orchestrator/<story_id>/
  ├── state.yaml
  ├── history.jsonl
  ├── artifacts/<artifact_type>/v<N>.md
  └── records/<record_type>/<NNNN>.md
  ```

- **The artifact path** = the `path` template from `artifact-registry.yaml` with the replaced prefix. For example, `clarification-record` v1 → `docs/pilots/orchestrator/US-001/artifacts/clarification-record/v1.md`.
- `artifacts/` and `records/` are not created during `init`.
- Any pilot path in `docs/stories/` is an error: stop.
- The pilot marker is the path. Fields that are not in the contract (`mode`, etc.) are not in `state.yaml`.
- Files of other pilots (in particular the manual `docs/pilots/US-001/`) are not inputs and are not changed.

## 4. Command `init`

1. **Check the arguments** (section 2).
2. **Check the Story source.** `story` is a Candidate Story in `docs/product/epic-map.md` (`story-source`). There is no Backlog yet (PRQ-01), so there are no other sources. Not found — stop.
3. **Check that the pilot does not exist yet.** If `docs/pilots/orchestrator/<story_id>/` **exists in any form** (even without `state.yaml`) — stop, list the files found and ask the human for a decision. Do not overwrite, move or delete existing files.
4. **Prerequisites.** FG-01 (Testing Strategy) blocks the registration of the first **real** Story (`workflow-policies.md` §15). A pilot is not a registration, so FG-01 does not block it, but it is not considered fulfilled either. Tell the human about it.
5. **Take the time:** `date -u +%Y-%m-%dT%H:%M:%SZ`. One value for `updated_at` and `ts`.
6. **Write `state.yaml`** (section 4.1).
7. **Write `history.jsonl`** with one event (section 4.2).
8. **Show the result** with the `status` command (section 5), without additional changes.

Steps 6–7 are one Orchestrator step (`workflow-state.md` §7). If writing the second file failed, do not fix it silently: tell the human that the pilot is inconsistent.

### 4.1. `state.yaml` during init

The stage values are from `stage-map.yaml` → `initial_state`. All top-level keys from `workflow-state.md` §3 are present; what is not yet defined is `null` or empty.

```yaml
# PILOT — outside the registered Workflow. Not a registered and not an approved Story.
story_id: US-001
story_type: null                    # until story-spec is registered
story_status: ACTIVE                # initial_state.story_status
current_stage: CLARIFICATION        # initial_state.stage
execution_state: READY              # initial_state.execution_state
last_stage_verdict: null
execution_mode: STEP                # Auto Mode is not supported
updated_at: <date -u>

artifacts: {}

records:
  review-report: {}
  applicability-decision: {}
  revalidation-record: []

approvals: {}

pending_human_decision: null

blocker: null

retry:
  technical_attempts: {}
  quality_loops:
    target: {}
    initiator: {}
  repeated_failure: {}
  extensions: []

git:                                # set at the first IMPLEMENTATION attempt (section 7.11): pilot/<story_id>, main
  feature_branch: null
  target_branch: null
  changeset_commit: null
  verified_target_base: null
  merge_protection_status: null
```

### 4.2. Initialization Event

For the pilot, `story.registered` (`workflow-state.md` §5) is used with an explicit marker that the Story is not registered in the Workflow. This decision is approved for an isolated MVP pilot. One JSON line:

```json
{"ts":"<date -u>","story_id":"US-001","actor":{"kind":"orchestrator","id":"story-orchestrator"},"event":"story.registered","stage":"CLARIFICATION","refs":[],"details":{"mode":"pilot","registered_in_workflow":false,"note":"Pilot state initialized outside the registered Workflow; not a registered or approved Story","story_status":"ACTIVE","execution_state":"READY","execution_mode":"STEP"}}
```

## 5. Command `status`

Read-only. No file changes — even if an inconsistency is found.

1. **Check the arguments** (section 2) and that `docs/pilots/orchestrator/<story_id>/state.yaml` and `history.jsonl` exist. If not — stop.
2. **Read** both files completely.
3. **Check consistency** (section 6).
4. **Show** (in Ukrainian):
   - the Story ID and the marker «PILOT — не зареєстрована в офіційному Workflow і офіційно не погоджена. Рішення Human Gates у пілоті симульовані»;
   - `story_status`, `current_stage`, `execution_state`, `last_stage_verdict`, `execution_mode`, `story_type`, `updated_at`. For `CANCELLED` — «скасовано в пілоті» and `records.cancellation-record`;
   - the current versions of artifacts with `status` and `currency`, the valid records from `records`, in particular `applicability-decision` by stage with its `result`. Next to `APPROVED` — «у пілоті». If the valid IMPACT_ANALYSIS decision is `NOT_APPLICABLE`, and `impact-analysis` is registered, mark it «історична версія, не є входом» (section 7.1);
   - `approvals` under the heading «Рішення пілоту (симульований Human Gate), не офіційне погодження Workflow»: for each gate — `decision`, `decided_by`, `decided_at`, `scope`, `approval_state`;
   - `pending_human_decision`, `blocker`, non-zero `retry.technical_attempts` and `retry.quality_loops`. For `kind: LOOPBACK_CONFIRMATION` — also the proposed target (`loopback_target` of the last `stage.completed` event), `crosses_gates` and the decisions available in the MVP (`RESUME`, `CANCEL`; section 8.7);
   - if the last routing event is `stage.completed` API_DESIGN, DB_DESIGN, DESIGN_REVIEW, IMPLEMENTATION_PLANNING or PLAN_REVIEW with `verdict: CHANGES_REQUIRED` and a `to_stage` without `crosses_gates` (`DB_DESIGN`, `API_DESIGN`, `SOLUTION_DESIGN` or `IMPACT_ANALYSIS`) — the marker «повернення з <stage> без підтримки повторного входу в MVP: `next` цю стадію не виконає» (section 7.8, API-D01, DB-D01);
   - the number of History events and the last event (`ts`, `event`, `stage`);
   - the check result: «узгоджено» or a list of discrepancies.

If there is a discrepancy: according to `workflow-state.md` §7 and `workflow-state-model.md` §12, the Story should have moved to `BLOCKED` (Integrity Error). The current version of the Skill **does not record** this. Tell the human what exactly diverged, where, and that the human resolves the discrepancy. Do not fix it.

## 6. Validation

Check only what the sources define directly. Each discrepancy — with a reference to the file and field.

**`state.yaml`** (`workflow-state.md` §3):

- all top-level keys from section 4.1 are present;
- `story_id` matches the argument and the directory name;
- `story_status` ∈ `ACTIVE | WAITING_FOR_HUMAN | BLOCKED | DONE | CANCELLED`;
- `execution_state` ∈ `READY | RUNNING | AWAITING_HUMAN | COMPLETED | INTERRUPTED | FAILED | INVALID_RESULT`;
- `execution_mode` ∈ `STEP | AUTO`;
- `current_stage` is a Stage ID from `stage-map.yaml` → `stages`;
- `last_stage_verdict` is `null` or one of `stage_verdicts`;
- `story_type` is `null` or one of `story_types`; if `story-spec` is registered, it matches its `story_type`;
- for each artifact: `status` ∈ `DRAFT | REVIEWED | APPROVED`, `currency` ∈ `CURRENT | STALE`.

**Mutual conditions** (`workflow-state-model.md` §3, §4):

- `story_status: WAITING_FOR_HUMAN` ⇔ `pending_human_decision` is not `null` ⇔ `execution_state: AWAITING_HUMAN`;
- `story_status: BLOCKED` ⇔ `blocker` is not `null`;
- `story_status: CANCELLED` ⇔ `records.cancellation-record` is not empty; then `pending_human_decision: null`, `execution_state: COMPLETED`, and the last History event is `story.cancelled` with this record in `refs` (section 8.5.3).

**`approvals`** (`workflow-state.md` §3, §4). If `approvals.HUMAN_SPEC_APPROVAL` is not `null`:

- there are the fields `decision`, `decided_by`, `decided_at`, `scope`, `package`, `comment`, `approval_state`, `previous`; `decision: APPROVE`; `approval_state` ∈ `VALID | SUSPENDED | INVALIDATED`;
- for `approval_state: VALID`, each element of `scope` is the current version of its type in `artifacts` with `status: APPROVED`;
- History has a `human_decision.applied` event of the `HUMAN_SPEC_APPROVAL` stage with `details.decision: APPROVE`, whose `ts` equals `decided_at`.

**`history.jsonl`** (`workflow-state.md` §5):

- each line is valid JSON with the fields `ts`, `story_id`, `actor`, `event`, `stage`, `refs`, `details`;
- `story_id` matches the state;
- `event` is from the event table of §5;
- `actor.kind` ∈ `agent | human | orchestrator`;
- `ts` does not decrease;
- the first event is the initialization event (section 4.2);
- `execution_state: RUNNING` ⇔ the last event is `stage.started` or `human_decision.applied` for `current_stage`. The exception is `human_decision.applied` with `details.decision: RESUME` (`LOOPBACK_CONFIRMATION`, section 8.7.3, or `RECOVERY`, section 8.11): after it, `execution_state: READY`;
- `execution_state: AWAITING_HUMAN` ⇔ the last event is `human_decision.requested` for `current_stage`.

**Files** (`workflow-state-model.md` §12, step 2):

- each version from `artifacts` exists at the pilot path, and its front matter has the same `artifact_type`, `version` and `story_id`;
- each record from `records` exists, and its front matter has the same `record_id` and `story_id`. For `records.applicability-decision`, this is each value `<STAGE>: { ENTRY: <record_id> }`; its front matter also has `target_stage: <STAGE>` and `kind: ENTRY`.

**Git Binding** is not checked while all `git` fields are `null` (before IMPLEMENTATION). Say so in the status. Otherwise: `git.feature_branch: pilot/<story_id>`, `git.target_branch: main`; a non-null `git.changeset_commit` exists (`git cat-file -t`) and equals the `changeset_commit` of the current `implementation-changeset` file; `status` also shows the current branch and whether the session runs in the pilot worktree (section 7.11.1, steps 1–2), without changing anything.

## 7. Command `next`

Executes one attempt of the current stage and stops.

### 7.1. Supported Stages

| Stage | Stage Skill | Where it runs | Output | Input artifacts |
|---|---|---|---|---|
| `CLARIFICATION` | `.claude/skills/clarification/SKILL.md` | inline, main session (interview with the human) | `clarification-record` | — |
| `SPECIFICATION` | `.claude/skills/specification/SKILL.md` | a separate `general-purpose` subagent | `story-spec` | `clarification-record` |
| `SPEC_REVIEW` | `.claude/skills/spec-review/SKILL.md` | a separate `general-purpose` subagent | the `review-report` record | `story-spec`, `clarification-record` |
| `IMPACT_ANALYSIS` | `.claude/skills/impact-analysis/SKILL.md` | a separate `general-purpose` subagent | the `applicability-decision` record (always, when the assessment is completed) and `impact-analysis` (only `PASS`) | `story-spec` |
| `SOLUTION_DESIGN` | `.claude/skills/solution-design/SKILL.md` | a separate `general-purpose` subagent | `solution-design` | `story-spec`; `impact-analysis` — only if IMPACT_ANALYSIS is applicable (section 7.2, step 7) |
| `API_DESIGN` | `.claude/skills/api-design/SKILL.md` | a separate `general-purpose` subagent | the `applicability-decision` record (always, when the assessment is completed) and `api-design` v1 (only `PASS`) | `story-spec`, `solution-design`. `impact-analysis` is never passed: it is not an input of the stage (`stage-io-map.yaml` → `API_DESIGN`) |
| `DB_DESIGN` | `.claude/skills/db-design/SKILL.md` | a separate `general-purpose` subagent | the `applicability-decision` record (always, when the assessment is completed) and `db-design` v1 (only `PASS`) | `story-spec`, `solution-design`; `impact-analysis` — only if IMPACT_ANALYSIS is applicable (section 7.2, step 7); `api-design` — only if API_DESIGN is applicable (section 7.2, step 8) |
| `DESIGN_REVIEW` | `.claude/skills/design-review/SKILL.md` | a separate `general-purpose` subagent (isolated context, OD-10) | the `review-report` record | `story-spec`, `solution-design`, the three `applicability-decision` records of IMPACT_ANALYSIS, API_DESIGN and DB_DESIGN; `impact-analysis`, `api-design`, `db-design` — each only if its stage is applicable (section 7.2, steps 7–9) |
| `PLAN_REVIEW` | `.claude/skills/plan-review/SKILL.md` | a separate `general-purpose` subagent (always isolated, OD-10) | the `review-report` record | `implementation-plan`, the two PLANNED `applicability-decision` records, `story-spec`, `solution-design`, `testing-strategy` (section 7.2, step 11); `impact-analysis`, `api-design`, `db-design` — each only if applicable (section 7.2, steps 7–9) |
| `IMPLEMENTATION` | `.claude/skills/implementation/SKILL.md` | a separate `general-purpose` subagent in the pilot worktree (section 7.11) | `implementation-changeset` (`PASS` only) and commits in `pilot/<story_id>` | `implementation-plan`, `story-spec`, `solution-design`; `api-design`, `db-design` — only if applicable |
| `CODE_REVIEW` | `.claude/skills/code-review/SKILL.md` | a separate fresh `general-purpose` subagent (isolated context, OD-10, section 7.11) | the `review-report` record | `implementation-changeset`, `implementation-plan`, `solution-design`; `api-design`, `db-design` — only if applicable |
| `SECURITY_REVIEW`, `RECONCILIATION` | `.claude/skills/security-review/SKILL.md`, `.claude/skills/reconciliation/SKILL.md` | a fresh `general-purpose` subagent (isolated, section 7.12) | the ENTRY `applicability-decision` and, if applicable, `review-report` / `reconciliation-report` | section 7.12.2 |
| `VERIFICATION` | `.claude/skills/verification/SKILL.md` | a fresh `general-purpose` subagent; checks run in the verification workspace (section 7.12.3) | the `verification-report` record | section 7.12.2 |
| `PR_PREPARATION` | `.claude/skills/pr-preparation/SKILL.md` | a `general-purpose` subagent (section 7.13.1) | the `pull-request` record (`PASS` only), a pushed branch and an open PR | section 7.13.1 |
| `MERGE_READINESS_CHECK` | `.claude/skills/merge-readiness-check/SKILL.md` | a `general-purpose` subagent (section 7.13.2) | none (`outputs: []`); the Merge Protection Status | section 7.13.2 |
| `INTEGRATION_CONFIRMATION` | `.claude/skills/integration-confirmation/SKILL.md` | a `general-purpose` subagent (section 7.13.4) | `integration-record`, `post-merge-validation-record` (if performed) | section 7.13.4 |
| `IMPLEMENTATION_PLANNING` | `.claude/skills/implementation-planning/SKILL.md` | a separate `general-purpose` subagent | `implementation-plan` v1 and two `applicability-decision` records with `kind: PLANNED` (SECURITY_REVIEW, RECONCILIATION) — only `PASS` | `story-spec`, `solution-design`, the DESIGN_REVIEW `review-report`, `testing-strategy` (section 7.2, step 10); `impact-analysis`, `api-design`, `db-design` — each only if its stage is applicable (section 7.2, steps 7–9) |

The output and the owner are from `artifact-registry.yaml` (`produced_by`), the inputs — from `stage-io-map.yaml`. SPEC_REVIEW checks both inputs (`stage-map.yaml` → `reviews: [CLARIFICATION, SPECIFICATION]`). `codebase`, `architecture-foundation` and `product-foundation` for IMPACT_ANALYSIS, SOLUTION_DESIGN, API_DESIGN and DB_DESIGN are sources, not registered artifacts: the Stage Skill reads them itself (`impact-analysis/SKILL.md` §2, `solution-design/SKILL.md` §2, `api-design/SKILL.md` §2, `db-design/SKILL.md` §2). Allowed verdicts and routes — only from `stage-map.yaml` (section 7.6).

**Conditional input `impact-analysis`.** The condition `when: applicable` (`stage-io-map.yaml`) is determined by the valid `applicability-decision` of the stage, that is, `records.applicability-decision.IMPACT_ANALYSIS.ENTRY` with `result: APPLICABLE`, not by the existence of the artifact. If the valid decision is `NOT_APPLICABLE`, a previously registered version of `impact-analysis` remains in `artifacts` as a historical one, but it is not an input of any stage, and the Orchestrator does not pass it (section 7.2, step 7).

### 7.2. Pre-Run Check

1. **Check the arguments** and consistency (section 6). A discrepancy — stop, as in `status`.
2. **`story_status` must be `ACTIVE`.** Otherwise — show the state and stop. For `pending_human_decision.kind: GATE` at HUMAN_SPEC_APPROVAL or HUMAN_TECHNICAL_APPROVAL and for `kind: LOOPBACK_CONFIRMATION` at IMPACT_ANALYSIS, SOLUTION_DESIGN, API_DESIGN, DB_DESIGN, DESIGN_REVIEW, IMPLEMENTATION_PLANNING, PLAN_REVIEW, IMPLEMENTATION, CODE_REVIEW, SECURITY_REVIEW, RECONCILIATION, VERIFICATION or MERGE_READINESS_CHECK, at `GATE` HUMAN_PR_APPROVAL and at `HUMAN_ACTION` HUMAN_MERGE, say that the human makes the decision with the `decide` command (sections 8, 8.7). For `CANCELLED` — the Story is cancelled in the pilot, the state is terminal and is not reopened. For `BLOCKED` at SOLUTION_DESIGN with the reason Escalation Required — say that, after the escalation decision is recorded in the committed Foundation, the human may apply the Recovery `RESUME` with `decide` (section 8.11); `next` does not unblock the Story. Other human decisions and other Recovery are not implemented yet.
   A special case — `WAITING_FOR_HUMAN` with `pending_human_decision.kind: CLARIFICATION_ANSWERS`: a new `next` run means that the interview of this attempt was interrupted together with the session. **MVP limitation:** continuing the interview in a new session is not supported, and the contract does not define a transition from `AWAITING_HUMAN` to `INTERRUPTED` during session recovery (`workflow-state-model.md` §12 describes only `RUNNING`). Change nothing: explain this to the human and stop.
3. **`current_stage` must be in table 7.1.** Another stage — "not supported yet", stop. For PR_PREPARATION, MERGE_READINESS_CHECK and INTEGRATION_CONFIRMATION, steps 6–11 are replaced by section 7.13. For IMPLEMENTATION and CODE_REVIEW, steps 6–11 are replaced by section 7.11 (7.11.1–7.11.3); steps 7–9 still determine `api` and `db`. For SECURITY_REVIEW, RECONCILIATION and VERIFICATION, steps 6, 10–11 are replaced by section 7.12 (steps 7–9 determine `api`, `db`, `impact`).
4. **`execution_state`:**
   - `READY`, `INTERRUPTED`, `FAILED`, `INVALID_RESULT` — a new attempt can be started;
   - `RUNNING` — the previous attempt was interrupted without a result (`workflow-state-model.md` §12, step 3). Record the `INTERRUPTED` failure (section 7.7) and stop. The human starts the new attempt with the next `next`;
   - anything else — stop and explain.
5. **Input artifacts** (`stage-io-map.yaml`, rule R1). For each input from table 7.1 (the conditional `impact-analysis` and `api-design` — only if they are passed according to steps 7 and 8):
   - the type is registered in `state.yaml` → `artifacts`, `currency: CURRENT`, the file of the current version exists;
   - each upstream artifact from its `inputs` (`type`, `version`, a type different from the type of the input itself) is the current version of that type in `artifacts` (`artifact-rules.md` §4);
   - a reference to the previous version of the same type is lineage, not an upstream dependency (`artifact-rules.md` §3): allowed only for `version > 1`, only to `version - 1`, and the file of that version exists at the pilot path. It does not have to be current: it is `SUPERSEDED`;
   - each source from its `inputs` (`source`, `path`, `commit`) still has the same `git log -1 --format=%h -- <path>` (`artifact-rules.md` §4).

   Not met — change nothing, explain to the human and stop. **MVP limitation:** the current version of the Skill does not record the `staleness.detected` event and the transition to `BLOCKED` (Upstream Changed).
6. **Approval and entry trigger (IMPACT_ANALYSIS, SOLUTION_DESIGN, API_DESIGN, DB_DESIGN, DESIGN_REVIEW, IMPLEMENTATION_PLANNING and PLAN_REVIEW).** The precondition of these stages is an approved `story-spec` (`stages.md`, IMPACT_ANALYSIS and SOLUTION_DESIGN; `workflow-state-model.md` §6, step 4). For API_DESIGN, DB_DESIGN, DESIGN_REVIEW, IMPLEMENTATION_PLANNING and PLAN_REVIEW, `solution-design` must also be current: it is checked as an input in step 5:
   - `approvals.HUMAN_SPEC_APPROVAL` is not `null`, `decision: APPROVE`, `approval_state: VALID`;
   - the current version of `story-spec` in `artifacts` has `status: APPROVED` and matches the `story-spec` version in `approvals.HUMAN_SPEC_APPROVAL.scope`;
   - **the entry trigger** is the last History event with `details.to_stage`, as in section 7.3.1. Its `to_stage` must equal `current_stage`. Allowed triggers:

     | Stage | Trigger |
     |---|---|
     | IMPACT_ANALYSIS | `human_decision.applied` HUMAN_SPEC_APPROVAL with `decision: APPROVE` (first entry); `stage.completed` SOLUTION_DESIGN with `verdict: CHANGES_REQUIRED` and `loopback_target: IMPACT_ANALYSIS` (re-entry, section 7.3.2); `human_decision.applied` IMPACT_ANALYSIS with `decision: RESUME` (section 8.7.3) |
     | SOLUTION_DESIGN | `stage.completed` IMPACT_ANALYSIS with `verdict: PASS` or `NOT_APPLICABLE`; `human_decision.applied` SOLUTION_DESIGN with `decision: RESUME` |
     | API_DESIGN | `stage.completed` SOLUTION_DESIGN with `verdict: PASS`; `human_decision.applied` API_DESIGN with `decision: RESUME` |
     | DB_DESIGN | `stage.completed` API_DESIGN with `verdict: PASS` or `NOT_APPLICABLE`; `human_decision.applied` DB_DESIGN with `decision: RESUME` |
     | DESIGN_REVIEW | `stage.completed` DB_DESIGN with `verdict: PASS` or `NOT_APPLICABLE`; `human_decision.applied` DESIGN_REVIEW with `decision: RESUME` |
     | IMPLEMENTATION_PLANNING | `stage.completed` DESIGN_REVIEW with `verdict: PASS`; `human_decision.applied` IMPLEMENTATION_PLANNING with `decision: RESUME` |
     | PLAN_REVIEW | `stage.completed` IMPLEMENTATION_PLANNING with `verdict: PASS`; `human_decision.applied` PLAN_REVIEW with `decision: RESUME` |

     Another trigger means a route that this MVP does not execute. In particular, `stage.completed` API_DESIGN, DB_DESIGN, DESIGN_REVIEW, IMPLEMENTATION_PLANNING or PLAN_REVIEW with `verdict: CHANGES_REQUIRED` and a `loopback_target` without `crosses_gates` (section 7.8, API-D01, DB-D01): explain that re-entry into the target after a Loopback from that stage is not implemented in the MVP, and that the Story cannot continue with `next`;
   - SOLUTION_DESIGN: `artifacts.solution-design` is absent. The MVP does not support re-entry into SOLUTION_DESIGN with a new version (`solution-design/SKILL.md` §3, step 1);
   - API_DESIGN: `artifacts.api-design` is absent. The MVP supports only `api-design` v1 (`api-design/SKILL.md` §3, step 1);
   - DB_DESIGN: `artifacts.db-design` is absent. The MVP supports only `db-design` v1 (`db-design/SKILL.md` §3, step 1);
   - IMPLEMENTATION_PLANNING: `artifacts.implementation-plan` is absent (only v1);
   - PLAN_REVIEW: `artifacts.implementation-plan` v1 is registered and `CURRENT` (step 5).

   Not met — change nothing, explain to the human and stop.
7. **IMPACT_ANALYSIS applicability (SOLUTION_DESIGN, DB_DESIGN, DESIGN_REVIEW, IMPLEMENTATION_PLANNING and PLAN_REVIEW).** Determines whether `impact-analysis` is passed (section 7.1, "Conditional input"):
   - `records.applicability-decision.IMPACT_ANALYSIS.ENTRY` exists, and the record file exists;
   - its `subjects` is the current version of `story-spec`;
   - the last `stage.completed` event of the IMPACT_ANALYSIS stage has `refs` with this `<record_id>` and a `verdict` consistent with the record's `result`: `PASS` ⇔ `APPLICABLE`, `NOT_APPLICABLE` ⇔ `NOT_APPLICABLE`;
   - **`APPLICABLE`:**
     - `artifacts.impact-analysis` exists;
     - the `refs` of the same event contain `impact-analysis@<version>` of the current version in `artifacts`, that is, the artifact was created by the same assessment;
     - step 5 is performed for it (`currency: CURRENT`, the file exists);
     - `impact` = the path of this version;
   - **`NOT_APPLICABLE`:**
     - `impact` is not passed;
     - if `artifacts.impact-analysis` exists (a historical version from a previous applicable assessment), it is not an input: do not pass it and do not check it according to step 5.

   Any other combination is a state discrepancy. Change nothing, explain to the human what exactly does not match (file, field, event), and stop.
8. **API_DESIGN applicability (DB_DESIGN, DESIGN_REVIEW, IMPLEMENTATION_PLANNING and PLAN_REVIEW).** Determines whether `api-design` is passed, by the same rules as step 7:
   - `records.applicability-decision.API_DESIGN.ENTRY` exists, and the record file exists;
   - its `subjects` are the current versions of `story-spec` and `solution-design`;
   - the last `stage.completed` event of the API_DESIGN stage has `refs` with this `<record_id>` and a `verdict` consistent with the record's `result`: `PASS` ⇔ `APPLICABLE`, `NOT_APPLICABLE` ⇔ `NOT_APPLICABLE`;
   - **`APPLICABLE`:** `artifacts.api-design` exists; the `refs` of the same event contain `api-design@1`; step 5 is performed for it; `api` = the path of this version;
   - **`NOT_APPLICABLE`:** `api` is not passed; an `api-design` file, if any, is not an input.

   Any other combination is a state discrepancy. Change nothing, explain to the human what exactly does not match (file, field, event), and stop.
9. **DB_DESIGN applicability (DESIGN_REVIEW, IMPLEMENTATION_PLANNING and PLAN_REVIEW).** Determines whether `db-design` is passed, by the same rules as step 8, with `records.applicability-decision.DB_DESIGN.ENTRY`, the last `stage.completed` event of DB_DESIGN, `artifacts.db-design` and `db-design@1`; `APPLICABLE` → `db` = the path of `db-design` v1, `NOT_APPLICABLE` → `db` is not passed. For DESIGN_REVIEW, steps 7–9 also give the three record paths `ia_decision`, `api_decision`, `db_decision`: all three records must exist, otherwise it is a state discrepancy.

   Any other combination is a state discrepancy. Change nothing, explain to the human what exactly does not match (file, field, event), and stop.
10. **IMPLEMENTATION_PLANNING inputs (only IMPLEMENTATION_PLANNING).**
   - **The DESIGN_REVIEW report:** `records.review-report.DESIGN_REVIEW` exists, its file exists, it is in the `refs` of the last `stage.completed` DESIGN_REVIEW event with `verdict: PASS`, and its `subjects` are exactly the current `solution-design`, the conditional artifacts passed according to steps 7–9 and the three valid `applicability-decision` ENTRY records (section 7.6.5). Otherwise — a state discrepancy: stop;
   - **The Testing Strategy** (`stage-io-map.yaml` → `IMPLEMENTATION_PLANNING`: `testing-strategy`, a mandatory input): `artifact-registry.yaml` → `sources.testing-strategy.path` is not `null`, and the file exists. Otherwise (FG-01, `workflow-policies.md` §15) — change nothing, explain to the human that the stage cannot start without the Testing Strategy, which is created by separate work outside the Story Workflow, and stop. No `stage.started` event, no state change.
11. **PLAN_REVIEW inputs (only PLAN_REVIEW).**
   - **The PLANNED records:** `records.applicability-decision.SECURITY_REVIEW.PLANNED` and `RECONCILIATION.PLANNED` exist, their files exist, and both are in the `refs` of the last `stage.completed` IMPLEMENTATION_PLANNING event with `verdict: PASS` together with `implementation-plan@1`. Otherwise — a state discrepancy: stop;
   - **The Testing Strategy:** the same check as in step 10 (`stage-io-map.yaml` → `PLAN_REVIEW`: `testing-strategy`, a mandatory input). Missing — stop without `stage.started` and without state changes.

### 7.3. Attempt Parameters

Computed by the Orchestrator, not by the Stage Skill:

| Parameter | How it is determined |
|---|---|
| `attempt` | The number of `stage.started` events for `current_stage` in History + 1 |
| `version` | For a document: the current version of the output type in `artifacts` + 1; if there is none — `1` |
| `record_id` | For a record: `<record_type>/<NNNN>`, where `record_type` is `review-report` (SPEC_REVIEW, DESIGN_REVIEW and PLAN_REVIEW; one numbering of the type for the whole Story) or `applicability-decision` (IMPACT_ANALYSIS, API_DESIGN, DB_DESIGN and IMPLEMENTATION_PLANNING; one numbering of the type for the whole Story), and `NNNN` is the next number of the type in the Story (`artifact-rules.md` §2): the highest number `<record_type>/…` in the `refs` of the `stage.completed` events of History + 1, four digits; if there are none — `0001` |
| `output` | The path of the output type according to the registry with the pilot prefix (section 3). For SPEC_REVIEW, DESIGN_REVIEW and PLAN_REVIEW — the record `records/review-report/<NNNN>.md`. For IMPACT_ANALYSIS — the document `artifacts/impact-analysis/v<version>.md`. For SOLUTION_DESIGN — `artifacts/solution-design/v<version>.md`. For API_DESIGN — `artifacts/api-design/v1.md`. For DB_DESIGN — `artifacts/db-design/v1.md`. For IMPLEMENTATION_PLANNING — `artifacts/implementation-plan/v1.md` |
| `record_output` | Only IMPACT_ANALYSIS, API_DESIGN and DB_DESIGN: `records/applicability-decision/<NNNN>.md` under the pilot root |
| `clarification`, `spec` | The paths of the current versions of `clarification-record` and `story-spec` (for SPECIFICATION — only `clarification`; for IMPACT_ANALYSIS, SOLUTION_DESIGN, API_DESIGN, DB_DESIGN, DESIGN_REVIEW, IMPLEMENTATION_PLANNING and PLAN_REVIEW — only `spec`) |
| `design` | Only API_DESIGN, DB_DESIGN, DESIGN_REVIEW, IMPLEMENTATION_PLANNING and PLAN_REVIEW: the path of the current version of `solution-design` |
| `api` | Only DB_DESIGN, DESIGN_REVIEW, IMPLEMENTATION_PLANNING and PLAN_REVIEW, and only according to section 7.2, step 8 (`APPLICABLE`): the path of `api-design` v1 |
| `db` | Only DESIGN_REVIEW, IMPLEMENTATION_PLANNING and PLAN_REVIEW, and only according to section 7.2, step 9 (`APPLICABLE`): the path of `db-design` v1 |
| `plan`, `sec_decision`, `rec_decision` | Only PLAN_REVIEW: the path of `implementation-plan` v1 and of the two PLANNED records (section 7.2, step 11). `testing_strategy` — as for IMPLEMENTATION_PLANNING |
| `review`, `testing_strategy` | Only IMPLEMENTATION_PLANNING: the path of `records.review-report.DESIGN_REVIEW` and the path `sources.testing-strategy.path` (section 7.2, step 10) |
| `sec_record_id`, `sec_record_output`, `rec_record_id`, `rec_record_output` | Only IMPLEMENTATION_PLANNING: two `applicability-decision` numbers by the general rule — `<N>` for SECURITY_REVIEW and `<N+1>` for RECONCILIATION — and their paths `records/applicability-decision/<NNNN>.md`. A new attempt receives new numbers |
| `ia_decision`, `api_decision`, `db_decision` | Only DESIGN_REVIEW: the paths of the valid records `records.applicability-decision.<STAGE>.ENTRY` (section 7.2, steps 7–9) |
| `impact` | Only SOLUTION_DESIGN, DB_DESIGN, DESIGN_REVIEW, IMPLEMENTATION_PLANNING and PLAN_REVIEW, and only according to section 7.2, step 7 (`APPLICABLE`): the path of the current version of `impact-analysis` |
| `previous_reports` | For SPEC_REVIEW, DESIGN_REVIEW and PLAN_REVIEW: the paths of the previous `review-report` files of the same stage from History (`stage.completed` of that stage); if there are none — do not pass |
| `previous`, `reason` | SPECIFICATION: only for `version > 1` — according to section 7.3.1. IMPACT_ANALYSIS — according to section 7.3.2 |
| `reason` (SOLUTION_DESIGN, API_DESIGN, DB_DESIGN, IMPLEMENTATION_PLANNING) | Only if the entry trigger (section 7.2, step 6) is `human_decision.applied` of the current stage with `details.decision: RESUME` (section 8.7.3). The value is the `details.reason` of this event verbatim. Otherwise do not pass. `version` is always `1` |

For API_DESIGN and DB_DESIGN, each attempt receives a new `record_id` and `record_output` by the general rule, in particular after `CHANGES_REQUIRED` and `RESUME`: the previous record is already registered. `output` stays `v1`, because `api-design` / `db-design` is created only for `PASS`.

`focus` for SPEC_REVIEW is not passed: the Orchestrator does not suggest hypotheses or findings.

#### 7.3.1. Re-Entry After a Return (`version > 1`)

Only SPECIFICATION is supported. **The trigger** is the last History event with `details.to_stage`, that is, the last routing event: `stage.completed` or `human_decision.applied` of a gate decision. Attempt events (`stage.started`, `stage.failed`) do not change the trigger. The trigger has `to_stage: SPECIFICATION` and is one of two kinds:

| Trigger | Conditions | `reason` |
|---|---|---|
| Loopback from SPEC_REVIEW | `stage.completed` of the `SPEC_REVIEW` stage with `verdict: CHANGES_REQUIRED`; `refs` contain the trigger record `review-report/<NNNN>` registered in `records.review-report.SPEC_REVIEW`; `last_stage_verdict: CHANGES_REQUIRED` | `"SPEC_REVIEW CHANGES_REQUIRED: <record_id> (<report path>); BLOCKING: <F-NNN, …>; NON_BLOCKING: <F-NNN, …>"` — the IDs of the `OPEN` findings from the front matter of the trigger report, without retelling their content |
| REQUEST_CHANGES at HUMAN_SPEC_APPROVAL (section 8.4) | `human_decision.applied` of the `HUMAN_SPEC_APPROVAL` stage with `actor.kind: human`, `details.decision: REQUEST_CHANGES` and a non-empty `details.reason`; `last_stage_verdict: PASS` (a Gate Decision does not change the verdict). BLOCKING findings are not required | `"HUMAN_SPEC_APPROVAL REQUEST_CHANGES (<actor.id>, <ts>): <details.reason verbatim>"` |

Not met, or `current_stage: CLARIFICATION` with `version > 1` — change nothing, explain and stop. **MVP limitation:** re-entry into CLARIFICATION (a new version through an interview, Staleness of `story-spec` and a repeated pass of SPECIFICATION according to the Rework Protocol, `workflow-state-model.md` §14) is not implemented. The return is not redirected to another stage.

Parameters (`specification/SKILL.md` §2):

| Parameter | Value |
|---|---|
| `previous` | The path of the current registered version of `story-spec` (`v<N-1>`). It does not change |
| `reason` | According to the trigger kind (the table above) |
| `version`, `output` | According to the general rule: `v<N>` — a new file |
| `clarification` | The current version of `clarification-record` |

Findings are the reviewer's diagnostics, and the human's reason is an indication of what to fix. Neither is a source of product decisions: they are still taken only from `clarification-record` (D-03).

#### 7.3.2. IMPACT_ANALYSIS: First Entry and Re-Entry (D-4)

Each entry into IMPACT_ANALYSIS is a new applicability assessment and a new `applicability-decision` (AP-04, AP-08; `impact-analysis/SKILL.md` §3). Re-entry is a return from SOLUTION_DESIGN: an automatic Loopback without `crosses_gates` (section 7.8).

**`version`, `previous` and `output`** — according to the general rule for a document:

| State of `artifacts.impact-analysis` | `version` | `previous` |
|---|---|---|
| none (first entry or all previous assessments were `NOT_APPLICABLE`) | `1` | not passed |
| a version `N` exists, valid or historical (section 7.1, "Conditional input") | `N+1` | the path of `v<N>`. It does not change |

`output` — `artifacts/impact-analysis/v<version>.md`. The file is created only for `PASS`. Otherwise, the `version` of the next attempt stays the same.

**`reason`** — the reasons for the current entry, verbatim, each on a separate line in History order. They are collected from the last event that brought the Story into IMPACT_ANALYSIS from another stage (the trigger from section 7.2, step 6 — APPROVE or a loopback from SOLUTION_DESIGN) up to the last event with `to_stage: IMPACT_ANALYSIS`:

| Event | `reason` line |
|---|---|
| `stage.completed` SOLUTION_DESIGN with `verdict: CHANGES_REQUIRED`, `loopback_target: IMPACT_ANALYSIS` | `"SOLUTION_DESIGN CHANGES_REQUIRED (attempt <N>, <ts>): <details.summary verbatim>"` |
| `human_decision.applied` IMPACT_ANALYSIS with `decision: RESUME` | `"LOOPBACK_CONFIRMATION RESUME (<actor.id>, <ts>): <details.reason verbatim>"` |

If there are no lines (first entry after APPROVE without `RESUME`), `reason` is not passed. SOLUTION_DESIGN findings are diagnostics, not a source of requirements (D-03).

Previous versions of the artifact, records and History events do not change. Re-entry does not change the counters: Quality Loops were accounted for during the `CHANGES_REQUIRED` of SOLUTION_DESIGN (section 7.8). The HUMAN_SPEC_APPROVAL approval is not affected, because `story-spec` and `clarification-record` do not change.

**Unregistered Working Draft.** Before starting any attempt, check whether the `output` file (for IMPACT_ANALYSIS, API_DESIGN and DB_DESIGN — also `record_output`; for IMPLEMENTATION_PLANNING — also `sec_record_output` and `rec_record_output`) exists while it is not in `state.yaml` → `artifacts` or `records`. If it exists, it is a Working Draft of a previous attempt (`artifact-rules.md` §3, item 1). Do not overwrite, delete or register it. Change nothing in the state: show the path, explain that an unregistered Working Draft was found, and stop. The human makes the decision outside the Skill.

> **Temporary MVP limitation.** The contract allows a repeated attempt to overwrite a Working Draft. This Skill deliberately does not do that until Recovery is implemented. The Working Draft contract does not change.

### 7.4. Attempt Start

One Orchestrator step:

- `state.yaml`: `execution_state: RUNNING`, `updated_at`;
- `history.jsonl`: the `stage.started` event.

```json
{"ts":"<date -u>","story_id":"US-001","actor":{"kind":"orchestrator","id":"story-orchestrator"},"event":"stage.started","stage":"SPECIFICATION","refs":["clarification-record@1"],"details":{"attempt":1,"version":1,"output":"docs/pilots/orchestrator/US-001/artifacts/story-spec/v1.md","mode":"pilot"}}
```

`refs` are the input artifacts of the attempt (`<type>@<version>`); for CLARIFICATION — `[]`. For SPEC_REVIEW, `details` contain `record_id` instead of `version`. For IMPACT_ANALYSIS, `refs: [story-spec@<N>]`, and for a re-entry with `version > 1` — also `impact-analysis@<N-1>`. `details` contain `version`, `output`, `record_id` and `record_output`, as well as `previous` and `reason`, if they exist (section 7.3.2). For SOLUTION_DESIGN, `refs` consist of `story-spec@<N>`, `impact-analysis@<M>` (only if `impact` is passed) and the valid IMPACT_ANALYSIS `applicability-decision` that determined the input. `details` contain `version`, `output` and `reason`, if it exists. For API_DESIGN, `refs: [story-spec@<N>, solution-design@<M>]`; `details` contain `version`, `output`, `record_id`, `record_output` and `reason`, if it exists. For DB_DESIGN, `refs` consist of `story-spec@<N>`, `solution-design@<M>`, `impact-analysis@<L>` and `api-design@1` (each only if passed) and the valid IMPACT_ANALYSIS and API_DESIGN `applicability-decision` records that determined the inputs; `details` as for API_DESIGN. For DESIGN_REVIEW, `refs` consist of `story-spec@<N>`, `solution-design@<M>`, the passed conditional artifacts and the three `applicability-decision` records; `details` contain `record_id` instead of `version`, `output` and `previous_reports`, if any. For PLAN_REVIEW, `refs` consist of `implementation-plan@1`, the two PLANNED records, `story-spec@<N>`, `solution-design@<M>` and the passed conditional artifacts; `details` as for DESIGN_REVIEW. For IMPLEMENTATION_PLANNING, `refs` consist of `story-spec@<N>`, `solution-design@<M>`, the passed conditional artifacts and `review-report/<NNNN>`; `details` contain `version`, `output`, `sec_record_id`, `rec_record_id`, their outputs and `reason`, if it exists. For a re-entry into SPECIFICATION (7.3.1), `refs` additionally contain the previous version (`story-spec@1`), and for a loopback from SPEC_REVIEW — also the trigger record (`review-report/0001`). `details` contain `previous` and `reason`. For the "human decision" trigger there is no record: the decision is identified by `reason` (`actor.id`, `ts`).

### 7.5. Stage Execution

Stage Skills have `disable-model-invocation: true`, so they are not invoked with the Skill tool: the executor reads `SKILL.md` and acts according to it. While the stage is running, only the Stage Skill rules apply. The Orchestrator does not suggest decisions and changes the state only where it is said below.

A stage ends with a result envelope (`workflow-state.md` §6). Without an envelope, there is no result. The human may stop the stage — the attempt is `INTERRUPTED`. A technical failure of a tool or the environment is `FAILED`. Neither is the `BLOCKED` verdict (`workflow-state-model.md` §3).

#### 7.5.1. CLARIFICATION — Inline

1. Read `.claude/skills/clarification/SKILL.md` completely.
2. Execute it **inline in the main session** with explicit arguments:
   `story=<ID> output=<output> version=<version> attempt=<attempt> mode=pilot`.
   Do not pass `source`: the Skill takes `story-source` from the registry.
3. Record waiting for the human's answers according to section 7.5.2.

#### 7.5.2. Waiting for the Human's Answers (CLARIFICATION)

Waiting for answers during CLARIFICATION is `AWAITING_HUMAN`, not `BLOCKED` (`workflow-state-model.md` §3, §4). For each interview round:

**Before showing the round** — one Orchestrator step:

- `state.yaml`: `story_status: WAITING_FOR_HUMAN`, `execution_state: AWAITING_HUMAN`, `updated_at` and
  ```yaml
  pending_human_decision:
    kind: CLARIFICATION_ANSWERS
    stage: CLARIFICATION
    options: []           # the answers are free-form, there are no decision options
    package: []           # there are no registered artifacts to review yet
    requested_at: <date -u>
  ```
- the `human_decision.requested` event: `actor` — orchestrator, `details: { attempt, round, questions: [CQ-NN, …] }`.

Then show the round (in Ukrainian) and wait for the human's answer in the same session.

**After the human's answer** — one Orchestrator step:

- `state.yaml`: `story_status: ACTIVE`, `execution_state: RUNNING`, `pending_human_decision: null`, `updated_at`;
- the `human_decision.applied` event: `actor: { kind: human, id: <decided_by> }`, `details: { attempt, round, answered: [CQ-NN, …], deferred: [CQ-NN, …] }`.

Only the question IDs get into History. The content of the answers is recorded only by `clarification-record` (`stage-io-map.yaml` → `CLARIFICATION`), the chat transcript is not stored. The Stage Skill asks for `decided_by` once. Until it is available, do not record the `human_decision.applied` event: first obtain the identifier.

If the human stops the interview while waiting, first `pending_human_decision: null` and `story_status: ACTIVE`, then the `INTERRUPTED` failure according to section 7.7.

#### 7.5.3. SPECIFICATION, SPEC_REVIEW, IMPACT_ANALYSIS, SOLUTION_DESIGN, API_DESIGN, DB_DESIGN, DESIGN_REVIEW, IMPLEMENTATION_PLANNING and PLAN_REVIEW — Subagent

1. Before the run, remember `git status --porcelain --untracked-files=all` (snapshot A) and the `git hash-object` of each registered artifact or record file of the pilot (the pilot files are not tracked by Git, so `git status` will not show their changes).
2. Start **exactly one** subagent for the current stage with the Agent tool with `subagent_type: general-purpose`. The environment may run the subagent in the background. This does not change the order: execution is sequential (steps 3–5). The prompt is only the text below for the stage, without the context of the main session (the chat, the interview, the reasoning and self-assessment of the producer stage, the Orchestrator's opinion of the artifacts are not passed).

   SPECIFICATION:

   ```text
   You are executing the SPECIFICATION stage of a pilot Story outside the registered Workflow.
   Read .claude/skills/specification/SKILL.md completely and execute it exactly.
   Arguments: story=<ID> clarification=<clarification> output=<output> version=<version> attempt=<attempt> mode=pilot [previous=<previous> reason="<reason>"]
   There is no human in this context: ask nothing.
   [If reason is from SPEC_REVIEW:] Read the report from reason yourself. Findings are the reviewer's diagnostics, not a source of product decisions: do not invent decisions that are not in clarification-record.
   [If reason is from HUMAN_SPEC_APPROVAL:] reason is the verbatim reason with which the human returned the specification at the gate. It indicates what to fix, but it is not a source of product decisions: do not invent decisions that are not in clarification-record.
   [If there is reason:] Create a new version in output; do not change the previous version.
   Write only the output file. Do not change state.yaml, history.jsonl or other files. Do not start other Skills or subagents.
   At the end of your response, return the result envelope — the YAML block `result:` according to section 6 of the Stage Skill.
   ```

   SPEC_REVIEW:

   ```text
   You are executing the SPEC_REVIEW stage of a pilot Story outside the registered Workflow as an independent reviewer.
   Read .claude/skills/spec-review/SKILL.md completely and execute it exactly, including the restrictions from its front matter.
   Arguments: story=<ID> spec=<spec> clarification=<clarification> output=<output> record_id=<record_id> attempt=<attempt> mode=pilot [previous_reports=<paths>]
   Read the inputs yourself from the files. You have no other information about these artifacts or their author.
   There is no human in this context: ask nothing.
   Write only the output file. Do not change the checked artifacts, state.yaml, history.jsonl or other files. Bash — only git log and date. Do not start other Skills or subagents.
   At the end of your response, return the result envelope — the YAML block `result:` according to section 6 of the Stage Skill.
   ```

   IMPACT_ANALYSIS:

   ```text
   You are executing the IMPACT_ANALYSIS stage of a pilot Story outside the registered Workflow.
   Read .claude/skills/impact-analysis/SKILL.md completely and execute it exactly.
   Arguments: story=<ID> spec=<spec> output=<output> version=<version> record_output=<record_output> record_id=<record_id> attempt=<attempt> mode=pilot [previous=<previous>] [reason="<reason>"]
   There is no human in this context: ask nothing.
   [If there is reason:] reason contains the reasons for this entry: SOLUTION_DESIGN findings and/or the human's verbatim reason for rejecting the proposed Loopback (RESUME). It is not a source of requirements: do not invent requirements that are not in story-spec. Assess applicability anew.
   [If there is previous:] previous is the previous version of impact-analysis. Do not change it; for PASS, create a new version in output.
   Write only the record_output file and, only for PASS, the output file. Do not change state.yaml, history.jsonl, story-spec, the code or other files. Bash — only git log, git status, git ls-files and date. Do not start other Skills or subagents.
   At the end of your response, return the result envelope — the YAML block `result:` according to section 6 of the Stage Skill.
   ```

   SOLUTION_DESIGN:

   ```text
   You are executing the SOLUTION_DESIGN stage of a pilot Story outside the registered Workflow.
   Read .claude/skills/solution-design/SKILL.md completely and execute it exactly.
   Arguments: story=<ID> spec=<spec> [impact=<impact>] output=<output> version=<version> attempt=<attempt> mode=pilot [reason="<reason>"]
   There is no human in this context: ask nothing.
   [If impact is not passed:] IMPACT_ANALYSIS is not applicable for this Story. Do not search for or use impact-analysis, even if such a file exists.
   [If there is reason:] reason is the human's verbatim reason for rejecting the proposed Loopback (RESUME). It is not a source of requirements: do not invent requirements that are not in story-spec.
   Write only the output file and only for PASS. Do not change state.yaml, history.jsonl, story-spec, impact-analysis, the code or other files. Bash — only git log, git status, git ls-files and date. Do not start other Skills or subagents.
   At the end of your response, return the result envelope — the YAML block `result:` according to section 6 of the Stage Skill.
   ```

   API_DESIGN:

   ```text
   You are executing the API_DESIGN stage of a pilot Story outside the registered Workflow.
   Read .claude/skills/api-design/SKILL.md completely and execute it exactly.
   Arguments: story=<ID> spec=<spec> design=<design> output=<output> version=1 record_output=<record_output> record_id=<record_id> attempt=<attempt> mode=pilot [reason="<reason>"]
   There is no human in this context: ask nothing.
   impact-analysis is not an input of this stage: do not search for or use it, even if such a file exists.
   [If there is reason:] reason is the human's verbatim reason for rejecting the proposed Loopback (RESUME). It is not a source of requirements: do not invent requirements that are not in story-spec. Assess applicability anew.
   Write only the record_output file and, only for PASS, the output file. Do not change state.yaml, history.jsonl, story-spec, solution-design, docs/api/, the code or other files. Bash — only git log, git status, git ls-files and date. Do not start other Skills or subagents.
   At the end of your response, return the result envelope — the YAML block `result:` according to section 6 of the Stage Skill.
   ```

   DB_DESIGN:

   ```text
   You are executing the DB_DESIGN stage of a pilot Story outside the registered Workflow.
   Read .claude/skills/db-design/SKILL.md completely and execute it exactly.
   Arguments: story=<ID> spec=<spec> design=<design> [api=<api>] [impact=<impact>] output=<output> version=1 record_output=<record_output> record_id=<record_id> attempt=<attempt> mode=pilot [reason="<reason>"]
   There is no human in this context: ask nothing.
   [If api is not passed:] API_DESIGN is not applicable for this Story. Do not search for or use api-design, even if such a file exists.
   [If impact is not passed:] IMPACT_ANALYSIS is not applicable for this Story. Do not search for or use impact-analysis, even if such a file exists.
   [If there is reason:] reason is the human's verbatim reason for rejecting the proposed Loopback (RESUME). It is not a source of requirements: do not invent requirements that are not in story-spec. Assess applicability anew.
   Write only the record_output file and, only for PASS, the output file. Do not change state.yaml, history.jsonl, story-spec, solution-design, api-design, impact-analysis, migrations/, the code or other files. Do not create SQL or migration files and do not access databases. Bash — only git log, git status, git ls-files and date. Do not start other Skills or subagents.
   At the end of your response, return the result envelope — the YAML block `result:` according to section 6 of the Stage Skill.
   ```

   DESIGN_REVIEW:

   ```text
   You are executing the DESIGN_REVIEW stage of a pilot Story outside the registered Workflow as an independent reviewer.
   Read .claude/skills/design-review/SKILL.md completely and execute it exactly, including the restrictions from its front matter.
   Arguments: story=<ID> spec=<spec> design=<design> [impact=<impact>] [api=<api>] [db=<db>] ia_decision=<ia_decision> api_decision=<api_decision> db_decision=<db_decision> output=<output> record_id=<record_id> attempt=<attempt> mode=pilot [previous_reports=<paths>]
   Read the inputs yourself from the files. You have no other information about these artifacts or their authors.
   [For each of impact, api, db that is not passed:] that stage is not applicable for this Story. Do not search for or use its artifact, even if such a file exists.
   There is no human in this context: ask nothing.
   Write only the output file. Do not change the checked artifacts, state.yaml, history.jsonl or other files. Bash — only git log, git status and date. Do not start other Skills or subagents.
   At the end of your response, return the result envelope — the YAML block `result:` according to section 6 of the Stage Skill.
   ```

   IMPLEMENTATION_PLANNING:

   ```text
   You are executing the IMPLEMENTATION_PLANNING stage of a pilot Story outside the registered Workflow.
   Read .claude/skills/implementation-planning/SKILL.md completely and execute it exactly.
   Arguments: story=<ID> spec=<spec> design=<design> [impact=<impact>] [api=<api>] [db=<db>] review=<review> testing_strategy=<testing_strategy> output=<output> version=1 sec_record_output=<sec_record_output> sec_record_id=<sec_record_id> rec_record_output=<rec_record_output> rec_record_id=<rec_record_id> attempt=<attempt> mode=pilot [reason="<reason>"]
   There is no human in this context: ask nothing.
   [For each of impact, api, db that is not passed:] that stage is not applicable for this Story. Do not search for or use its artifact, even if such a file exists.
   [If there is reason:] reason is the human's verbatim reason for rejecting the proposed Loopback (RESUME). It is not a source of requirements.
   Write only output, sec_record_output and rec_record_output, and only for PASS. Do not change state.yaml, history.jsonl, the upstream artifacts, the review, the code or other files. Do not write code, tests, SQL or migrations. Bash — only git log, git status, git ls-files and date. Do not start other Skills or subagents.
   At the end of your response, return the result envelope — the YAML block `result:` according to section 6 of the Stage Skill.
   ```

   PLAN_REVIEW:

   ```text
   You are executing the PLAN_REVIEW stage of a pilot Story outside the registered Workflow as an independent reviewer.
   Read .claude/skills/plan-review/SKILL.md completely and execute it exactly, including the restrictions from its front matter.
   Arguments: story=<ID> plan=<plan> spec=<spec> design=<design> [impact=<impact>] [api=<api>] [db=<db>] sec_decision=<sec_decision> rec_decision=<rec_decision> testing_strategy=<testing_strategy> output=<output> record_id=<record_id> attempt=<attempt> mode=pilot [previous_reports=<paths>]
   Read the inputs yourself from the files. You have no other information about these artifacts or their authors.
   [For each of impact, api, db that is not passed:] that stage is not applicable for this Story. Do not search for or use its artifact.
   There is no human in this context: ask nothing.
   Write only the output file. Do not change the checked artifacts, state.yaml, history.jsonl or other files. Bash — only git log, git status and date. Do not start other Skills or subagents.
   At the end of your response, return the result envelope — the YAML block `result:` according to section 6 of the Stage Skill.
   ```

3. Wait for the **actual completion** of the subagent and its final response (for a background run — the completion notification). A background start does not mean that the subagent has finished. While it is running:
   - do not process its result and do not guess it;
   - do not change `state.yaml`, `history.jsonl` and the pilot files;
   - do not start other subagents and stages.
4. Only after completion, take `git status --porcelain --untracked-files=all` again (snapshot B) and the `git hash-object` of the same files. The difference A→B must consist exactly of the new files that correspond to the elements of the envelope's `outputs`; if `outputs` are empty, there is no difference. For SPECIFICATION, SPEC_REVIEW, SOLUTION_DESIGN, DESIGN_REVIEW and PLAN_REVIEW, this is the `output` file. For IMPACT_ANALYSIS, API_DESIGN and DB_DESIGN, `{ record: <record_id> }` corresponds to `record_output`, and `{ type: impact-analysis | api-design | db-design, version }` — to `output`. For IMPLEMENTATION_PLANNING, `{ type: implementation-plan, version: 1 }` corresponds to `output`, and the two records to `sec_record_output` and `rec_record_output`. The hashes of registered files do not change (`artifact-rules.md` §3, item 2). Any other change — the result is rejected (`INVALID_RESULT`) with a list of files. Do not roll anything back: the human makes the decision.
5. The envelope is the last `result:` block in the subagent's response. A failure of the Agent tool — `FAILED`. A response without an envelope — `INVALID_RESULT`.

### 7.6. Result Envelope Check

The Orchestrator accepts the result only when **everything** is met (`workflow-state.md` §6):

- there is a `result` block with the fields `story_id`, `stage`, `attempt`, `verdict`, `outputs`, `loopback_target`, `blocker`, `summary`;
- `story_id`, `stage`, `attempt` match the attempt parameters;
- `verdict` is allowed for the stage according to `stage-map.yaml`: `optional: false` → not `NOT_APPLICABLE`; `CHANGES_REQUIRED` — only if the stage's `loopbacks` are not empty. CLARIFICATION: `PASS | BLOCKED`. SPECIFICATION and SPEC_REVIEW: `PASS | CHANGES_REQUIRED | BLOCKED`. IMPACT_ANALYSIS, API_DESIGN and DB_DESIGN (`optional: true`): `PASS | NOT_APPLICABLE | CHANGES_REQUIRED | BLOCKED`. SOLUTION_DESIGN, DESIGN_REVIEW, IMPLEMENTATION_PLANNING and PLAN_REVIEW (`optional: false`): `PASS | CHANGES_REQUIRED | BLOCKED`;
- `loopback_target`: for `CHANGES_REQUIRED` — a target from the stage's `loopbacks` (SPECIFICATION → `CLARIFICATION`; SPEC_REVIEW → `SPECIFICATION` or `CLARIFICATION`; IMPACT_ANALYSIS → `SPECIFICATION` or `CLARIFICATION`, both with `crosses_gates`; SOLUTION_DESIGN → `IMPACT_ANALYSIS` without `crosses_gates` or `SPECIFICATION` or `CLARIFICATION` with `crosses_gates`; API_DESIGN → `SOLUTION_DESIGN` or `IMPACT_ANALYSIS` without `crosses_gates` or `SPECIFICATION` or `CLARIFICATION` with `crosses_gates`; DB_DESIGN → `API_DESIGN`, `SOLUTION_DESIGN` or `IMPACT_ANALYSIS` without `crosses_gates` or `SPECIFICATION` or `CLARIFICATION` with `crosses_gates`; DESIGN_REVIEW and IMPLEMENTATION_PLANNING → `DB_DESIGN`, `API_DESIGN`, `SOLUTION_DESIGN` or `IMPACT_ANALYSIS` without `crosses_gates` or `SPECIFICATION` or `CLARIFICATION` with `crosses_gates`; PLAN_REVIEW → `IMPLEMENTATION_PLANNING`, `DB_DESIGN`, `API_DESIGN`, `SOLUTION_DESIGN` or `IMPACT_ANALYSIS` without `crosses_gates` or `SPECIFICATION` or `CLARIFICATION` with `crosses_gates`); otherwise `null`;
- for SPEC_REVIEW — section 7.6.1, for IMPACT_ANALYSIS — section 7.6.2, for API_DESIGN — section 7.6.3, for DB_DESIGN — section 7.6.4, for DESIGN_REVIEW — section 7.6.5, for IMPLEMENTATION_PLANNING — section 7.6.6, for PLAN_REVIEW — section 7.6.7 instead of the document rules below;
- for `PASS`: `blocker: null`; `outputs` — exactly `[{ type: <output type>, version: <version> }]`; the `output` file exists; its front matter (`artifact-rules.md` §1):
  - `artifact_type`, `story_id`, `version` match;
  - `produced_by: { stage: <current_stage>, attempt: <attempt> }`;
  - `created_at` — ISO-8601 UTC;
  - `inputs` are not empty, each source has `source`, `path`, `commit`;
  - for SPECIFICATION: `inputs` contain `{ type: clarification-record, version: <current version> }`; `story_type` is one of `story_types`; `acceptance_criteria` is a non-empty list, and each ID is in the body;
  - for SOLUTION_DESIGN (`solution-design/SKILL.md` §5):
    - `inputs` contain `{ type: story-spec, version: <current version> }`;
    - `{ type: impact-analysis, version: <version from impact> }` is in `inputs` if and only if `impact` is passed (section 7.2, step 7). A reference to any other version of `impact-analysis`, in particular a historical one after `NOT_APPLICABLE`, is a violation;
    - the `source` of each source is a key from the registry's `sources` (`architecture-foundation`, `product-foundation`, `codebase`); `path` is not `.` (D-3);
    - each `AC-NN` ID from `acceptance_criteria` of the current `story-spec` is in the body (AC traceability);
    - `version: 1`, so there is no `revision_reason`;
  - for SPECIFICATION, if `version > 1`: there is `revision_reason`, and `inputs` contain `{ type: story-spec, version: <N-1> }` (`artifact-rules.md` §1, `specification/SKILL.md` §5). `revision_reason` references each `BLOCKING` finding of the trigger report (a loopback from SPEC_REVIEW) or the human's decision `HUMAN_SPEC_APPROVAL REQUEST_CHANGES` (the trigger is the human's decision; references to findings are not required, section 7.3.1);
  - there is no status, Currency or approval;
  - the first line of the body is the pilot line of the Stage Skill;
- for `CHANGES_REQUIRED`: `outputs: []`, `blocker: null`;
- for `BLOCKED`: `outputs: []`, `blocker` has `reason`, `details`, `needed`.

An unmet condition → the result is rejected: `INVALID_RESULT` with a list of violations (section 7.7).

#### 7.6.1. Envelope and `review-report` of SPEC_REVIEW

- `outputs`: for `PASS` and `CHANGES_REQUIRED` — exactly `[{ record: <record_id> }]`; for `BLOCKED` — `[{ record: <record_id> }]` (the check was performed) or `[]` (there was no check, `spec-review/SKILL.md` §6);
- `blocker`: `null` for `PASS` and `CHANGES_REQUIRED`; for `BLOCKED` — `reason`, `details`, `needed`;
- if `outputs` are not empty — the `output` file exists, and its front matter (`artifact-rules.md` §2):
  - `record_type: review-report`, `record_id` and `story_id` match;
  - `produced_by: { stage: SPEC_REVIEW, attempt: <attempt> }`;
  - `created_at` — ISO-8601 UTC;
  - `subjects` — exactly the current versions `{ type: story-spec, version }` and `{ type: clarification-record, version }`;
  - `verdict` matches the envelope;
  - each finding has an `id` `F-NNN`, `severity` ∈ `BLOCKING | NON_BLOCKING`, `summary`, `status` ∈ `OPEN | RESOLVED | DISPOSITIONED` (`artifact-rules.md` §6);
  - `PASS` — there is no `OPEN` `BLOCKING`; `CHANGES_REQUIRED` — there is at least one `OPEN` `BLOCKING`;
  - each `OPEN` finding from `previous_reports` is present with a new state;
  - the first line of the body is the pilot line of the Stage Skill.

#### 7.6.2. Envelope, `applicability-decision` and `impact-analysis` of IMPACT_ANALYSIS

The rules — `impact-analysis/SKILL.md` §5–6, `artifact-rules.md` §1–2, `stage-map.yaml` → `APPLICABILITY_IMPACT_ANALYSIS` (AP-01, AP-03).

**`outputs` and `blocker`** (the order of elements does not matter):

| Verdict | `outputs` | `blocker` |
|---|---|---|
| `PASS` | exactly `{ type: impact-analysis, version: <version> }` and `{ record: <record_id> }` | `null` |
| `NOT_APPLICABLE` | exactly `[{ record: <record_id> }]` | `null` |
| `CHANGES_REQUIRED` | exactly `[{ record: <record_id> }]` | `null` |
| `BLOCKED` | `[{ record: <record_id> }]` (the assessment is completed) or `[]` | `reason`, `details`, `needed` |

The `output` file (`impact-analysis`) exists only for `PASS`. For other verdicts, it does not exist.

**`applicability-decision`** — if `outputs` contain a record, the `record_output` file exists, and its front matter:

- `record_type: applicability-decision`; `record_id` and `story_id` match;
- `produced_by: { stage: IMPACT_ANALYSIS, attempt: <attempt> }`; `created_at` — ISO-8601 UTC;
- `subjects` — exactly `[{ type: story-spec, version: <current version> }]`;
- `target_stage: IMPACT_ANALYSIS`, `kind: ENTRY`, `result` ∈ `APPLICABLE | NOT_APPLICABLE`;
- `triggers` — exactly the keys `IA-01`…`IA-05`, each with `status` ∈ `met | not_met` and a non-empty `ref`;
- `result: NOT_APPLICABLE` ⇔ all triggers are `not_met`; `result: APPLICABLE` ⇔ at least one is `met` (AP-01, AP-03);
- `basis` is not empty. It contains `{ type: story-spec, version: <current version> }` and the `story_type` fact, which matches `state.yaml` → `story_type`. No element of `basis` references the repository root (`path: .`);
- there are no `verdict` and `findings` fields;
- consistency with the verdict: `PASS` and `CHANGES_REQUIRED` — only with `result: APPLICABLE`; `NOT_APPLICABLE` — only with `result: NOT_APPLICABLE`; `BLOCKED` — any `result`;
- the first line of the body is the pilot line of the Stage Skill.

**`impact-analysis`** — only for `PASS`. The `output` file exists, and its front matter (`artifact-rules.md` §1):

- `artifact_type: impact-analysis`, `story_id`, `version` match; `produced_by: { stage: IMPACT_ANALYSIS, attempt: <attempt> }`; `created_at` — ISO-8601 UTC;
- `inputs` contain `{ type: story-spec, version: <current version> }`. Each source has `source`, `path` and `commit`; `path` is not `.` (D-3);
- `version: 1` — there is no `revision_reason`, and there are no references to other versions of `impact-analysis` in `inputs`;
- `version > 1` (section 7.3.2) — there is a non-empty `revision_reason`, and `inputs` contain exactly one reference to the `impact-analysis` type: `{ type: impact-analysis, version: <N-1> }` (lineage, `artifact-rules.md` §1, §3);
- there is no status, Currency or approval;
- the first line of the body is the pilot line of the Stage Skill.

#### 7.6.3. Envelope, `applicability-decision` and `api-design` of API_DESIGN

The rules — `api-design/SKILL.md` §5–6, `artifact-rules.md` §1–2, `stage-map.yaml` → `APPLICABILITY_API_DESIGN` (AP-01, AP-03). The check is structural: the semantic quality of the design is checked by DESIGN_REVIEW, not by the Orchestrator.

**`outputs` and `blocker`** — as in section 7.6.2, with `api-design` instead of `impact-analysis`:

| Verdict | `outputs` | `blocker` |
|---|---|---|
| `PASS` | exactly `{ type: api-design, version: 1 }` and `{ record: <record_id> }` | `null` |
| `NOT_APPLICABLE` | exactly `[{ record: <record_id> }]` | `null` |
| `CHANGES_REQUIRED` | exactly `[{ record: <record_id> }]` | `null` |
| `BLOCKED` | `[{ record: <record_id> }]` (the assessment is completed) or `[]` | `reason`, `details`, `needed` |

The `output` file (`api-design`) exists only for `PASS`. For other verdicts, it does not exist.

**`applicability-decision`** — if `outputs` contain a record, the `record_output` file exists, and its front matter:

- `record_type: applicability-decision`; `record_id` and `story_id` match;
- `produced_by: { stage: API_DESIGN, attempt: <attempt> }`; `created_at` — ISO-8601 UTC;
- `subjects` — exactly `{ type: story-spec, version: <current version> }` and `{ type: solution-design, version: <current version> }`;
- `target_stage: API_DESIGN`, `kind: ENTRY`, `result` ∈ `APPLICABLE | NOT_APPLICABLE`;
- `triggers` — exactly the keys `API-01`…`API-04`, each with `status` ∈ `met | not_met` and a non-empty `ref`;
- `result: NOT_APPLICABLE` ⇔ all triggers are `not_met`; `result: APPLICABLE` ⇔ at least one is `met` (AP-01, AP-03);
- `basis` — the `basis_facts` of the stage: exactly `{ type: story-spec, version: <current version> }` and `{ type: solution-design, version: <current version> }`. No element references the repository root (`path: .`);
- there are no `verdict` and `findings` fields;
- consistency with the verdict: `PASS` and `CHANGES_REQUIRED` — only with `result: APPLICABLE`; `NOT_APPLICABLE` — only with `result: NOT_APPLICABLE`; `BLOCKED` — any `result`;
- the first line of the body is the pilot line of the Stage Skill.

**`api-design`** — only for `PASS`. The `output` file exists, and its front matter (`artifact-rules.md` §1):

- `artifact_type: api-design`, `story_id` match, `version: 1`; `produced_by: { stage: API_DESIGN, attempt: <attempt> }`; `created_at` — ISO-8601 UTC;
- `inputs` contain `{ type: story-spec, version: <current version> }` and `{ type: solution-design, version: <current version> }` and no reference to `impact-analysis` or to another version of these types;
- the `source` of each source is a key from the registry's `sources` (`architecture-foundation`, `codebase`); `path` is not `.` (D-3); each source has `commit`;
- there is no `revision_reason`, status, Currency or approval;
- the body contains the sections 1–11 of `api-design/SKILL.md` §5.2 (by heading), and section 12 "Human Decision" if and only if `reason` was passed;
- the body references at least one `AC-NN`, and each referenced `AC-NN` is in `acceptance_criteria` of the current `story-spec`. Not every AC of `story-spec` has to be referenced;
- the first line of the body is the pilot line of the Stage Skill.

#### 7.6.4. Envelope, `applicability-decision` and `db-design` of DB_DESIGN

As section 7.6.3, with these differences (`db-design/SKILL.md` §5–6, `stage-map.yaml` → `APPLICABILITY_DB_DESIGN`). The check is structural.

- **`outputs` and `blocker`** — the table of section 7.6.3 with `{ type: db-design, version: 1 }` instead of `api-design`. The `output` file exists only for `PASS`.
- **`applicability-decision`:** `produced_by: { stage: DB_DESIGN, attempt: <attempt> }`; `target_stage: DB_DESIGN`, `kind: ENTRY`; `triggers` — exactly the keys `DB-01`…`DB-05`, each with `status` ∈ `met | not_met` and a non-empty `ref`; `subjects` and `basis` — exactly the current versions of `story-spec` and `solution-design`, without `api-design` and `impact-analysis`; the other rules of section 7.6.3 apply (`result` ⇔ triggers, consistency with the verdict, no `verdict` / `findings`, no `path: .`, the pilot line).
- **`db-design`** (only `PASS`): `artifact_type: db-design`, `story_id` match, `version: 1`; `produced_by: { stage: DB_DESIGN, attempt: <attempt> }`; `created_at` — ISO-8601 UTC; `inputs` contain the current `story-spec` and `solution-design`, `{ type: api-design, version: 1 }` if and only if `api` is passed, and `{ type: impact-analysis, version: <version from impact> }` if and only if `impact` is passed — a reference to any other version or to a conditional input that was not passed is a violation; each source is a registry `sources` key with `commit`, `path` is not `.`; no `revision_reason`, status, Currency or approval; the body contains sections 1–10 of `db-design/SKILL.md` §5.2 (by heading), and section 11 "Human Decision" if and only if `reason` was passed; at least one `AC-NN` is referenced, and each referenced `AC-NN` is in `acceptance_criteria` of the current `story-spec`; the first line of the body is the pilot line.

#### 7.6.5. Envelope and `review-report` of DESIGN_REVIEW

As section 7.6.1 (`design-review/SKILL.md` §5–6), with these differences:

- `produced_by: { stage: DESIGN_REVIEW, attempt: <attempt> }`;
- `subjects` — exactly `{ type: solution-design, version }` and `{ type, version }` of each passed conditional artifact (`impact-analysis`, `api-design`, `db-design`), all current, and `{ record }` of each of the three `applicability-decision` records that were passed. `story-spec` is not a subject;
- each `OPEN` finding from the previous DESIGN_REVIEW reports (`previous_reports`) is present with a new state;
- the other rules of section 7.6.1 apply: `outputs`, `blocker`, `verdict` matches the envelope, the findings structure and severity, `PASS` ⇔ no `OPEN` `BLOCKING`, `CHANGES_REQUIRED` — at least one `OPEN` `BLOCKING`, the pilot line.

#### 7.6.6. Envelope, `implementation-plan` and PLANNED records of IMPLEMENTATION_PLANNING

The rules — `implementation-planning/SKILL.md` §5–6, `artifact-rules.md` §1–2. The check is structural; PLAN_REVIEW checks the content.

- **`outputs` and `blocker`:** `PASS` — exactly `{ type: implementation-plan, version: 1 }`, `{ record: <sec_record_id> }`, `{ record: <rec_record_id> }`, `blocker: null`; `CHANGES_REQUIRED` — `[]`, `blocker: null`; `BLOCKED` — `[]`, `blocker` with `reason`, `details`, `needed`. Files exist only for `PASS`.
- **`implementation-plan`:** `artifact_type`, `story_id`, `version: 1`; `produced_by: { stage: IMPLEMENTATION_PLANNING, attempt: <attempt> }`; `created_at` — ISO-8601 UTC; `inputs` contain the current `story-spec` and `solution-design`, each passed conditional artifact (and no other version of them), `{ record: <review record_id> }` and `{ source: testing-strategy, path, commit }`; each source has `commit`, `path` is not `.`; no `revision_reason`, status, Currency or approval; the body contains sections 1–9 of `implementation-planning/SKILL.md` §5.1, and section 10 if and only if `reason` was passed; each `AC-NN` of `acceptance_criteria` of the current `story-spec` appears in the body (each AC is covered by a step and a check); the pilot line.
- **Each PLANNED record:** `record_type: applicability-decision`, `record_id` matches the argument, `story_id`; `produced_by: { stage: IMPLEMENTATION_PLANNING, attempt }`; `kind: PLANNED`; `target_stage: SECURITY_REVIEW` for `sec_record_id` and `RECONCILIATION` for `rec_record_id`; `triggers` — exactly `SEC-01`…`SEC-11` or `REC-01`…`REC-06`, each with `status` and a non-empty `ref`; `result` ⇔ the triggers (AP-01, AP-03); `subjects` and `basis` are not empty and reference only current versions; no `verdict` / `findings`; the pilot line.

#### 7.6.7. Envelope and `review-report` of PLAN_REVIEW

As section 7.6.1 (`plan-review/SKILL.md` §5–6), with these differences: `produced_by: { stage: PLAN_REVIEW, attempt: <attempt> }`; `subjects` — exactly `{ type: implementation-plan, version: 1 }` and `{ record }` of the two PLANNED records that were passed (`story-spec` and the design artifacts are not subjects); each `OPEN` finding of the previous PLAN_REVIEW reports is present with a new state. The other rules of section 7.6.1 apply.

### 7.7. Attempt Failure

`INTERRUPTED`, `FAILED` or `INVALID_RESULT`, one step:

- `execution_state` — the corresponding value, `retry.technical_attempts.<stage>` + 1, `updated_at`;
- the `stage.failed` event with `details: { attempt, execution_state, reason }`;
- the attempt file is not registered: it remains a Working Draft;
- if the counter has reached `stage-map.yaml` → `retry_policy.defaults.technical_attempts`: also `story_status: BLOCKED`, `blocker: { reason: "Retry Budget Exhausted", source: ORCHESTRATOR, details, needed: "a human's decision through Recovery" }` and the `story.blocked` event.

### 7.8. Applying an Accepted Result

The route — only from `stage-map.yaml` (`workflow-state.md` §7). One step: first `state.yaml` (section 7.10), then the events in `history.jsonl`. In all cases, `retry.technical_attempts.<stage>` is deleted (a reset after an accepted result, `workflow-state-model.md` §10) and `updated_at` is updated.

**SPEC_REVIEW record.** If `outputs` contain a record, it is registered with any verdict: `records.review-report.SPEC_REVIEW: <record_id>` (`workflow-state.md` §3). The `refs` of the `stage.completed` events contain `<record_id>`.

**IMPLEMENTATION_PLANNING records.** Only for `PASS`: `records.applicability-decision.SECURITY_REVIEW: { PLANNED: <sec_record_id> }` and `records.applicability-decision.RECONCILIATION: { PLANNED: <rec_record_id> }` (`workflow-state.md` §3). The `ENTRY` keys of these stages are not touched: a PLANNED decision is a forecast and does not replace an ENTRY decision (AP-06, AP-10).

**PLAN_REVIEW record.** If `outputs` contain a record, it is registered with any verdict: `records.review-report.PLAN_REVIEW: <record_id>`. The `refs` of the `stage.completed` events contain `<record_id>`.

**DESIGN_REVIEW record.** If `outputs` contain a record, it is registered with any verdict: `records.review-report.DESIGN_REVIEW: <record_id>` (`workflow-state.md` §3). The `refs` of the `stage.completed` events contain `<record_id>`.

**Applicability record (IMPACT_ANALYSIS, API_DESIGN, DB_DESIGN).** If `outputs` contain a record, it is registered with any verdict: `records.applicability-decision.<stage>: { ENTRY: <record_id> }`, where `<stage>` is `IMPACT_ANALYSIS`, `API_DESIGN` or `DB_DESIGN` (`workflow-state.md` §3). The previous `ENTRY` value of the same stage is replaced, because the last record by purpose becomes valid. Its file remains unchanged. The `refs` of the `stage.completed` events contain `<record_id>`.

**`PASS`** of SPEC_REVIEW → `HUMAN_SPEC_APPROVAL` (the stage's `next`, type `human_gate`). The gate is not executed; the Orchestrator only records that the Story is waiting for the human (`workflow-state-model.md` §3: for `human_gate` — `AWAITING_HUMAN`):

- `status: REVIEWED` for `story-spec` and `clarification-record` — only for the versions that are in the report's `subjects` (`workflow-state.md` §3, "Artifact Statuses");
- `last_stage_verdict: PASS`, `current_stage: HUMAN_SPEC_APPROVAL`, `story_status: WAITING_FOR_HUMAN`, `execution_state: AWAITING_HUMAN`;
- `pending_human_decision` (`workflow-state.md` §3):
  ```yaml
  pending_human_decision:
    kind: GATE
    stage: HUMAN_SPEC_APPROVAL
    options: [APPROVE, REQUEST_CHANGES, CANCEL]   # the gate's decisions; RECONFIRM is not offered in the MVP (README.md)
    package:
      - { type: clarification-record, version: <N> }
      - { type: story-spec, version: <N> }
      - { record: <record_id> }
    requested_at: <date -u>
  ```
- `approvals` do not change;
- the events `stage.completed` (`to_stage: HUMAN_SPEC_APPROVAL`) and `human_decision.requested` (`stage: HUMAN_SPEC_APPROVAL`, `refs` — the elements of `package`).

Then `next` stops at step 7.2 (`story_status` is not `ACTIVE`). `next` does not make or simulate the gate decision: the human makes it with the `decide` command (section 8).

**`PASS`** of CLARIFICATION and SPECIFICATION → the stage's `next` (CLARIFICATION → `SPECIFICATION`, SPECIFICATION → `SPEC_REVIEW`):

- `artifacts.<output type>: { version: <version>, status: DRAFT, currency: CURRENT }` — the Version Status of the new version before review (`workflow-state-model.md` §5). The previous version becomes `SUPERSEDED` without a separate record (`workflow-state.md` §3); its file remains unchanged;
- for SPECIFICATION: `story_type` — from the front matter of `story-spec` (`workflow-state.md` §3);
- `last_stage_verdict: PASS`;
- `current_stage: <next>`, `execution_state: READY`, `story_status: ACTIVE`;
- the event:

```json
{"ts":"<date -u>","story_id":"US-001","actor":{"kind":"orchestrator","id":"story-orchestrator"},"event":"stage.completed","stage":"SPECIFICATION","refs":["story-spec@1"],"details":{"attempt":1,"verdict":"PASS","outputs":["story-spec@1"],"to_stage":"SPEC_REVIEW","summary":"<summary from the envelope>","mode":"pilot"}}
```

**`PASS`** and **`NOT_APPLICABLE`** of IMPACT_ANALYSIS → `SOLUTION_DESIGN` (the stage's `next`; for `NOT_APPLICABLE` — RR-02):

- for `PASS`: `artifacts.impact-analysis: { version: <version>, status: DRAFT, currency: CURRENT }`. On re-entry, the previous version becomes `SUPERSEDED` without a separate record, and its file remains unchanged;
- for `NOT_APPLICABLE`, `artifacts` do not change. If there is an `impact-analysis` from a previous applicable assessment, it remains as a historical version without changing `status` and `currency`. It is no longer an input, because the valid decision is `NOT_APPLICABLE` (section 7.1, "Conditional input");
- the record — according to the "applicability record" rule above;
- `last_stage_verdict: <verdict>`;
- `current_stage: SOLUTION_DESIGN`, `execution_state: READY`, `story_status: ACTIVE`;
- `artifacts` of other types, `approvals` and `retry.quality_loops` do not change;
- the `stage.completed` event with `refs` — the outputs (`impact-analysis@<version>`, if any, and `<record_id>`), `details: { attempt, verdict, outputs, to_stage: SOLUTION_DESIGN, summary, mode }`.

The next `next` will execute SOLUTION_DESIGN.

**`PASS`** of SOLUTION_DESIGN → `API_DESIGN` (the stage's `next`):

- `artifacts.solution-design: { version: <version>, status: DRAFT, currency: CURRENT }`;
- `last_stage_verdict: PASS`;
- `current_stage: API_DESIGN`, `execution_state: READY`, `story_status: ACTIVE`;
- `artifacts` of other types, `records`, `approvals` and `retry.quality_loops` do not change;
- the `stage.completed` event with `refs: [solution-design@<version>]`, `details: { attempt, verdict: PASS, outputs, to_stage: API_DESIGN, summary, mode }`.

The next `next` will execute API_DESIGN.

**`PASS`** and **`NOT_APPLICABLE`** of API_DESIGN → `DB_DESIGN` (the stage's `next`; for `NOT_APPLICABLE` — RR-02):

- for `PASS`: `artifacts.api-design: { version: 1, status: DRAFT, currency: CURRENT }`;
- for `NOT_APPLICABLE`, `artifacts` do not change;
- the record — according to the "applicability record" rule above;
- `last_stage_verdict: <verdict>`;
- `current_stage: DB_DESIGN`, `execution_state: READY`, `story_status: ACTIVE`;
- `artifacts` of other types, `approvals` and `retry.quality_loops` do not change;
- the `stage.completed` event with `refs` — the outputs (`api-design@1`, if any, and `<record_id>`), `details: { attempt, verdict, outputs, to_stage: DB_DESIGN, summary, mode }`.

The next `next` will execute DB_DESIGN.

**`PASS`** and **`NOT_APPLICABLE`** of DB_DESIGN → `DESIGN_REVIEW` (the stage's `next`; for `NOT_APPLICABLE` — RR-02):

- for `PASS`: `artifacts.db-design: { version: 1, status: DRAFT, currency: CURRENT }`;
- for `NOT_APPLICABLE`, `artifacts` do not change;
- the record — according to the "applicability record" rule above;
- `last_stage_verdict: <verdict>`;
- `current_stage: DESIGN_REVIEW`, `execution_state: READY`, `story_status: ACTIVE`;
- `artifacts` of other types, `approvals` and `retry.quality_loops` do not change;
- the `stage.completed` event with `refs` — the outputs (`db-design@1`, if any, and `<record_id>`), `details: { attempt, verdict, outputs, to_stage: DESIGN_REVIEW, summary, mode }`.

The next `next` will execute DESIGN_REVIEW.

**`PASS`** of DESIGN_REVIEW → `IMPLEMENTATION_PLANNING` (the stage's `next`):

- the record — according to the "DESIGN_REVIEW record" rule above;
- `last_stage_verdict: PASS`;
- `current_stage: IMPLEMENTATION_PLANNING`, `execution_state: READY`, `story_status: ACTIVE`;
- `status: REVIEWED` for the design artifact versions that are in the report's `subjects` (`workflow-state.md` §3, "Artifact Statuses"). `PASS` is not a Human Approval (`artifact-rules.md` §5); `approvals` and `retry.quality_loops` do not change;
- the `stage.completed` event with `refs: [<record_id>]`, `details: { attempt, verdict: PASS, outputs, to_stage: IMPLEMENTATION_PLANNING, summary, mode }`.

The next `next` will execute IMPLEMENTATION_PLANNING (if the Testing Strategy exists, section 7.2, step 10).

**`PASS`** of IMPLEMENTATION_PLANNING → `PLAN_REVIEW` (the stage's `next`):

- `artifacts.implementation-plan: { version: 1, status: DRAFT, currency: CURRENT }`;
- the records — according to the "IMPLEMENTATION_PLANNING records" rule above;
- `last_stage_verdict: PASS`; `current_stage: PLAN_REVIEW`, `execution_state: READY`, `story_status: ACTIVE`;
- `artifacts` of other types, `approvals` and `retry.quality_loops` do not change;
- the `stage.completed` event with `refs: [implementation-plan@1, <sec_record_id>, <rec_record_id>]`, `details: { attempt, verdict: PASS, outputs, to_stage: PLAN_REVIEW, summary, mode }`.

The next `next` will execute PLAN_REVIEW (if the Testing Strategy exists, section 7.2, step 11).

**`PASS`** of PLAN_REVIEW → `HUMAN_TECHNICAL_APPROVAL` (the stage's `next`, type `human_gate`). The gate is not executed; the Orchestrator only records that the Story is waiting for the human, as for HUMAN_SPEC_APPROVAL:

- the record — according to the "PLAN_REVIEW record" rule above;
- `status: REVIEWED` for `implementation-plan` v1;
- `last_stage_verdict: PASS`, `current_stage: HUMAN_TECHNICAL_APPROVAL`, `story_status: WAITING_FOR_HUMAN`, `execution_state: AWAITING_HUMAN`;
- `pending_human_decision`:
  ```yaml
  pending_human_decision:
    kind: GATE
    stage: HUMAN_TECHNICAL_APPROVAL
    options: [APPROVE, REQUEST_CHANGES, CANCEL]   # RECONFIRM is not offered in the MVP
    package:                                      # every approval_scope element of the gate (workflow-state.md §4) + the review reports
      - { type: impact-analysis, version: <N> }   # or { record: <IMPACT_ANALYSIS ENTRY> } with NOT_APPLICABLE
      - { type: solution-design, version: <N> }
      - { type: api-design, version: 1 }          # or { record: <API_DESIGN ENTRY> } with NOT_APPLICABLE
      - { type: db-design, version: 1 }           # or { record: <DB_DESIGN ENTRY> } with NOT_APPLICABLE
      - { type: implementation-plan, version: 1 }
      - { record: <each applicability-decision ENTRY of IMPACT_ANALYSIS, API_DESIGN, DB_DESIGN and PLANNED of SECURITY_REVIEW, RECONCILIATION> }
      - { record: <records.review-report.DESIGN_REVIEW> }
      - { record: <records.review-report.PLAN_REVIEW> }
    requested_at: <date -u>
  ```
- `approvals` do not change; no approval is created;
- the events `stage.completed` (`to_stage: HUMAN_TECHNICAL_APPROVAL`) and `human_decision.requested` (`stage: HUMAN_TECHNICAL_APPROVAL`, `refs` — the elements of `package`).

Then `next` stops at step 7.2 (`story_status` is not `ACTIVE`). `next` does not make or simulate the gate decision: the human makes it with the `decide` command (section 8.8).

**`CHANGES_REQUIRED`** of IMPACT_ANALYSIS, SOLUTION_DESIGN, API_DESIGN, DB_DESIGN, DESIGN_REVIEW, IMPLEMENTATION_PLANNING or PLAN_REVIEW with `loopback_target` `SPECIFICATION` or `CLARIFICATION`. These targets have `crosses_gates: [HUMAN_SPEC_APPROVAL]`, so the Loopback is performed only after the human's decision (RR-04, `workflow-state-model.md` §6). The Orchestrator does not perform it, it only requests confirmation (D-1). Below, `<stage>` is `IMPACT_ANALYSIS`, `SOLUTION_DESIGN`, `API_DESIGN`, `DB_DESIGN`, `DESIGN_REVIEW`, `IMPLEMENTATION_PLANNING` or `PLAN_REVIEW`:

- for IMPACT_ANALYSIS, API_DESIGN and DB_DESIGN: the record — according to the "applicability record" rule above; for DESIGN_REVIEW and PLAN_REVIEW — according to their record rules. SOLUTION_DESIGN and IMPLEMENTATION_PLANNING create no records for `CHANGES_REQUIRED`;
- `last_stage_verdict: CHANGES_REQUIRED`; `artifacts` and `approvals` do not change;
- `retry.quality_loops` and `retry.repeated_failure` do not change. Quality Loops count returns, and there is no return yet (`workflow-state-model.md` §10). In the MVP, the accounting will be done by `ROUTE_BACK` when it is implemented; `RESUME` is not a return (decision D-1);
- `current_stage` does not change (`<stage>`); `story_status: WAITING_FOR_HUMAN`, `execution_state: AWAITING_HUMAN`;
- `pending_human_decision` (`workflow-state.md` §3):
  ```yaml
  pending_human_decision:
    kind: LOOPBACK_CONFIRMATION
    stage: <stage>
    options: [ROUTE_BACK, RESUME, CANCEL]          # loopback_confirmation_decisions; ROUTE_BACK is rejected in the MVP (section 8.7.2)
    package:
      - { type: story-spec, version: <N> }        # the approved artifact that the Loopback would change
      - { record: <record_id> }                   # IMPACT_ANALYSIS, API_DESIGN, DB_DESIGN: the applicability-decision of this attempt; DESIGN_REVIEW, PLAN_REVIEW: its review-report
    requested_at: <date -u>
  ```
- events:
  - `stage.completed` with `details: { attempt, verdict: CHANGES_REQUIRED, outputs, loopback_target, crosses_gates: [HUMAN_SPEC_APPROVAL], summary, mode }`, without `to_stage`;
  - `human_decision.requested` (`stage: <stage>`, `refs` — the elements of `package`, `details: { kind: LOOPBACK_CONFIRMATION, options, loopback_target, mode }`).

Then `next` stops at step 7.2 (`story_status` is not `ACTIVE`). The human makes the decision with the `decide` command (section 8.7).

**`CHANGES_REQUIRED`** with a Loopback without `crosses_gates` → `loopback_target`: SPECIFICATION → `CLARIFICATION`; SPEC_REVIEW → `SPECIFICATION` or `CLARIFICATION`; SOLUTION_DESIGN → `IMPACT_ANALYSIS`; API_DESIGN → `SOLUTION_DESIGN` or `IMPACT_ANALYSIS`; DB_DESIGN → `API_DESIGN`, `SOLUTION_DESIGN` or `IMPACT_ANALYSIS`; DESIGN_REVIEW and IMPLEMENTATION_PLANNING → `DB_DESIGN`, `API_DESIGN`, `SOLUTION_DESIGN` or `IMPACT_ANALYSIS`; PLAN_REVIEW → `IMPLEMENTATION_PLANNING`, `DB_DESIGN`, `API_DESIGN`, `SOLUTION_DESIGN` or `IMPACT_ANALYSIS`. These loopbacks do not have `crosses_gates`, so the human's decision is not needed (`stage-map.yaml` RR-03, RR-04):

- for API_DESIGN and DB_DESIGN: the record — according to the "applicability record" rule above;
- `last_stage_verdict: CHANGES_REQUIRED`; the Version Status of artifacts does not change;
- `retry.quality_loops.target.<loopback_target>` + 1 and `retry.quality_loops.initiator.<stage>` + 1 (`workflow-state-model.md` §10);
- for SPEC_REVIEW, DESIGN_REVIEW and PLAN_REVIEW: the record — according to its record rule above; `retry.repeated_failure.<F-NNN>` + 1 for each `OPEN` `BLOCKING` finding of the report (`workflow-state.md` §3);
- if any counter has reached the corresponding value of `retry_policy.defaults` (`quality_loops`, `repeated_failure`) — `story_status: BLOCKED`, `blocker: { reason: "Retry Budget Exhausted", source: ORCHESTRATOR, details, needed }`, `execution_state: COMPLETED`, `current_stage` does not change, the events `stage.completed` and `story.blocked`;
- otherwise `current_stage: <loopback_target>`, `execution_state: READY`, `story_status: ACTIVE`, the `stage.completed` event with `details: { attempt, verdict, loopback_target, to_stage, summary, mode }`.

Do not start the target stage. The next `next` will perform its re-entry according to section 7.3.1 if it is SPECIFICATION, or according to section 7.3.2 if it is IMPACT_ANALYSIS after a Loopback from SOLUTION_DESIGN. For CLARIFICATION, it will stop (MVP limitation, 7.3.1).

> **MVP limitation (API-D01, DB-D01).** After a Loopback from API_DESIGN (to SOLUTION_DESIGN or IMPACT_ANALYSIS), from DB_DESIGN (to API_DESIGN, SOLUTION_DESIGN or IMPACT_ANALYSIS) or from DESIGN_REVIEW or IMPLEMENTATION_PLANNING (to DB_DESIGN, API_DESIGN, SOLUTION_DESIGN or IMPACT_ANALYSIS) or from PLAN_REVIEW (to IMPLEMENTATION_PLANNING, DB_DESIGN, API_DESIGN, SOLUTION_DESIGN or IMPACT_ANALYSIS), the route and the counters are applied, but re-entry into the target is not implemented: the next `next` stops at section 7.2, step 6, and changes nothing. In the report to the human, do not present the Story as executable by `next`: say that the Story is at `<loopback_target>` without supported re-entry, and that `decide` is not available, because the Story is not waiting for a human decision. The human decides outside the Skill.

Re-entry does not change the counters: Quality Loops have already been accounted for during `CHANGES_REQUIRED`.

**`BLOCKED`** → there is no route (`stage-map.yaml` RR-05):

- for IMPACT_ANALYSIS, API_DESIGN and DB_DESIGN: the record, if it is in `outputs` — according to the "applicability record" rule above; for DESIGN_REVIEW and PLAN_REVIEW — according to their record rules;
- `execution_state: COMPLETED`, `last_stage_verdict: BLOCKED`, `story_status: BLOCKED`;
- `blocker: { reason, source: STAGE, details, needed }` from the envelope;
- `current_stage` does not change;
- the events `stage.completed` (verdict `BLOCKED`, without `to_stage`) and `story.blocked`.

### 7.9. Completion

After step 7.7 or 7.8, stop (Step Mode). Show the summary with the `status` command. Do not start the next stage.

### 7.10. Safe Update of `state.yaml`

- Do not use `sed` (or other line-based replacements in the shell) to insert or rewrite multi-line YAML mappings.
- Change `state.yaml` with the Edit tool (an exact fragment replacement) or rewrite the file completely with the Write tool. Preserve the existing indentation, key order and comments.
- After writing, read `state.yaml` again and check the structure: all keys of section 4.1 are in place, nested mappings have correct indentation, new values are where they should be.
- Only after that, append the event to `history.jsonl`. If the check did not pass — do not write the event, show the human the discrepancy and stop.

**The human's text** (`reason`, `comment`, `by`) is recorded verbatim: after unescaping, the value exactly matches the argument. Do not shorten, translate or "fix" it.

- **YAML** (`state.yaml`, front matter of records): always a double-quoted string. Escape `\` → `\\`, `"` → `\"`, a line break → `\n`, a tab → `\t`. A missing optional argument — `null` without quotes, not the string `"null"`.
- **JSON** (`history.jsonl`): a JSON string with the same escapes, other control characters — `\uXXXX`. A missing optional argument — `null` without quotes. Cyrillic and other Unicode characters stay as they are. An event is exactly one line.
- **Subagent prompt** (`reason` in 7.5.3): in double quotes with the same escapes.
- After writing, read the written value and make sure that it did not break the structure: the YAML mapping is intact, the new event is one JSON line with all the fields of `workflow-state.md` §5.

### 7.11. IMPLEMENTATION and CODE_REVIEW — Pilot Execution Model (IMP-D01)

**Decision IMP-D01 (pilot only).** `workflow-policies.md` §9 leaves branch naming to the Stage Skills and Artifact Contracts; this section defines it for pilots. The pilot remains outside the registered Workflow.

- Code changes of a pilot are made only in a **dedicated linked Git worktree** on the branch **`pilot/<story_id>`**, created by the human before the pilot (for example `git worktree add ../cp-pilot-US-001 -b pilot/US-001 main`). The whole pilot — `init` through the last stage — runs with Claude Code started inside that worktree; the pilot root `docs/pilots/orchestrator/<story_id>/` lies in that worktree.
- The main development worktree is never modified. The Orchestrator does not create, delete or switch worktrees or branches, and does not check out anything.
- The target branch is `main` (`workflow-policies.md` §9: PRs go to the main branch). It is never changed.
- **The pilot root is not code.** `state.yaml`, `history.jsonl`, `artifacts/` and `records/` under the pilot root are owned by the Orchestrator and the Stage Skills; they are excluded from every "clean worktree" and changeset check (the pilot form of the `':(exclude)docs/stories/<story_id>'` rule of `workflow-state.md` §8), and they are **never staged or committed** by IMPLEMENTATION.
- **Commits** are made only by the IMPLEMENTATION subagent, only in `pilot/<story_id>`, and only when the human's `next` command carries `allow_commit=IMPLEMENTATION` (section 7.11.2). Push, PR creation, merge, rebase, history rewriting and remote operations are not authorized by this decision. `allow_commit=IMPLEMENTATION` covers local commits only; review and verification stages have no write permission at all; only PR_PREPARATION may perform remote operations (section 7.13.1), under its own explicit permission (`AGENTS.md`).

#### 7.11.1. Execution Environment Check

Performed for IMPLEMENTATION and CODE_REVIEW after section 7.2, steps 1–5, read-only (`git rev-parse`, `git worktree list --porcelain`, `git status`, `git diff`, `git merge-base`, `git cat-file`, `git config`). The first violation — change nothing (no `stage.started`), explain to the human what exactly is wrong and stop. Below, `<root>` is the pilot root.

1. **Linked worktree.** `git rev-parse --git-dir` differs from `git rev-parse --git-common-dir`; `git rev-parse --show-toplevel` differs from the path of the first `worktree` entry of `git worktree list --porcelain` (the main development worktree); `<root>/state.yaml` lies under `--show-toplevel`.
2. **Branch.** `git rev-parse --abbrev-ref HEAD` is exactly `pilot/<story_id>` (not detached, not `main`); `refs/heads/main` exists; `git config --get branch.pilot/<story_id>.remote` is empty (no upstream: remote operations are not authorized).
3. **Clean outside the pilot root.** `git status --porcelain --untracked-files=all -- . ':(exclude)<root>'` is empty. Changes under `<root>` are allowed and are not code.
4. **Git Binding** (`workflow-state-model.md` §12, step 2):
   - IMPLEMENTATION `version: 1`: `git.feature_branch`, `git.target_branch`, `git.changeset_commit` are `null` or equal `pilot/<story_id>`, `main`, `null`; the branch has no own changes outside `<root>`: `git diff --name-only $(git merge-base main HEAD) HEAD -- . ':(exclude)<root>'` is empty;
   - IMPLEMENTATION `version > 1` and CODE_REVIEW: `git.feature_branch: pilot/<story_id>`, `git.target_branch: main`; `git.changeset_commit` equals the `changeset_commit` of the current `implementation-changeset` file and `git rev-parse HEAD`.

#### 7.11.2. Explicit Commit Permission

- The `next` argument `allow_commit=IMPLEMENTATION` is the only commit permission. It is valid only when `current_stage: IMPLEMENTATION` and only for that one attempt; with another stage it is an argument error (section 2). It is not stored in `state.yaml` and is not reused by a later `next`.
- A plain `next` never authorizes a commit: at IMPLEMENTATION without `allow_commit=IMPLEMENTATION`, change nothing (no `stage.started`), explain that the stage creates commits in `pilot/<story_id>` and needs the explicit argument, and stop.
- The permission covers only `git add <explicit paths>` and `git commit` in `pilot/<story_id>` of the current worktree. It does not cover push, merge, rebase, amend of commits not created in this attempt, reset, branch or worktree operations, changes of `main` or of another worktree.
- It is recorded in the `stage.started` event as `details.commit_permission: "allow_commit=IMPLEMENTATION"`.

#### 7.11.3. Entry, Inputs and Parameters

**Preconditions (both stages):** `approvals.HUMAN_TECHNICAL_APPROVAL` exists, `decision: APPROVE`, `approval_state: VALID`, and each element of its `scope` is the current version (or the valid record) in `state.yaml` with `status: APPROVED` for documents; `approvals.HUMAN_SPEC_APPROVAL` is `VALID` for the current `story-spec` (`stages.md` §10, preconditions).

**Entry trigger** (the last routing event, as in section 7.2, step 6):

| Stage | Trigger | `version` / extra arguments |
|---|---|---|
| IMPLEMENTATION | `human_decision.applied` HUMAN_TECHNICAL_APPROVAL `APPROVE` | `1`; `artifacts.implementation-changeset` is absent |
| IMPLEMENTATION | `stage.completed` CODE_REVIEW, SECURITY_REVIEW, RECONCILIATION or VERIFICATION `CHANGES_REQUIRED`, `loopback_target: IMPLEMENTATION`, `to_stage: IMPLEMENTATION` | current + 1; `findings` = the path of the trigger `review-report` (its `refs`) |
| IMPLEMENTATION | `human_decision.applied` IMPLEMENTATION `RESUME` (section 8.7.3) | current + 1, or `1` if none is registered; `reason` = `details.reason` verbatim; `findings` as in the previous routing trigger, if it was a review or verification stage |
| CODE_REVIEW | `stage.completed` IMPLEMENTATION `PASS`, `to_stage: CODE_REVIEW` | — |
| CODE_REVIEW | `human_decision.applied` CODE_REVIEW `RESUME` | — (the human's reason stays in History) |

Another trigger — a route this MVP does not execute: stop.

**Inputs** (`stage-io-map.yaml` → `IMPLEMENTATION`, `CODE_REVIEW`): `plan` — `implementation-plan` v1 (`status: APPROVED`); `spec` — the current `story-spec` (IMPLEMENTATION only); `design` — the current `solution-design`; `api`, `db` — only according to section 7.2, steps 8–9; `architecture-foundation` and `codebase` are read by the Skill. CODE_REVIEW also gets `changeset` — the current `implementation-changeset`. The Testing Strategy is not an input of these stages.

**Parameters:** `attempt` — section 7.3; IMPLEMENTATION: `output` = `<root>/artifacts/implementation-changeset/v<version>.md`, `feature_branch: pilot/<story_id>`, `target_branch: main`; CODE_REVIEW: `record_id` — the next `review-report/<NNNN>` (one Story-wide numbering, section 7.3), `output` = `<root>/records/review-report/<NNNN>.md`, `previous_reports` — the previous CODE_REVIEW reports from History.

**Attempt start** (section 7.4): IMPLEMENTATION `version: 1` also sets `git.feature_branch: pilot/<story_id>` and `git.target_branch: main` in the same step (the first time the pilot gets a Git Binding). `stage.started`: `refs` — the input artifacts (`implementation-plan@1`, `story-spec@<N>`, `solution-design@<M>`, the passed conditional ones; for CODE_REVIEW `implementation-changeset@<N>`); `details` — `attempt`, `version` or `record_id`, `output`, `feature_branch`, `base_head` (`git rev-parse HEAD`), `target_head` (`git rev-parse main`), `commit_permission` (IMPLEMENTATION), `findings` / `reason` / `previous_reports` if passed, `mode`.

#### 7.11.4. Execution

As section 7.5.3 (exactly one `general-purpose` subagent, a prompt without the main session's context, waiting for actual completion). Before the run remember: `H0 = git rev-parse HEAD`, `M0 = git rev-parse main`, the snapshot `git status --porcelain --untracked-files=all` and the `git hash-object` of the registered pilot files.

IMPLEMENTATION:

```text
You are executing the IMPLEMENTATION stage of a pilot Story outside the registered Workflow.
Read .claude/skills/implementation/SKILL.md completely and execute it exactly.
Arguments: story=<ID> plan=<plan> spec=<spec> design=<design> [api=<api>] [db=<db>] feature_branch=pilot/<ID> target_branch=main output=<output> version=<version> attempt=<attempt> mode=pilot commit=allowed [findings=<paths>] [reason="<reason>"]
You work in the current Git worktree on branch pilot/<ID>. Do not switch branches or worktrees and do not touch any other directory.
The pilot root <root> holds Orchestrator-owned State, History, artifacts and records. They are not code changes: never stage or commit them, and never use `git add -A`, `git add .` or `git commit -a`; stage only explicit code, test and build paths.
commit=allowed permits only `git add <paths>` and `git commit` on pilot/<ID>. No push, merge, rebase, reset, amend of earlier commits, branch or worktree operations, no change of main.
[For each of api, db that is not passed:] that stage is not applicable for this Story. Do not search for or use its artifact.
There is no human in this context: ask nothing.
Do not change state.yaml, history.jsonl or other pilot files except output. Do not start other Skills or subagents.
At the end of your response, return the result envelope — the YAML block `result:` according to section 6 of the Stage Skill.
```

CODE_REVIEW (a fresh subagent; the IMPLEMENTATION subagent and its response are never reused):

```text
You are executing the CODE_REVIEW stage of a pilot Story outside the registered Workflow as an independent reviewer.
Read .claude/skills/code-review/SKILL.md completely and execute it exactly, including the restrictions from its front matter (allowed-tools, disallowed-tools).
Arguments: story=<ID> changeset=<changeset> plan=<plan> design=<design> [api=<api>] [db=<db>] output=<output> record_id=<record_id> attempt=<attempt> mode=pilot [previous_reports=<paths>]
Evidence is the actual Git content between base_commit and changeset_commit. You have no other information about the implementation or its author.
[For each of api, db that is not passed:] that stage is not applicable for this Story. Do not search for or use its artifact.
There is no human in this context: ask nothing.
Write only the output file. Do not change code, tests, the checked artifacts, state.yaml, history.jsonl or other files. Git — read-only commands only; no checkout, switch, commit, merge, reset or push. Do not run tests. Do not start other Skills or subagents.
At the end of your response, return the result envelope — the YAML block `result:` according to section 6 of the Stage Skill.
```

**After completion** (instead of section 7.5.3, step 4), all checks; any violation → `INVALID_RESULT` with the list (section 7.7). Do not roll anything back: the human decides.

- the current branch is still `pilot/<story_id>`; `git rev-parse main` = `M0`; no upstream was configured (section 7.11.1, step 2);
- **no remote write** (the missing upstream is not proof): for each remote of `git remote`, `git ls-remote --heads <remote> refs/heads/pilot/<story_id>` returns nothing, as before the run; a found ref → `INVALID_RESULT` and a report to the human. If a remote cannot be queried, record `remote_check: unavailable` in `details` and tell the human; it is not reported as verified;
- the pilot root: exactly the new files of `outputs`; the hashes of registered files do not change;
- **CODE_REVIEW:** `HEAD` = `H0`; outside `<root>` the snapshot is unchanged;
- **IMPLEMENTATION `PASS`:** `HEAD` ≠ `H0`, and `H0` is an ancestor of `HEAD`; outside `<root>` the worktree is clean; the changeset's `base_commit` is `H0` for `version: 1` and the `base_commit` of version `N-1` for `version > 1`; its `changeset_commit` = `HEAD`; `git diff --name-status <base_commit> <changeset_commit>` equals its "Changed Files" exactly and contains no path under `docs/pilots/`, `docs/stories/`, `docs/contracts/`, `docs/workflow/`, `docs/product/`, `docs/architecture/` or `.claude/`;
- **IMPLEMENTATION `CHANGES_REQUIRED` / `BLOCKED`:** `HEAD` = `H0` (no partial commits). Uncommitted changes outside `<root>` are allowed only if the envelope describes them; the Orchestrator does not discard them and lists them to the human. The next attempt stops at section 7.11.1, step 3 until the human resolves them.

#### 7.11.5. Envelope Check

Section 7.6 (fields, `story_id`, `stage`, `attempt`), with:

- **IMPLEMENTATION:** verdicts `PASS | CHANGES_REQUIRED | BLOCKED`; `CHANGES_REQUIRED` — `loopback_target` from the stage's `loopbacks` (`IMPLEMENTATION_PLANNING`, `DB_DESIGN`, `API_DESIGN`, `SOLUTION_DESIGN`, `IMPACT_ANALYSIS`, `SPECIFICATION`, `CLARIFICATION`; all with `crosses_gates`). `PASS`: `outputs` exactly `[{ type: implementation-changeset, version }]`, `blocker: null`; the file's front matter (`artifact-rules.md` §1): `artifact_type: implementation-changeset`, `story_id`, `version`, `produced_by: { stage: IMPLEMENTATION, attempt }`, `created_at` ISO-8601 UTC, `inputs` with the current `implementation-plan`, `story-spec`, `solution-design`, the passed conditional artifacts (and no other version of them), sources with `commit`; for `version > 1` — `revision_reason` and `{ type: implementation-changeset, version: <N-1> }`; the body has sections 1–6 and 8 of `implementation/SKILL.md` §5 (7 only with `findings`, 9 only with `reason`) with `feature_branch`, `target_branch`, `base_commit`, `changeset_commit`; the pilot line. Other verdicts — `outputs: []`.
- **CODE_REVIEW:** as section 7.6.1 (`code-review/SKILL.md` §5–6): verdicts `PASS | CHANGES_REQUIRED | BLOCKED`; `CHANGES_REQUIRED` — `loopback_target` `IMPLEMENTATION` (no gate) or `IMPLEMENTATION_PLANNING`, `DB_DESIGN`, `API_DESIGN`, `SOLUTION_DESIGN`, `IMPACT_ANALYSIS`, `SPECIFICATION`, `CLARIFICATION` (with `crosses_gates`); `outputs` `[{ record: <record_id> }]`, or `[]` only for a `BLOCKED` before the check; `produced_by: { stage: CODE_REVIEW, attempt }`; `subjects` exactly `[{ type: implementation-changeset, version: <current>, changeset_commit: <git.changeset_commit> }]`; `PASS` ⇔ no `OPEN` `BLOCKING` finding; each `OPEN` finding of `previous_reports` present with a new state.

#### 7.11.6. Applying the Result

One step, section 7.8 rules (`retry.technical_attempts.<stage>` deleted, `updated_at`), and:

- **IMPLEMENTATION `PASS` → CODE_REVIEW:** `artifacts.implementation-changeset: { version, status: DRAFT, currency: CURRENT }` (the previous version becomes `SUPERSEDED`); `git.changeset_commit: <changeset_commit>`; `last_stage_verdict: PASS`, `current_stage: CODE_REVIEW`, `execution_state: READY`, `story_status: ACTIVE`; `approvals` do not change (the new changeset is not in the HUMAN_TECHNICAL_APPROVAL scope); `stage.completed` with `refs: [implementation-changeset@<version>]`, `details: { attempt, verdict: PASS, outputs, changeset_commit, to_stage: CODE_REVIEW, summary, mode }`.
- **CODE_REVIEW record:** registered with any verdict when in `outputs`: `records.review-report.CODE_REVIEW: <record_id>`; `refs` of `stage.completed` contain it.
- **CODE_REVIEW `PASS` → SECURITY_REVIEW:** `status: REVIEWED` for the reviewed `implementation-changeset` version; `last_stage_verdict: PASS`, `current_stage: SECURITY_REVIEW`, `execution_state: READY`, `story_status: ACTIVE`; `stage.completed` with `to_stage: SECURITY_REVIEW`. The next `next` executes SECURITY_REVIEW (section 7.12).
- **CODE_REVIEW `CHANGES_REQUIRED` → IMPLEMENTATION** (no gate): as the gate-free Loopback of section 7.8 — `retry.quality_loops.target.IMPLEMENTATION` + 1, `.initiator.CODE_REVIEW` + 1, `retry.repeated_failure.<F-NNN>` + 1 for each `OPEN` `BLOCKING` finding, the Retry Budget check; `current_stage: IMPLEMENTATION`, `to_stage: IMPLEMENTATION`. Re-entry is supported (section 7.11.3) and again needs `allow_commit=IMPLEMENTATION`.
- **`CHANGES_REQUIRED` with `crosses_gates`** (every IMPLEMENTATION target; CODE_REVIEW targets other than IMPLEMENTATION): `LOOPBACK_CONFIRMATION`, as in section 7.8 for the targets through a gate, with `crosses_gates` from `stage-map.yaml` and `package`: the registered artifact the Loopback would change (`implementation-plan` for IMPLEMENTATION_PLANNING, `db-design` / `api-design` / `solution-design` / `impact-analysis` for their stages — for a non-applicable stage `{ record: <its ENTRY> }`, `story-spec` for SPECIFICATION and CLARIFICATION), and for CODE_REVIEW also `{ record: <record_id> }`. Decisions — section 8.7 (`RESUME`, `CANCEL`; `ROUTE_BACK` rejected).
- **`BLOCKED`:** section 7.8 (`BLOCKED`), with the CODE_REVIEW record if any. For IMPLEMENTATION, `details` of `stage.completed` also list the uncommitted paths reported by the check of section 7.11.4.

### 7.12. SECURITY_REVIEW, RECONCILIATION and VERIFICATION — Quality Pipeline

All three run in the pilot worktree after section 7.2, steps 1–5 and 7–9 (`api`, `db`, `impact`), the checks of section 7.11.1, steps 1–3 and 4 (the CODE_REVIEW form), and section 7.12.1. Each is one fresh `general-purpose` subagent (section 7.11.4 mechanics; the prompts below). None of them has commit permission; `allow_commit` with these stages is an argument error.

#### 7.12.1. Common Entry Checks

The first violation — change nothing (no `stage.started`), explain, stop.

1. **Approvals** — as section 7.11.3 (HUMAN_SPEC_APPROVAL and HUMAN_TECHNICAL_APPROVAL `VALID` for the current versions).
2. **Review binding** (`workflow-state.md` §8, pilot form): `records.review-report.CODE_REVIEW` has `verdict: PASS`, no `OPEN` `BLOCKING`, `subjects` = the current `{ type: implementation-changeset, version, changeset_commit: git.changeset_commit }`, and it is in the `refs` of the last `stage.completed` CODE_REVIEW event with `verdict: PASS`; `git diff --quiet <changeset_commit> HEAD -- . ':(exclude)<root>'`; outside `<root>` nothing is uncommitted (7.11.1, step 3).
3. **Entry trigger** (the last routing event):

   | Stage | Trigger |
   |---|---|
   | SECURITY_REVIEW | `stage.completed` CODE_REVIEW `PASS`, `to_stage: SECURITY_REVIEW`; `human_decision.applied` SECURITY_REVIEW `RESUME` |
   | RECONCILIATION | `stage.completed` SECURITY_REVIEW `PASS` or `NOT_APPLICABLE`, `to_stage: RECONCILIATION`; `human_decision.applied` RECONCILIATION `RESUME` |
   | VERIFICATION | `stage.completed` RECONCILIATION `PASS` or `NOT_APPLICABLE`, `to_stage: VERIFICATION`; `human_decision.applied` VERIFICATION `RESUME` |

4. **Earlier quality stages** (RECONCILIATION, VERIFICATION): `records.applicability-decision.SECURITY_REVIEW.ENTRY` exists, its `subjects` are the current Changeset Content Identity, and it is in the `refs` of the last `stage.completed` SECURITY_REVIEW event; `APPLICABLE` ⇔ that event's verdict is `PASS` and `records.review-report.SECURITY_REVIEW` (in the same `refs`, `verdict: PASS`, `subjects` = the current identity) is passed as `security_review`; `NOT_APPLICABLE` ⇔ verdict `NOT_APPLICABLE`, no `security_review`. VERIFICATION: the same for `records.applicability-decision.RECONCILIATION.ENTRY` and `records.reconciliation-report` (`reconciliation_report`). An ENTRY record bound to an older changeset identity is never reused (AP-08): the stage must be passed again.
5. **PLANNED records** (SECURITY_REVIEW, RECONCILIATION): `records.applicability-decision.<STAGE>.PLANNED` exists and its file exists; it is passed as `planned_decision` and is never changed.

#### 7.12.2. Parameters

`attempt` — section 7.3. Story-wide numbering (section 7.3): `decision_id` — the next `applicability-decision/<NNNN>`; `record_id` — the next `review-report/<NNNN>` (SECURITY_REVIEW), `reconciliation-report/<NNNN>` (RECONCILIATION) or `verification-report/<NNNN>` (VERIFICATION); paths `<root>/records/<record_type>/<NNNN>.md`. Each attempt receives new numbers. `previous_reports` — the previous reports of the same stage from History. `reason` (SECURITY_REVIEW only) — `details.reason` of a `RESUME` trigger, verbatim.

| Stage | Inputs |
|---|---|
| SECURITY_REVIEW | `changeset`, `design`, `api`/`db` if applicable, `design_review` (`records.review-report.DESIGN_REVIEW`), `code_review`, `planned_decision`, `spec` |
| RECONCILIATION | `spec`, `design`, `api`/`db`/`impact` if applicable, `plan`, `changeset`, `code_review`, `security_review` if applicable, `security_decision`, `planned_decision`, `history` (the pilot `history.jsonl`, read-only) |
| VERIFICATION | `spec`, `changeset`, `plan`, `testing_strategy` (section 7.2, step 10 check: `sources.testing-strategy.path` set and the file exists), `code_review`, `security_review` / `reconciliation_report` if applicable, `security_decision`, `reconciliation_decision`, `state`, `history` (read-only), `workspace` (section 7.12.3), `target_branch: main`, `design`/`api`/`db` |

`stage.started`: `refs` — the inputs (`<type>@<version>`, `<record_id>`); `details` — `attempt`, `record_id`, `decision_id` (SR, REC), outputs, `changeset_commit`, `previous_reports`/`reason` if any, VERIFICATION also `workspace`, `workspace_head`, `target_base`, `testing_strategy_status` (the document status line, e.g. `DRAFT`), `mode`.

**Testing Strategy status.** In a pilot a `DRAFT` Testing Strategy is used as is and the status is reported to the human; it is not approved by the Orchestrator and does not close FG-01 for registered Stories.

#### 7.12.3. Verification Workspace (OD-12)

The integration result is prepared **by the human** before `next`, and passed as `next story=<ID> mode=pilot workspace=<path>` (allowed only at VERIFICATION). The Orchestrator does not create, switch, merge or delete anything. Manual prerequisite, for example:

```
git worktree add --detach ../cp-verify-US-001 main      # T = current main
cd ../cp-verify-US-001 && git merge --no-ff --no-edit <changeset_commit>   # only if main is not already contained in changeset_commit
```

The merge commit, if any, exists only on the detached `HEAD` of that worktree: no branch, no push, `main` unchanged.

Checks (read-only, `git -C <workspace>`); any violation or anything that cannot be established — stop before `stage.started` and explain what is missing:

1. `<workspace>` exists, is a linked worktree of the same repository (`git -C <workspace> rev-parse --git-common-dir` = this worktree's), and is neither this pilot worktree nor the main worktree (section 7.11.1, step 1).
2. `HEAD` is detached (`git -C <workspace> symbolic-ref -q HEAD` fails) — the integration result is on no branch.
3. Clean: `git -C <workspace> status --porcelain --untracked-files=all` is empty.
4. `T = git rev-parse main`, `W = git -C <workspace> rev-parse HEAD`, `C = git.changeset_commit`, `B` = the changeset's `base_commit`, `P` = its changed paths. Necessary: `T` and `C` are ancestors of `W`.
5. **Content identity (not only ancestry):**
   - `git diff --name-only B T -- <P>` is empty — the target did not touch approved paths; otherwise the integration content is not the approved one and needs a new IMPLEMENTATION (stop, explain);
   - `git diff --quiet C W -- <P>` — each approved path in `W` equals `C` (deletions included);
   - `git diff --name-only T W` ⊆ `P` — nothing else differs from the target base.

   Together: the tree of `W` is exactly `T` with the approved changes applied.
6. `T` is recorded as the Target Base Identity of the attempt (`details.target_base`); the Skill's `verified_target_base` must equal it.

#### 7.12.4. Prompts

SECURITY_REVIEW:

```text
You are executing the SECURITY_REVIEW stage of a pilot Story outside the registered Workflow as an independent reviewer.
Read .claude/skills/security-review/SKILL.md completely and execute it exactly, including the restrictions from its front matter (allowed-tools, disallowed-tools).
Arguments: story=<ID> changeset=<changeset> design=<design> [api=<api>] [db=<db>] design_review=<design_review> code_review=<code_review> planned_decision=<planned_decision> output=<output> record_id=<record_id> record_output=<record_output> decision_id=<decision_id> attempt=<attempt> spec=<spec> mode=pilot [previous_reports=<paths>] [reason="<reason>"]
Evidence is the actual Git content between base_commit and changeset_commit. Earlier reports are inputs to verify, not proof. You have no other information about the implementation or its author.
Assess applicability on entry yourself (SEC-01…SEC-11); the PLANNED decision may only be expanded and must not be changed. Handle every SEC-10 note of the CODE_REVIEW report.
[For each of api, db that is not passed:] that stage is not applicable for this Story. Do not search for or use its artifact.
There is no human in this context: ask nothing.
Write only record_output and, if applicable, output. Do not change code, Git history, artifacts, records, state.yaml, history.jsonl or other files. Git — read-only only; no checkout, commit, merge, reset or push. Do not start other Skills or subagents.
At the end of your response, return the result envelope — the YAML block `result:` according to section 6 of the Stage Skill.
```

RECONCILIATION — the same frame with `.claude/skills/reconciliation/SKILL.md`, its arguments (section 7.12.2, `history=<path>` read-only) and: "Assess REC-01…REC-06 yourself; the PLANNED decision may only be expanded. Check traceability only; do not run tests or the application."

VERIFICATION:

```text
You are executing the VERIFICATION stage of a pilot Story outside the registered Workflow.
Read .claude/skills/verification/SKILL.md completely and execute it exactly.
Arguments: story=<ID> spec=<spec> changeset=<changeset> plan=<plan> testing_strategy=<testing_strategy> code_review=<code_review> [security_review=<security_review>] security_decision=<security_decision> reconciliation_decision=<reconciliation_decision> [reconciliation_report=<reconciliation_report>] state=<state> history=<history> workspace=<workspace> target_branch=main output=<output> record_id=<record_id> attempt=<attempt> [design=<design>] [api=<api>] [db=<db>] mode=pilot [previous_reports=<paths>]
Run every check only inside workspace. The Testing Strategy is <status>; apply it as written and report its status. A check you could not run is not_executed or blocked, never passed. Summaries of earlier stages are not evidence.
There is no human in this context: ask nothing.
Write only output. Do not change code, tests, the repository, branches, artifacts, records, state.yaml, history.jsonl or other files. No commit, merge, rebase, reset, push. Do not start other Skills or subagents.
At the end of your response, return the result envelope — the YAML block `result:` according to section 6 of the Stage Skill.
```

**After completion** (section 7.11.4, CODE_REVIEW form): `HEAD` of this worktree and `main` unchanged; outside `<root>` unchanged; in `<root>` exactly the files of `outputs`; for VERIFICATION also `git -C <workspace> rev-parse HEAD` = `W` and the workspace status has no tracked changes (`git -C <workspace> status --porcelain --untracked-files=no` empty). Otherwise `INVALID_RESULT`.

#### 7.12.5. Envelope Check

Section 7.6 fields; `CHANGES_REQUIRED` — `loopback_target` from the stage's `loopbacks` (`IMPLEMENTATION` without a gate; `IMPLEMENTATION_PLANNING`, `DB_DESIGN`, `API_DESIGN`, `SOLUTION_DESIGN`, `IMPACT_ANALYSIS`, `SPECIFICATION`, `CLARIFICATION` with `crosses_gates`).

- **SECURITY_REVIEW / RECONCILIATION** (`optional: true`): verdicts `PASS | NOT_APPLICABLE | CHANGES_REQUIRED | BLOCKED`. `NOT_APPLICABLE` — `outputs` exactly `[{ record: <decision_id> }]`; `PASS`, `CHANGES_REQUIRED` and a review-based `BLOCKED` — `[{ record: <decision_id> }, { record: <record_id> }]`; a `BLOCKED` before the assessment — `[]`. The ENTRY record (`artifact-rules.md` §2): `record_type: applicability-decision`, `record_id`, `story_id`, `produced_by: { stage, attempt }`, `kind: ENTRY`, `target_stage: <stage>`, `subjects` exactly the current `{ type: implementation-changeset, version, changeset_commit }`, `triggers` exactly `SEC-01`…`SEC-11` / `REC-01`…`REC-06` with `status` and source, `result` ⇔ triggers (AP-01) and `APPLICABLE` if the PLANNED is `APPLICABLE` (AP-06, AP-10), `basis` containing `{ record: <planned id> }`; `NOT_APPLICABLE` ⇔ `result: NOT_APPLICABLE`; a report exists only with `result: APPLICABLE`. The report: `produced_by`, `subjects` containing the current identity (for `reconciliation-report` also the traced versions), `verdict` = envelope verdict, `PASS` ⇔ no `OPEN` `BLOCKING`, previous `OPEN` findings present. The pilot line in each file.
- **VERIFICATION** (`optional: false`): verdicts `PASS | CHANGES_REQUIRED | BLOCKED`; `outputs` `[{ record: <record_id> }]`, or `[]` only for a `BLOCKED` before the checks. The report: `record_type: verification-report`, `record_id`, `produced_by`, `subjects` and `verified_changeset` = the current identity, `verified_target_base` = `T` (section 7.12.3), `verdict` = envelope verdict, `findings`, `dod` (conditions 1–7), `follow_up`; body sections 1–10 (`verification/SKILL.md` §5), each `AC-NN` of the current `story-spec` in section 4. **`PASS` is accepted only if** every check in section 3 of the body has state `passed` with a command and evidence, every `AC-NN` has evidence, every `dod` condition is `met`, and no finding is `OPEN` `BLOCKING`. The Orchestrator does not infer `PASS` from `summary`; any `not_executed`, `blocked` or `failed` check with `PASS` → `INVALID_RESULT`.

#### 7.12.6. Applying the Result

One step (section 7.8 rules):

- **Records** (any verdict, when in `outputs`): `records.applicability-decision.<STAGE>.ENTRY: <decision_id>` (`PLANNED` unchanged); `records.review-report.SECURITY_REVIEW`, `records.reconciliation-report`, `records.verification-report: <record_id>`. VERIFICATION also sets `git.verified_target_base: <verified_target_base>` from the registered report (`workflow-state.md` §3). `refs` of `stage.completed` contain the records.
- **`PASS` / `NOT_APPLICABLE`** (RR-02): SECURITY_REVIEW → `current_stage: RECONCILIATION`; RECONCILIATION → `VERIFICATION`; VERIFICATION `PASS` → `PR_PREPARATION`; `last_stage_verdict`, `execution_state: READY`, `story_status: ACTIVE`, `to_stage`. VERIFICATION `PASS` sets `status: REVIEWED` for nothing new (the changeset is already `REVIEWED`); `approvals` do not change. The next `next` executes PR_PREPARATION (section 7.13.1).
- **`CHANGES_REQUIRED` → IMPLEMENTATION** (no gate) and **through a gate** — as section 7.11.6 for CODE_REVIEW (Quality Loops `target.IMPLEMENTATION` / `initiator.<stage>`, `repeated_failure` per `OPEN` `BLOCKING` finding, Retry Budget; `LOOPBACK_CONFIRMATION` with the package of section 7.11.6 plus the stage's record). Re-entry into IMPLEMENTATION is section 7.11.3 with `findings` = this stage's report.
- **`BLOCKED`** — section 7.8 with the records if any.

### 7.13. PR_PREPARATION, MERGE_READINESS_CHECK, INTEGRATION_CONFIRMATION — Delivery Pipeline

All three run in the pilot worktree (section 7.11.1, steps 1–2) after section 7.2, steps 1–5, as one `general-purpose` subagent each (section 7.11.4 mechanics). Every remote command of the Orchestrator and of the Skills targets the repository explicitly: `R` = the `repository` argument (PR_PREPARATION) or the `repository` of `records.pull-request` (later stages); `URL` = `https://github.com/<R>.git`; `gh … --repo <R>`. A configured `git remote` is never the source of the repository identity or of a permission. Orchestrator remote queries are read-only (`git ls-remote <URL>`, `gh pr view`, `gh repo view`, `gh api` GET).

#### 7.13.1. PR_PREPARATION

**Permission** (`AGENTS.md`, the same model as section 7.11.2): `next … allow_remote=PR_PREPARATION repository=<owner/name>` — valid only at PR_PREPARATION, only for this attempt, not stored. It covers exactly: `git push <URL> pilot/<story_id>` without force and without `-u`, and one `gh pr create --repo <R>` from `pilot/<story_id>` to `main`. It does not cover force push, push of any other ref, merge, PR edits after creation, history rewriting, repository creation or configuration. Without both arguments — change nothing (no `stage.started`), explain, stop. `allow_commit` never implies it. If `R` is the project's primary repository (the repository of the main worktree's `origin`), say so explicitly to the human in the stop/confirmation message; the explicit `repository=` argument is the selection.

**Entry checks** (the first violation — no `stage.started`, explain, stop):

1. Approvals HUMAN_SPEC_APPROVAL and HUMAN_TECHNICAL_APPROVAL `VALID` (section 7.11.3); trigger — `stage.completed` VERIFICATION `PASS`, `to_stage: PR_PREPARATION`.
2. Review binding — section 7.12.1, step 2; the SECURITY_REVIEW / RECONCILIATION ENTRY records and reports — section 7.12.1, step 4, for the current identity.
3. `records.verification-report`: `verdict: PASS`, `verified_changeset` = the current `{ version, changeset_commit }`, `verified_target_base` = `git.verified_target_base`, no `OPEN` `BLOCKING` finding.
4. Repository identity: `gh repo view <R> --json nameWithOwner` returns exactly `R`; `git ls-remote <URL> refs/heads/main` = `git.verified_target_base` (a moved target is not pushed: explain; the stage would return `CHANGES_REQUIRED` → VERIFICATION); `git ls-remote <URL> refs/heads/pilot/<story_id>` is empty or equals `git.changeset_commit`.

**Parameters:** `record_id` — the next `pull-request/<NNNN>` (Story-wide numbering, section 7.3), `output` = `<root>/records/pull-request/<NNNN>.md`; `verification`, `code_review`, `security_review` / `reconciliation_report` if applicable, `spec`, `changeset`, `repository=<R>`, `feature_branch=pilot/<story_id>`, `target_branch=main`, `remote_permission=granted`, `mode=pilot`. `stage.started` `details` also contain `repository`, `remote_permission: "allow_remote=PR_PREPARATION"`, `target_base`.

**Prompt** — the frame of section 7.12.4 with `.claude/skills/pr-preparation/SKILL.md`, these arguments, and: "Every push and gh command targets exactly <R> (push to <URL>, gh with --repo <R>). Push pilot/<ID> without force and without -u; create one PR to main. Do not merge, edit or close PRs, rewrite history, create commits or touch other refs or repositories."

**After completion** (in addition to section 7.11.4, CODE_REVIEW form — local `HEAD`, `main` and the worktree unchanged, in `<root>` only `outputs`): for `PASS` — `gh pr view <number> --repo <R> --json state,headRefOid,headRefName,baseRefName,url` gives `OPEN`, `headRefOid` = `changeset_commit`, `headRefName: pilot/<story_id>`, `baseRefName: main`; `git ls-remote <URL> refs/heads/pilot/<story_id>` = `changeset_commit`; `git ls-remote <URL> refs/heads/main` still = `git.verified_target_base`. For other verdicts, a remote change (a pushed branch, an open PR) is reported to the human and left as is.

**Envelope** — section 7.6 and `pr-preparation/SKILL.md` §6: verdicts `PASS | CHANGES_REQUIRED | BLOCKED`; `CHANGES_REQUIRED` — `IMPLEMENTATION` or `VERIFICATION` (`CHANGESET_CONTENT_UNCHANGED`), no gate; `PASS` — `outputs` exactly `[{ record: <record_id> }]`; other verdicts `[]`. The record: `record_type: pull-request`, `record_id`, `story_id`, `produced_by: { stage: PR_PREPARATION, attempt }`, `subjects` = the current changeset identity and `{ record: <verification-report> }`, `repository: R`, `number` = the observed PR, `source_branch: pilot/<story_id>`, `target_branch: main`, `changeset` = the current identity; the pilot line. A record that does not match the observed PR → `INVALID_RESULT`.

**Applying:**

- **`PASS` → HUMAN_PR_APPROVAL** (`human_gate`): `records.pull-request: <record_id>`; `last_stage_verdict: PASS`, `current_stage: HUMAN_PR_APPROVAL`, `story_status: WAITING_FOR_HUMAN`, `execution_state: AWAITING_HUMAN`; `pending_human_decision: { kind: GATE, stage: HUMAN_PR_APPROVAL, options: [APPROVE, REQUEST_CHANGES, CANCEL], package: <exactly the elements of section 8.9.1, step 4>, requested_at }`; no approval is created; events `stage.completed` (`refs: [<record_id>]`, `to_stage: HUMAN_PR_APPROVAL`) and `human_decision.requested` (`refs` = `package`). The human decides with `decide` (section 8.9).
- **`CHANGES_REQUIRED`** (no gate): the gate-free Loopback of section 7.8 (counters, Retry Budget, `current_stage: <loopback_target>`). Re-entry into VERIFICATION or IMPLEMENTATION after PR_PREPARATION is not implemented: the next `next` stops at the entry trigger check (as API-D01); a pushed branch or PR stays for the human.
- **`BLOCKED`** — section 7.8.

#### 7.13.2. MERGE_READINESS_CHECK

**Entry:** trigger — `human_decision.applied` HUMAN_PR_APPROVAL `APPROVE`, HUMAN_MERGE `REQUEST_CHANGES` with `to_stage: MERGE_READINESS_CHECK`, or MERGE_READINESS_CHECK `RESUME`; `approvals.HUMAN_PR_APPROVAL` `VALID`, its `scope` contains the current changeset identity, `records.verification-report` and `records.pull-request`.

**Parameters:** `approval` = `<root>/state.yaml` (read-only), `pull_request`, `verification`, `changeset`, `attempt`, `mode=pilot`; `merge_protection_policy` is not passed (PRQ-02: `sources.merge-protection-policy.path: null`; no mechanism is mandatory, `workflow-policies.md` §15).

**Prompt** — the frame of section 7.12.4 with `.claude/skills/merge-readiness-check/SKILL.md`: "Write no files. Read-only Git and platform queries only, always against <R>. Do not merge, push, edit PRs or change protection settings."

**After completion:** no file changed anywhere (`outputs: []`); local `HEAD`, `main` unchanged.

**Envelope:** verdicts `PASS | CHANGES_REQUIRED | BLOCKED`; `outputs: []`; `CHANGES_REQUIRED` — `VERIFICATION` (`CHANGESET_CONTENT_UNCHANGED`), `PR_PREPARATION` or `IMPLEMENTATION`, all `crosses_gates: [HUMAN_PR_APPROVAL]`. **Merge Protection Status:** `summary` must begin exactly with `merge_protection_status: <S>;` with `<S>` one of `ACTIVE`, `NOT_SUPPORTED`, `UNAVAILABLE`, `DISABLED` (regular expression `^merge_protection_status: (ACTIVE|NOT_SUPPORTED|UNAVAILABLE|DISABLED);`); missing or another value → `INVALID_RESULT`, never read as `ACTIVE`. `PASS` with `DISABLED`, or with `UNAVAILABLE` (the MVP has no way to record a human-approved integration order) → `INVALID_RESULT`. The status is a derived check result recorded with its `summary` as evidence; it is not proof that protection is enabled.

**Applying:**

- `git.merge_protection_status: <S>` for every accepted result; `stage.completed` `details` contain `merge_protection_status` and `summary`.
- **`PASS` → HUMAN_MERGE** (`human_action`): `last_stage_verdict: PASS`, `current_stage: HUMAN_MERGE`, `story_status: WAITING_FOR_HUMAN`, `execution_state: AWAITING_HUMAN`; `pending_human_decision: { kind: HUMAN_ACTION, stage: HUMAN_MERGE, options: [CONFIRM_MERGED, REQUEST_CHANGES, CANCEL], package: [{ record: <pull-request> }, { record: <verification-report> }, { type: implementation-changeset, version, changeset_commit }], requested_at }`; events `stage.completed` (`to_stage: HUMAN_MERGE`) and `human_decision.requested` (`refs` = `package`, `details: { kind: HUMAN_ACTION, options, merge_protection_status, mode }`). Tell the human that the merge is done only by the human, on the platform, and then confirmed with `decide … decision=CONFIRM_MERGED` (section 8.10). For `NOT_SUPPORTED`, state the limitation from `summary`.
- **`CHANGES_REQUIRED`** (through HUMAN_PR_APPROVAL): `LOOPBACK_CONFIRMATION` as in section 7.8, `crosses_gates: [HUMAN_PR_APPROVAL]`, `package: [{ type: implementation-changeset, version, changeset_commit }, { record: <pull-request> }]`; `decide` — section 8.7 (`RESUME` re-runs the check; `CANCEL`; `ROUTE_BACK` rejected).
- **`BLOCKED`** (Merge Protection Unavailable / Disabled, External Dependency) — section 7.8.

#### 7.13.3. HUMAN_PR_APPROVAL and HUMAN_MERGE

Unchanged: section 8.9 (the gate; `APPROVE` → MERGE_READINESS_CHECK) and section 8.10 (the human merges outside the Orchestrator; `CONFIRM_MERGED` only after the read-only remote checks of 8.10.3, recording the observed `merge_commit`, `merged_at`; `REQUEST_CHANGES` only to MERGE_READINESS_CHECK; `CANCEL`). The Orchestrator never merges.

#### 7.13.4. INTEGRATION_CONFIRMATION and DONE

**Entry:** the last routing event is `human_decision.applied` HUMAN_MERGE `CONFIRM_MERGED` with `repository` = `records.pull-request.repository`, `pr_number` = its `number`, `target_branch: main`, a non-empty `merge_commit`; `approvals.HUMAN_PR_APPROVAL` `VALID` for the current identity.

**Workspace** (`next … workspace=<path>`, mandatory; used by the Skill only for Post-Merge Validation). Prepared by the human, for example `git fetch <URL> main && git worktree add --detach ../cp-postmerge-US-001 FETCH_HEAD`. Checks (as section 7.12.3, steps 1–3): a linked worktree of this repository, neither this pilot worktree nor the main worktree, detached, clean; and `git -C <workspace> rev-parse HEAD` = `git ls-remote <URL> refs/heads/main` (the current remote target tip), with `merge_commit` an ancestor of it. Not met — stop before `stage.started`.

**Parameters:** `approval` = `<root>/state.yaml`, `history` = `<root>/history.jsonl` (both read-only), `pull_request`, `verification`, `changeset`; `record_id` — the next `integration-record/<NNNN>`, `output` = `<root>/records/integration-record/<NNNN>.md`; always `pmv_record_id` — the next `post-merge-validation-record/<NNNN>`, `pmv_output` = its path, `workspace`, `plan` (`implementation-plan`), `testing_strategy` (section 7.2, step 10 check; its status line is reported, as in 7.12.2); `backlog_dir` is not passed (PRQ-01); `mode=pilot`.

**Prompt** — the frame of section 7.12.4 with `.claude/skills/integration-confirmation/SKILL.md`: "Establish the merge strategy and the integrated range from Git yourself; do not assume that the merge commit equals changeset_commit. Git — read-only and `git fetch <URL> <ref>` into FETCH_HEAD only; no checkout, merge, revert, reset, push. Post-Merge Validation, if needed, runs only in workspace; a check you could not run is not passed. Write only output and, if performed, pmv_output."

**After completion:** local `HEAD`, all local branches (`git for-each-ref refs/heads`) and the workspace `HEAD` unchanged; in `<root>` exactly the files of `outputs`.

**Envelope** (`integration-confirmation/SKILL.md` §5): verdicts `PASS | CHANGES_REQUIRED | BLOCKED`; `CHANGES_REQUIRED` — `loopback_target: HUMAN_MERGE`, `outputs: []`. `outputs` follow its verdict table; a `BLOCKED` of steps 3–6 must contain `{ record: <record_id> }`, otherwise `INVALID_RESULT`. `integration-record`: `record_type`, `record_id`, `story_id`, `produced_by: { stage: INTEGRATION_CONFIRMATION, attempt }`, `subjects` = the current changeset identity and `{ record: <pull-request> }`, `verdict` = envelope verdict, `merge_commit` = the event's `merge_commit`, `checks` INTEGRATION_MATCH, BASE_COMPARISON, POST_MERGE_VALIDATION, FOLLOW_UP_BACKLOG; the pilot line. **`PASS` is accepted only if:** INTEGRATION_MATCH passed with a recorded strategy, range and patch identities; `target_before_merge` is recorded; POST_MERGE_VALIDATION is `not_needed` ⇔ `target_before_merge` = `git.verified_target_base`, otherwise `performed` with `{ record: <pmv_record_id> }` in `outputs`, that record's `verdict: PASS` and every check `passed` with evidence; FOLLOW_UP_BACKLOG has no approved item — the approved `verification-report` `follow_up` is `[]` (in a pilot no `backlog-entry` can be created, PRQ-01, so approved FOLLOW_UP can only end in `BLOCKED`). Otherwise → `INVALID_RESULT`. The Orchestrator does not infer `PASS` from `summary`.

**Applying:**

- **Records** (when in `outputs`, any verdict): `records.integration-record: <record_id>`, `records.post-merge-validation-record: <pmv_record_id>`; `refs` of `stage.completed` contain them.
- **`PASS` → `DONE`** (the stage's `next: DONE`, `terminal_states`): `last_stage_verdict: PASS`, `story_status: DONE`, `execution_state: COMPLETED` (the attempt is completed; no further attempts), `current_stage` stays `INTEGRATION_CONFIRMATION` (`DONE` is a terminal state, not a Stage ID, as `CANCELLED` in 8.5.3), `pending_human_decision: null`, `blocker: null`; `approvals`, `artifacts`, `git` do not change; events `stage.completed` (`details: { attempt, verdict: PASS, outputs, merge_commit, target_before_merge, post_merge_validation, to_stage: DONE, summary, mode }`) and `story.done` (`refs` — the integration record, the PMV record if any, `records.pull-request`; `details: { story_status: DONE, mode, registered_in_workflow: false }`). `DONE` is terminal; `status` shows «завершено в пілоті» and no command changes the Story afterwards.
- **`CHANGES_REQUIRED` → HUMAN_MERGE** (no gate; the merge has not been performed): `current_stage: HUMAN_MERGE` with the `pending_human_decision` of section 7.13.2 (`PASS`) again; events `stage.completed` (`to_stage: HUMAN_MERGE`) and `human_decision.requested`.
- **`BLOCKED`** (Integration Mismatch, Post-Merge Validation Failed, Missing Information) — section 7.8 with the records; the resolution is a human's Recovery, not implemented in the MVP.

## 8. Command `decide`

Applies the human's decision for the current `pending_human_decision`. Works in the main session, without a subagent. Two kinds of waiting are supported:

| `pending_human_decision.kind` | Stage | Decisions in the MVP | Sections |
|---|---|---|---|
| `GATE` | HUMAN_SPEC_APPROVAL | `APPROVE`, `REQUEST_CHANGES`, `CANCEL` | 8.2–8.5 |
| `GATE` | HUMAN_TECHNICAL_APPROVAL | `APPROVE`, `CANCEL`; `REQUEST_CHANGES` is rejected without changes | 8.2, 8.5, 8.8 |
| `GATE` | HUMAN_PR_APPROVAL | `APPROVE`, `CANCEL`; `REQUEST_CHANGES` is rejected without changes | 8.2, 8.5, 8.9 |
| `HUMAN_ACTION` | HUMAN_MERGE | `CONFIRM_MERGED`, `REQUEST_CHANGES` (only `target=MERGE_READINESS_CHECK`), `CANCEL` | 8.2, 8.5, 8.10 |
| `LOOPBACK_CONFIRMATION` | IMPACT_ANALYSIS, SOLUTION_DESIGN, API_DESIGN, DB_DESIGN, DESIGN_REVIEW, IMPLEMENTATION_PLANNING, PLAN_REVIEW | `RESUME`, `CANCEL`; `ROUTE_BACK` is rejected without changes | 8.2, 8.5, 8.7 |
| none (`story_status: BLOCKED`, `pending_human_decision: null`) — Recovery | SOLUTION_DESIGN, blocker Escalation Required | only `RESUME`; any other decision is rejected without changes | 8.2, 8.11 |

The human makes the decision, and the Orchestrator only checks and applies it (`workflow-state.md` §1).

The command is non-interactive: the decision, the human's identifier, the target, the reason and the comment come only from the arguments. The Orchestrator does not conduct a dialogue about the content of the artifacts, does not clarify anything and does not propose decisions. A gate approves the existing versions; it does not create new product decisions (D-03).

A decision in the pilot is a simulation of a Human Gate. It applies only within the pilot and is not an approval or cancellation of the Story in the official Workflow.

### 8.1. Arguments

| Argument | Decision | Rule |
|---|---|---|
| `decision` | all | For `GATE`: `APPROVE`, `REQUEST_CHANGES` or `CANCEL` (`gate_decisions` in `stage-map.yaml`); `RECONFIRM` is not offered in the MVP (`docs/contracts/README.md`, "Deferred. An Approved Simplification of the Workflow Is Needed"). For `LOOPBACK_CONFIRMATION`: `ROUTE_BACK`, `RESUME` or `CANCEL` (`loopback_confirmation_decisions`). For `HUMAN_ACTION` at HUMAN_MERGE: `CONFIRM_MERGED`, `REQUEST_CHANGES` or `CANCEL` (the stage's `decisions`). For Recovery (section 8.11): only `RESUME` (`recovery_actions`); `ROUTE_BACK`, `CANCEL` and `KEEP_BLOCKED` are not implemented for Recovery. Any other value is an error |
| `by` | all, mandatory | A non-empty identifier of the human who made the decision. In the pilot, this is a declared identifier, not verified authentication. The Orchestrator does not invent it, does not substitute it for the human and does not take the identifier of an agent or Skill |
| `target` | only `REQUEST_CHANGES`, mandatory | The target stage of the return (section 8.4.1) |
| `reason` | `REQUEST_CHANGES`, `CANCEL` and `RESUME`, mandatory | A non-empty reason. Recorded verbatim according to the rules for the human's text in section 7.10 (`workflow-policies.md` §3, §17). For `RESUME`, the next attempt of the stage receives it verbatim (section 7.3) |
| `evidence` | only Recovery `RESUME` (section 8.11), mandatory | A repository path of a Foundation document (`docs/architecture/` or `docs/product/`). Checked in section 8.11.1, step 4 |
| `comment` | only `APPROVE`, optional | Recorded verbatim according to the rules of section 7.10. It is not a source of requirements (section 8.3.1, step 3) |

If a mandatory argument is missing, there is an argument not allowed for this decision, or the `decision` is not allowed — explain how to repeat the command, and stop. `state.yaml` and `history.jsonl` do not change.

### 8.2. Common Check

For each decision. Check everything in order. The first violation — explain to the human what exactly is wrong and where (file, field), and stop. The state, History and files do not change.

1. **Consistency** of State, History and files — section 6. A discrepancy — as in `status`.
2. **Kind of waiting.** If `story_status: BLOCKED` and `pending_human_decision: null` — this is a Recovery decision: the check of section 8.11.1 instead of steps 2–4 below. Otherwise `story_status: WAITING_FOR_HUMAN`, `execution_state: AWAITING_HUMAN`, `pending_human_decision.stage` = `current_stage`. Then — according to `pending_human_decision.kind`:
   - `LOOPBACK_CONFIRMATION` — the check of section 8.7.1 instead of steps 3–4 below;
   - `GATE` at HUMAN_SPEC_APPROVAL — steps 3–4 below; `GATE` at HUMAN_TECHNICAL_APPROVAL — the check of section 8.8.1 instead of steps 3–4; `GATE` at HUMAN_PR_APPROVAL — the check of section 8.9.1 instead of steps 3–4;
   - `HUMAN_ACTION` at HUMAN_MERGE — the check of section 8.10.1 instead of steps 3–4;
   - another kind or `null` — the decision is not supported by this Skill: explain and stop.
3. **Gate state** (only `GATE`):
   - `current_stage: HUMAN_SPEC_APPROVAL`; in `stage-map.yaml` this stage has `type: human_gate`;
   - `pending_human_decision.stage: HUMAN_SPEC_APPROVAL`;
   - `decision` is in `pending_human_decision.options` and in the gate's `decisions` in `stage-map.yaml`;
   - **the package corresponds to the current versions.** `pending_human_decision.package` contains exactly:
     - `{ type, version }` for each type from the gate's `approval_scope` (`clarification-record`, `story-spec`), where `version` is the current version of that type in `artifacts`;
     - `{ record: <record_id> }`, where `<record_id>` is `records.review-report.SPEC_REVIEW`.

   The `refs` of the last `human_decision.requested` event match the elements of `package`.
4. **Previous approval** (only `GATE`). `approvals.HUMAN_SPEC_APPROVAL` is absent or `null`. If the record already exists, this is a pass of the gate after APPROVE. **MVP limitation:** such a pass (moving to `previous`, invalidation of the approval according to the Rework Protocol) is not implemented. Change nothing, explain and stop.

The decision time: `date -u +%Y-%m-%dT%H:%M:%SZ`, one value for all time fields and the `ts` of the events of this decision.

Application is one Orchestrator step (`workflow-state.md` §7): first the record file, if any (only CANCEL), then `state.yaml` according to section 7.10, then the events in `history.jsonl`. `reason`, `comment` and `by` are serialized in all files according to the rules for the human's text (section 7.10). In all decisions, the `actor` of the `human_decision.applied` event is the human from `by` (`{kind: human, id: <by>}`), `stage` is `pending_human_decision.stage` (HUMAN_SPEC_APPROVAL, HUMAN_TECHNICAL_APPROVAL, HUMAN_PR_APPROVAL, HUMAN_MERGE, IMPACT_ANALYSIS, SOLUTION_DESIGN, API_DESIGN, DB_DESIGN, DESIGN_REVIEW, IMPLEMENTATION_PLANNING, PLAN_REVIEW, IMPLEMENTATION, CODE_REVIEW, SECURITY_REVIEW, RECONCILIATION, VERIFICATION or MERGE_READINESS_CHECK), `refs` are the elements of `package`. Artifact and review report files do not change (`artifact-rules.md` §3, item 2).

### 8.3. APPROVE

#### 8.3.1. Check

1. **Scope artifacts.** For each type from `approval_scope`:
   - `status: REVIEWED`, `currency: CURRENT`, the file of the current version exists;
   - Currency is confirmed anew according to the rules of section 7.2, step 5: the upstream artifacts from `inputs` are the current versions, the sources have not changed after the recorded `commit`, a reference to the previous version is only lineage.
2. **Review report** `records.review-report.SPEC_REVIEW` (`artifact-rules.md` §2, §5):
   - front matter: `verdict: PASS`, `subjects` — exactly the current versions of both Scope types;
   - there is no finding with `severity: BLOCKING` and `status: OPEN` (`workflow-policies.md` §3, WF-V2-01);
   - the last `stage.completed` in History is of the `SPEC_REVIEW` stage with `verdict: PASS`, `to_stage: HUMAN_SPEC_APPROVAL`, and its `refs` contain this `<record_id>`.
3. **The comment does not change requirements** (D-03, `workflow-state-model.md` §5, Approved Content Identity). The approval concerns only the content of the versions from the Scope. `comment` is not interpreted and is not transferred into artifacts. If `comment` explicitly formulates a new requirement, changes an existing requirement or Acceptance Criterion, or chooses an option that is not in `story-spec` — do not apply the decision. Explain that a change of content is introduced only by a new version through `REQUEST_CHANGES` (section 8.4), and stop.

#### 8.3.2. Application

The route — only from `stage-map.yaml`: the gate's `decisions.APPROVE` → `IMPACT_ANALYSIS`.

**`state.yaml`:**

- `artifacts.clarification-record.status` and `artifacts.story-spec.status`: `APPROVED` — only for the versions from the Scope (`workflow-state.md` §3, "Artifact Statuses"). `version` and `currency` do not change;
- `approvals.HUMAN_SPEC_APPROVAL` (`workflow-state.md` §4):

  ```yaml
  approvals:
    HUMAN_SPEC_APPROVAL:
      decision: APPROVE
      decided_by: <by>
      decided_at: <date -u>
      scope:                                   # all elements of the gate's approval_scope, exact versions
        - { type: clarification-record, version: <N> }
        - { type: story-spec, version: <N> }
      package: [<record_id>]                   # the SPEC_REVIEW review-report: reviewed, not part of the Scope
      comment: "<comment>"                     # or null without quotes, if comment is not passed (section 7.10)
      approval_state: VALID
      previous: []
  ```

- `pending_human_decision: null`;
- **Quality Loops** (`workflow-state-model.md` §10, "Reset"): a full APPROVE resets the counters of the producer stages whose artifacts are part of the Scope, and of the review stages before this gate. A stage's counter is reset in both roles it can have: target and initiator of a return:
  - stages: the producer stages `CLARIFICATION` and `SPECIFICATION` (`produced_by` of the Scope types in `artifact-registry.yaml`) and the review stage `SPEC_REVIEW` (`reviews` of these stages, `next: HUMAN_SPEC_APPROVAL`);
  - `retry.quality_loops.target`: delete the keys `CLARIFICATION` and `SPECIFICATION`, if they exist;
  - `retry.quality_loops.initiator`: delete the keys `SPECIFICATION` (the initiator of the loopback to CLARIFICATION) and `SPEC_REVIEW`, if they exist. `CLARIFICATION` is never an initiator (`loopbacks: []`);
  - **do not change:** `retry.repeated_failure` (not reset within a Story), `retry.technical_attempts`, `retry.extensions`;
- `current_stage: IMPACT_ANALYSIS`, `story_status: ACTIVE`, `execution_state: READY`, `updated_at`;
- `last_stage_verdict` does not change: a Gate Decision is not a Stage Verdict (`workflow-state-model.md` §4);
- `blocker`, `records`, `git` do not change.

**`history.jsonl`** — the `human_decision.applied` event (`workflow-state.md` §5). There is no separate event for resetting counters, so the previous values are recorded in `details.retry_reset`: the history of the counters is not lost (`workflow-state-model.md` §10).

```json
{"ts":"<date -u>","story_id":"US-001","actor":{"kind":"human","id":"<by>"},"event":"human_decision.applied","stage":"HUMAN_SPEC_APPROVAL","refs":["clarification-record@1","story-spec@2","review-report/0002"],"details":{"decision":"APPROVE","to_stage":"IMPACT_ANALYSIS","scope":["clarification-record@1","story-spec@2"],"approval_state":"VALID","comment":"<comment or null>","retry_reset":{"quality_loops":{"target":{"SPECIFICATION":1},"initiator":{"SPEC_REVIEW":1}}},"mode":"pilot"}}
```

- `retry_reset` — the values of the keys that existed before the reset; a key that did not exist is not here either.

Do not start IMPACT_ANALYSIS. The next `next` will execute it (section 7).

### 8.4. REQUEST_CHANGES

A return through the Rework Protocol (`workflow-state-model.md` §14, the mechanism "REQUEST_CHANGES at a Human Gate"; `stage-map.yaml` RR-11).

#### 8.4.1. Check

1. **Target.** `target` is in the gate's `decisions.REQUEST_CHANGES` in `stage-map.yaml` (`SPECIFICATION` or `CLARIFICATION`). Another stage — the route is not declared (RR-01): explain and stop.
2. **The target is supported in the MVP.** The current Orchestrator executes only `target=SPECIFICATION`. For `target=CLARIFICATION`, reject the command before any changes: `state.yaml`, `history.jsonl` and the files do not change. Explain to the human the following:
   - the route `REQUEST_CHANGES → CLARIFICATION` is declared in the authoritative Workflow (`stage-map.yaml`), but this Orchestrator MVP does not execute it, because re-entry into CLARIFICATION is not implemented (section 7.3.1);
   - `target=SPECIFICATION` is supported now;
   - `target=CLARIFICATION` will become available when re-entry into CLARIFICATION is implemented.

   Do not redirect such a decision to SPECIFICATION. Only the human can change the target with a new command.
3. **Reason.** `reason` is non-empty. The Orchestrator does not interpret, shorten or supplement it. The reason says what to fix, but it is not a source of product decisions: they are still contained only in `clarification-record` (D-03). If the fix requires a new product decision, it is introduced only by a new version of `clarification-record`, and this is not supported in the MVP (step 2). `target=SPECIFICATION` does not replace such a decision. The human may leave the Story at the gate or choose `CANCEL`.

#### 8.4.2. Application

The route — `target` from `decisions.REQUEST_CHANGES`. In the MVP, this is only `SPECIFICATION` (section 8.4.1, step 2). There is no redirection to another stage.

**`state.yaml`:**

- `current_stage: <target>`, `story_status: ACTIVE`, `execution_state: READY`, `pending_human_decision: null`, `updated_at`;
- `last_stage_verdict` does not change (it remains the SPEC_REVIEW `PASS`): a Gate Decision is not a Stage Verdict (`workflow-state-model.md` §4);
- `retry` does not change: REQUEST_CHANGES does not count toward the agent Quality Loops and does not reset counters (`workflow-state-model.md` §10);
- `artifacts` do not change: the Version Status does not go backward, there are no new versions yet, so the Currency does not change either (Rework Protocol, steps 4–5, apply when the target stage creates a new version: section 7.8, `PASS`);
- `approvals` do not change: there is no previous approval (section 8.2, step 4), so there is nothing to invalidate (Rework Protocol, step 6);
- `records`, `blocker`, `git` do not change.

**`history.jsonl`** — the `human_decision.applied` event. It is also the event of entry into the Rework Protocol (§14, step 3): mechanism — `decision`, source — `stage`, target — `to_stage`, reason — `reason`, who — `actor`. There is no separate event.

```json
{"ts":"<date -u>","story_id":"US-001","actor":{"kind":"human","id":"<by>"},"event":"human_decision.applied","stage":"HUMAN_SPEC_APPROVAL","refs":["clarification-record@1","story-spec@2","review-report/0002"],"details":{"decision":"REQUEST_CHANGES","to_stage":"SPECIFICATION","reason":"<reason verbatim>","mode":"pilot"}}
```

Do not start the target stage. The next `next` will perform the re-entry into SPECIFICATION according to section 7.3.1 with the "human decision" trigger.

### 8.5. CANCEL

The Cancellation Protocol (`workflow-policies.md` §17, `stage-map.yaml` RR-08). `CANCELLED` is a terminal state: the Story is not reopened.

The procedure is the same for both kinds of waiting: `GATE` at HUMAN_SPEC_APPROVAL, HUMAN_TECHNICAL_APPROVAL or HUMAN_PR_APPROVAL, `HUMAN_ACTION` at HUMAN_MERGE (for the latter three, `subjects` and `refs` are taken from the gate's `package`) and `LOOPBACK_CONFIRMATION` at IMPACT_ANALYSIS, SOLUTION_DESIGN, API_DESIGN, DB_DESIGN, DESIGN_REVIEW, IMPLEMENTATION_PLANNING, PLAN_REVIEW, IMPLEMENTATION, CODE_REVIEW, SECURITY_REVIEW, RECONCILIATION, VERIFICATION or MERGE_READINESS_CHECK (`loopback_confirmation_decisions.CANCEL`). CANCEL never deletes the pilot branch or worktree and never discards uncommitted changes (`workflow-policies.md` §17, item 3). The examples below are given for the gate. For `LOOPBACK_CONFIRMATION`:
- the place of the stage in the record and the events is taken by `current_stage` (`IMPACT_ANALYSIS`, `SOLUTION_DESIGN`, `API_DESIGN`, `DB_DESIGN`, `DESIGN_REVIEW`, `IMPLEMENTATION_PLANNING`, `PLAN_REVIEW`, `IMPLEMENTATION`, `CODE_REVIEW`, `SECURITY_REVIEW`, `RECONCILIATION`, `VERIFICATION` or `MERGE_READINESS_CHECK`);
- `subjects`, `artifacts` and `refs` are taken from the current state and `package`, in particular `applicability-decision`, `impact-analysis` and `solution-design`, if they exist;
- the route is `loopback_confirmation_decisions.CANCEL` instead of the gate's `decisions.CANCEL`.

#### 8.5.1. Check

1. **Reason.** `reason` is non-empty.
2. **Record.** `record_id` — `cancellation-record/<NNNN>`, the next number of the type in the Story (`artifact-rules.md` §2): the highest number `cancellation-record/…` in the `refs` of History + 1, four digits; if there are none — `0001`. The path — `records/cancellation-record/<NNNN>.md` under the pilot root (section 3). If the file already exists — do not overwrite it, explain and stop.

#### 8.5.2. `cancellation-record`

Written by the Orchestrator by the human's decision (`artifact-registry.yaml` → `produced_by: ORCHESTRATOR`, `artifact-rules.md` §7). The header is common for records (`artifact-rules.md` §2), the fields are from the Cancellation Protocol list (`workflow-policies.md` §17). The state — before the decision is applied. The record is written in English; `reason` stays verbatim.

`produced_by` is the owner value from the registry (`ORCHESTRATOR`), not a stage. The header example in `artifact-rules.md` §2 describes records of agent stages (`{ stage, attempt }`). For an Orchestrator record the format is not defined, and a gate is not an owner and has no attempts. Therefore we do not invent a stage and an attempt: the stage at which the decision was made is recorded in `last_state.stage` and in History.

`last_state` — the values of `current_stage`, `story_status` and `execution_state` from `state.yaml` before the decision is applied. The body: "Story <ID> was cancelled by a human's decision at <current_stage> (<pending_human_decision.kind>)."

```markdown
---
record_type: cancellation-record
record_id: cancellation-record/0001
story_id: US-001
produced_by: ORCHESTRATOR                  # the owner of the type from artifact-registry.yaml
created_at: <date -u>
subjects:                                  # the current versions of the Story artifacts
  - { type: clarification-record, version: 1 }
  - { type: story-spec, version: 2 }
reason: "<reason>"                         # verbatim, YAML escaping according to section 7.10
last_state:                                # the last state of the Story before the cancellation
  stage: HUMAN_SPEC_APPROVAL
  story_status: WAITING_FOR_HUMAN
  execution_state: AWAITING_HUMAN
artifacts:                                 # the created artifacts: current versions with statuses and valid records
  - { type: clarification-record, version: 1, status: REVIEWED }
  - { type: story-spec, version: 2, status: REVIEWED }
  - { record: review-report/0002 }
feature_branch: { exists: false }          # a pilot without a feature branch (state.yaml → git)
open_pull_request: false
resources: []                              # there are no working resources (branches, PRs, uncommitted changes); the pilot files are not deleted
follow_up: []                              # there are no approved FOLLOW_UP that become a backlog-entry
---

> PILOT — outside the registered Workflow. The cancellation is simulated in the pilot, not a cancellation of the Story in the official Workflow.

Story <ID> was cancelled by a human's decision at HUMAN_SPEC_APPROVAL. The reason is in the `reason` field.
```

- `artifacts` — all keys of `artifacts` from `state.yaml` with their `version` and `status`, as well as each record from `records`. `SUPERSEDED` versions are not listed (`workflow-state.md` §3);
- `feature_branch`, `open_pull_request` — from `state.yaml` → `git`. If the `git` fields are not `null` — stop before writing the file: cancellation with Git resources is not supported in the pilot;
- `resources`: the agent deletes, closes and rolls back nothing (`workflow-policies.md` §17, item 3). No permissions for such actions are needed, because there are no resources;
- `follow_up: []`: the FOLLOW_UP Disposition is not decided at these stages, and there is no canonical Backlog yet (PRQ-01).

After writing, read the file and check the front matter. If the file could not be written — do not change `state.yaml` and History, explain and stop.

#### 8.5.3. Application

The route — the gate's `decisions.CANCEL` or `loopback_confirmation_decisions.CANCEL` → `CANCELLED` (`terminal_states`).

**`state.yaml`:**

- `story_status: CANCELLED`, `pending_human_decision: null`, `updated_at`;
- `execution_state: COMPLETED` — **an MVP decision, not a contract rule.** The contracts do not define the Execution State for `CANCELLED`, and no value fits literally:
  - `AWAITING_HUMAN` means waiting for a decision only "until the human makes a decision" (`workflow-state-model.md` §3);
  - `READY` means that an attempt can be started;
  - `COMPLETED` is described in the State Model as a completed attempt with a Stage Verdict, and a gate has no attempts and no verdict.

  `COMPLETED` is chosen as the closest value: the work at the current stage is completed and there will be no new attempts. A clarification is a matter of a future contract change;
- `current_stage` does not change (`HUMAN_SPEC_APPROVAL`, `HUMAN_TECHNICAL_APPROVAL`, `HUMAN_PR_APPROVAL`, `HUMAN_MERGE`, `IMPACT_ANALYSIS`, `SOLUTION_DESIGN`, `API_DESIGN`, `DB_DESIGN`, `DESIGN_REVIEW`, `IMPLEMENTATION_PLANNING`, `PLAN_REVIEW`, `IMPLEMENTATION`, `CODE_REVIEW`, `SECURITY_REVIEW`, `RECONCILIATION`, `VERIFICATION` or `MERGE_READINESS_CHECK`): `CANCELLED` is a terminal state, not a Stage ID (`stage-map.yaml` → `terminal_states`). The consequence of the decision is visible in `story_status` (`workflow-state.md` §4);
- `records.cancellation-record: <record_id>` — registration of the new record (`workflow-state.md` §7, step 3);
- `last_stage_verdict`, `artifacts`, `approvals`, `retry`, `blocker`, `git` do not change.

**`history.jsonl`** — two events:

```json
{"ts":"<date -u>","story_id":"US-001","actor":{"kind":"human","id":"<by>"},"event":"human_decision.applied","stage":"HUMAN_SPEC_APPROVAL","refs":["clarification-record@1","story-spec@2","review-report/0002"],"details":{"decision":"CANCEL","to_stage":"CANCELLED","reason":"<reason verbatim>","record":"cancellation-record/0001","mode":"pilot"}}
{"ts":"<date -u>","story_id":"US-001","actor":{"kind":"orchestrator","id":"story-orchestrator"},"event":"story.cancelled","stage":"HUMAN_SPEC_APPROVAL","refs":["cancellation-record/0001"],"details":{"story_status":"CANCELLED","execution_state":"COMPLETED","mode":"pilot"}}
```

For `LOOPBACK_CONFIRMATION`, in both events `stage` is `current_stage`, the `refs` of the first one are the elements of `package`, and the `details` of the first one also contain `kind: LOOPBACK_CONFIRMATION`.

Subsequent stages are not started. `next` and `decide` for `CANCELLED` stop (section 7.2, step 2; section 8.2, step 2).

### 8.6. Completion

After 8.3.2, 8.4.2, 8.5.3, 8.7.3, 8.8.3, 8.9.3 or 8.10.3, stop and show the summary with the `status` command. Do not start the next stage.

### 8.7. LOOPBACK_CONFIRMATION (IMPACT_ANALYSIS, SOLUTION_DESIGN, API_DESIGN, DB_DESIGN, DESIGN_REVIEW, IMPLEMENTATION_PLANNING, PLAN_REVIEW, IMPLEMENTATION, CODE_REVIEW, SECURITY_REVIEW, RECONCILIATION, VERIFICATION, MERGE_READINESS_CHECK)

Confirmation of a Loopback through a gate (`stage-map.yaml` RR-04, `loopback_confirmation_decisions`; `workflow-state-model.md` §6). In the MVP, it is created by a `CHANGES_REQUIRED` of IMPACT_ANALYSIS, SOLUTION_DESIGN, API_DESIGN, DB_DESIGN, DESIGN_REVIEW, IMPLEMENTATION_PLANNING or PLAN_REVIEW with the target `SPECIFICATION` or `CLARIFICATION` (section 7.8), and by a `CHANGES_REQUIRED` of IMPLEMENTATION, CODE_REVIEW, SECURITY_REVIEW, RECONCILIATION, VERIFICATION or MERGE_READINESS_CHECK with a target through HUMAN_TECHNICAL_APPROVAL (sections 7.11.6, 7.12.6). Below, `<stage>` is `current_stage`. The human makes the decision, the Orchestrator only checks and applies it. A decision in the pilot is a simulation, as at a gate.

#### 8.7.1. Check

Instead of steps 3–4 of section 8.2. The first violation — explain and stop, changing nothing.

1. **State:**
   - `current_stage` is `IMPACT_ANALYSIS`, `SOLUTION_DESIGN`, `API_DESIGN`, `DB_DESIGN`, `DESIGN_REVIEW`, `IMPLEMENTATION_PLANNING`, `PLAN_REVIEW`, `IMPLEMENTATION`, `CODE_REVIEW`, `SECURITY_REVIEW`, `RECONCILIATION`, `VERIFICATION` or `MERGE_READINESS_CHECK`; `pending_human_decision.kind: LOOPBACK_CONFIRMATION`, `pending_human_decision.stage` = `current_stage`;
   - `last_stage_verdict: CHANGES_REQUIRED`;
   - `decision` is in `pending_human_decision.options` and in `loopback_confirmation_decisions` in `stage-map.yaml`.
2. **Package.** `pending_human_decision.package` contains exactly:
   - `{ type: story-spec, version: <current version> }`;
   - only for IMPACT_ANALYSIS, API_DESIGN and DB_DESIGN — `{ record: <record_id> }`, where `<record_id>` is `records.applicability-decision.<stage>.ENTRY`; for DESIGN_REVIEW and PLAN_REVIEW — `{ record: <record_id> }`, where `<record_id>` is `records.review-report.<stage>`;
   - for MERGE_READINESS_CHECK — instead of the elements above, the package of section 7.13.2 (`CHANGES_REQUIRED`);
   - for IMPLEMENTATION, CODE_REVIEW, SECURITY_REVIEW, RECONCILIATION and VERIFICATION — instead of the elements above, the package of section 7.11.6 (the artifact the Loopback would change; for the review and verification stages also the record of the attempt).

   The `refs` of the last `human_decision.requested` event match `package`.
3. **The cause of the Loopback.** The last `stage.completed` event has `stage: <stage>`, `verdict: CHANGES_REQUIRED`, a `loopback_target` from the stage's `loopbacks` with `crosses_gates`, and no `to_stage`. For IMPACT_ANALYSIS, API_DESIGN, DB_DESIGN, DESIGN_REVIEW and PLAN_REVIEW, its `refs` contain `<record_id>`.
4. **The approval has not changed** (only for `RESUME`; `CANCEL` is available from any non-terminal state, RR-08). `approvals.HUMAN_SPEC_APPROVAL` has `approval_state: VALID`, and the `story-spec` version in its `scope` is the current one (section 7.2, step 6). For MERGE_READINESS_CHECK, `approvals.HUMAN_PR_APPROVAL` is `VALID` (section 7.13.2). For IMPLEMENTATION and the later stages, also `approvals.HUMAN_TECHNICAL_APPROVAL` is `VALID` for the current versions (section 7.11.3).

Then — according to `decision`: `ROUTE_BACK` → 8.7.2, `RESUME` → 8.7.3, `CANCEL` → section 8.5.

#### 8.7.2. ROUTE_BACK — Not Supported

Reject the command after the check of 8.7.1, **before any changes**: `state.yaml`, `history.jsonl` and the files do not change. The Story remains in `WAITING_FOR_HUMAN` with the same `pending_human_decision`. Explain to the human the following:

- the `ROUTE_BACK` route is declared in the authoritative Workflow (`loopback_confirmation_decisions`, the `loopbacks` of the `<stage>` stage), but this Orchestrator MVP does not execute it;
- it requires the full Rework Protocol (`workflow-state-model.md` §14), in particular:
  - re-entry into SPECIFICATION or CLARIFICATION after a Loopback through a gate;
  - invalidation of `HUMAN_SPEC_APPROVAL` after a new version of `story-spec`;
  - a repeated pass of the gate after APPROVE (section 8.2, step 4);
  - Quality Loops accounting;
- `RESUME` (section 8.7.3) and `CANCEL` (section 8.5) are available now.

Do not redirect the decision to another stage and do not apply another decision instead of it. Only the human can change the decision with a new command.

#### 8.7.3. RESUME

The Loopback is rejected: the Story stays at `<stage>` and starts a new attempt taking the human's decision into account (`loopback_confirmation_decisions.RESUME`).

**Check:**

- `reason` is non-empty, `target` and `comment` are not passed (section 8.1);
- **the reason does not change requirements** (D-03, `workflow-policies.md` §2). `RESUME` does not approve new requirements and does not cancel the stage's verdict: the new attempt again assesses the Story against the approved `story-spec`. If `reason` explicitly formulates a new requirement, changes an existing requirement or Acceptance Criterion, or chooses an option that is not in `story-spec` — do not apply the decision. Explain that a change of requirements is possible only through a new version of `story-spec` (`ROUTE_BACK`, which is not supported in the MVP, section 8.7.2), and stop.

**`state.yaml`:**

- `story_status: ACTIVE`, `execution_state: READY`, `pending_human_decision: null`, `updated_at`;
- `current_stage` does not change (`<stage>`);
- `last_stage_verdict` does not change (`CHANGES_REQUIRED`): a human's decision is not a Stage Verdict (`workflow-state-model.md` §4);
- `artifacts`, `records`, `approvals`, `retry`, `blocker`, `git` do not change. `story-spec` remains approved. `RESUME` does not change or reset the counters (`workflow-state-model.md` §10, decision D-1).

**`history.jsonl`** — the `human_decision.applied` event:

```json
{"ts":"<date -u>","story_id":"US-003","actor":{"kind":"human","id":"<by>"},"event":"human_decision.applied","stage":"IMPACT_ANALYSIS","refs":["story-spec@2","applicability-decision/0001"],"details":{"decision":"RESUME","kind":"LOOPBACK_CONFIRMATION","to_stage":"IMPACT_ANALYSIS","rejected_loopback_target":"SPECIFICATION","reason":"<reason verbatim>","mode":"pilot"}}
```

- `stage` and `to_stage` are `<stage>`. For SOLUTION_DESIGN and IMPLEMENTATION_PLANNING, `refs` are `package`, that is, only `story-spec@<N>`. For API_DESIGN and DB_DESIGN, `refs` are `story-spec@<N>` and `applicability-decision/<NNNN>`, as for IMPACT_ANALYSIS; for DESIGN_REVIEW and PLAN_REVIEW — `story-spec@<N>` and `review-report/<NNNN>`;
- `rejected_loopback_target` is the `loopback_target` of the event from step 3 of section 8.7.1.

Do not start the stage. The next `next` will execute a new attempt and pass `reason`: for IMPACT_ANALYSIS — according to section 7.3.2, for SOLUTION_DESIGN, API_DESIGN, DB_DESIGN and IMPLEMENTATION_PLANNING — verbatim according to section 7.3; DESIGN_REVIEW, PLAN_REVIEW, CODE_REVIEW, RECONCILIATION and VERIFICATION have no `reason` argument (SECURITY_REVIEW receives it, section 7.12.2): its new attempt receives the previous report through `previous_reports`, and the human's reason stays in History; IMPLEMENTATION receives `reason` according to section 7.11.3 and again needs `allow_commit=IMPLEMENTATION`. `version` does not change, because the artifact of this attempt was not registered. For IMPACT_ANALYSIS, API_DESIGN and DB_DESIGN, `record_id` is the next number, and the previous `applicability-decision` will be replaced by the new record after an accepted result (section 7.8).

### 8.8. HUMAN_TECHNICAL_APPROVAL

The human approves the technical design and the Implementation Plan (`stages.md` §9, Q5). The procedure is that of HUMAN_SPEC_APPROVAL (sections 8.2–8.5) with the gate's own `approval_scope` and `decisions` from `stage-map.yaml`.

#### 8.8.1. Check

Instead of steps 3–4 of section 8.2. The first violation — explain and stop, changing nothing.

1. **Gate state.** `current_stage: HUMAN_TECHNICAL_APPROVAL` (`type: human_gate`); `pending_human_decision.kind: GATE`, `pending_human_decision.stage: HUMAN_TECHNICAL_APPROVAL`; `decision` is in `pending_human_decision.options` and in the gate's `decisions`; `blocker: null`.
2. **Preconditions** (`stages.md` §9): the last `stage.completed` event is PLAN_REVIEW with `verdict: PASS`, `to_stage: HUMAN_TECHNICAL_APPROVAL`, and its `refs` contain `records.review-report.PLAN_REVIEW`. The PLAN_REVIEW report has `verdict: PASS`, no `OPEN` `BLOCKING` finding, and its `subjects` are the current `implementation-plan` and PLANNED records. The DESIGN_REVIEW report (`records.review-report.DESIGN_REVIEW`) has `verdict: PASS` and its `subjects` are the current design artifact versions and the valid ENTRY records.
3. **The package corresponds to the current state.** `pending_human_decision.package` contains exactly the elements of section 7.8 (PLAN_REVIEW `PASS`): for each `approval_scope` element of the gate — `{ type, version }` of the current version in `artifacts` (`solution-design`, `implementation-plan`, and `impact-analysis`, `api-design`, `db-design` of applicable stages) or, for a non-applicable stage, `{ record: <its ENTRY decision> }` with `NOT_APPLICABLE` (`workflow-state.md` §4); `{ record }` of each valid `applicability-decision` ENTRY of IMPACT_ANALYSIS, API_DESIGN, DB_DESIGN and PLANNED of SECURITY_REVIEW, RECONCILIATION; the two review reports. The `refs` of the last `human_decision.requested` event match `package`.
4. **Currency.** Each document of the Scope: `status: REVIEWED`, `currency: CURRENT`, the file exists; Currency is confirmed anew according to section 7.2, step 5.
5. **Previous approval.** `approvals.HUMAN_TECHNICAL_APPROVAL` is absent or `null`. A repeated pass is not implemented (as in section 8.2, step 4).
6. **The approval of HUMAN_SPEC_APPROVAL is still valid:** `approval_state: VALID`, its `scope` is the current `story-spec` and `clarification-record`.

#### 8.8.2. REQUEST_CHANGES — Not Supported

The gate's `decisions.REQUEST_CHANGES` targets (IMPLEMENTATION_PLANNING, DB_DESIGN, API_DESIGN, SOLUTION_DESIGN, IMPACT_ANALYSIS, SPECIFICATION, CLARIFICATION) are declared, but each requires re-entry with a new version of an already registered artifact, which this MVP does not implement. Reject the command after the check of 8.8.1, before any changes, and explain this; `APPROVE` and `CANCEL` are available. Do not redirect the decision.

#### 8.8.3. APPROVE

**Check:** as section 8.3.1, step 3 — `comment` does not change requirements or the design; otherwise do not apply the decision.

**Approval boundaries.** The approval is bound to the exact Scope elements (`artifact-rules.md` §5). The review reports are reviewed but are not part of the Scope: they go to `package`. The PLANNED decisions are approved as forecasts and do not replace future ENTRY decisions (AP-06, AP-10). The approval does not by itself approve a significant dependency or a risk that requires a separate human decision (`workflow-policies.md` §2, §7): such decisions are not recorded here.

**`state.yaml`:**

- `status: APPROVED` for each document of the Scope (`workflow-state.md` §3); `version` and `currency` do not change;
- `approvals.HUMAN_TECHNICAL_APPROVAL` (`workflow-state.md` §4):
  ```yaml
  approvals:
    HUMAN_TECHNICAL_APPROVAL:
      decision: APPROVE
      decided_by: <by>
      decided_at: <date -u>
      scope:                                   # the approval_scope elements of the gate, exact identities (section 8.8.1, step 3)
        - { type: solution-design, version: <N> }
        - { type: implementation-plan, version: 1 }
        - { type: impact-analysis, version: <N> }   # or { record: <ENTRY> } with NOT_APPLICABLE; the same for api-design, db-design
        - { record: <each applicability-decision ENTRY and PLANNED of the package> }
      package: [<DESIGN_REVIEW review-report>, <PLAN_REVIEW review-report>]
      comment: "<comment>"                     # or null
      approval_state: VALID
      previous: []
  ```
- `pending_human_decision: null`;
- **Quality Loops** reset, as in section 8.3.2: the producer stages of the Scope types (IMPACT_ANALYSIS, SOLUTION_DESIGN, API_DESIGN, DB_DESIGN, IMPLEMENTATION_PLANNING) and the review stages before this gate (DESIGN_REVIEW, PLAN_REVIEW) — their keys in `retry.quality_loops.target` and `.initiator` are deleted; `repeated_failure`, `technical_attempts`, `extensions` do not change;
- `current_stage: IMPLEMENTATION` (the gate's `decisions.APPROVE`), `story_status: ACTIVE`, `execution_state: READY`, `updated_at`; `last_stage_verdict`, `blocker`, `records`, `git` do not change.

**`history.jsonl`** — the `human_decision.applied` event, as in section 8.3.2: `stage: HUMAN_TECHNICAL_APPROVAL`, `refs` — the elements of `package`, `details: { decision: APPROVE, to_stage: IMPLEMENTATION, scope, approval_state: VALID, comment, retry_reset, mode }`.

Do not start IMPLEMENTATION. The next `next` with `allow_commit=IMPLEMENTATION`, started in the pilot worktree, executes it (section 7.11).

**CANCEL** — section 8.5 with `current_stage: HUMAN_TECHNICAL_APPROVAL` and the gate's `decisions.CANCEL`.

### 8.9. HUMAN_PR_APPROVAL

A human approves a specific version of the Story's changes before integration (`stages.md` §12, Q5). The procedure is that of section 8.8 with this gate's `approval_scope` and `decisions` (`stage-map.yaml`). The gate is reached only after PR_PREPARATION `PASS` (section 1, IMP-D01); the rules below apply to valid state produced by the integrated stages.

#### 8.9.1. Check

Instead of steps 3–4 of section 8.2. The first violation — explain and stop, changing nothing.

1. **Gate state.** `current_stage: HUMAN_PR_APPROVAL` (`type: human_gate`); `pending_human_decision.kind: GATE`, `stage: HUMAN_PR_APPROVAL`; `decision` is in `options` and in the gate's `decisions`; `blocker: null`.
2. **Preconditions.** The last `stage.completed` event is PR_PREPARATION with `verdict: PASS`, `to_stage: HUMAN_PR_APPROVAL`, and its `refs` contain `records.pull-request`. The `pull-request` record has this `story_id`, and its `changeset` equals the current `artifacts.implementation-changeset` (`version`) and `git.changeset_commit`. The `verification-report` (`records.verification-report`) has `verdict: PASS`, `verified_changeset` equal to the same identity and `verified_target_base` equal to `git.verified_target_base`; its DoD conditions 1–7 are `met`; no finding of any report is `OPEN` `BLOCKING`; each `NON_BLOCKING` and FOLLOW_UP finding has a Disposition (DoD 6).
3. **Live Git check** (read-only): the remote PR is open and its head commit equals `changeset_commit` (`gh pr view <number> --repo <pull-request.repository> --json state,headRefOid,baseRefName,headRefName`, with `baseRefName` / `headRefName` equal to the record's `target_branch` / `source_branch`); the remote tip of `target_branch` (`git ls-remote <URL of pull-request.repository> refs/heads/<target_branch>`) equals `verified_target_base`. Every remote query targets `pull-request.repository` explicitly; if that identity cannot be established, stop. A moved target branch or a different head — stop and explain: the approval would not be bound to the verified identity, and the route (a new PR_PREPARATION / VERIFICATION pass) is not implemented (8.9.2). If the remote cannot be read — stop.
4. **The package corresponds to the current state.** `pending_human_decision.package` contains exactly the `approval_scope` elements with their identities (`workflow-state.md` §4): `{ type: implementation-changeset, version, changeset_commit }`; `{ record }` of the CODE_REVIEW report and, if SECURITY_REVIEW was applicable, of the SECURITY_REVIEW report, both for this identity; `{ record }` of the `reconciliation-report` if RECONCILIATION was applicable; `{ record }` of the `verification-report`; `{ record }` of the ENTRY `applicability-decision` of SECURITY_REVIEW and RECONCILIATION; `{ record }` of the `pull-request`. The `refs` of the last `human_decision.requested` event match `package`.
5. **Mandatory rules.** No violation of a mandatory Foundation rule (in particular a Security Rule) is resolved by a `risk-acceptance-record` (WF-V2-01, RR-14).
6. **Previous approvals.** `approvals.HUMAN_PR_APPROVAL` is absent or `null` (a repeated pass is not implemented). `approvals.HUMAN_SPEC_APPROVAL` and `approvals.HUMAN_TECHNICAL_APPROVAL` are `VALID`.

#### 8.9.2. REQUEST_CHANGES — Not Supported

The targets of `decisions.REQUEST_CHANGES` (PR_PREPARATION and the earlier stages) all require re-entry with a new version or record of an already registered result, which this MVP does not implement. Reject the command after the check of 8.9.1, before any changes; `APPROVE` and `CANCEL` are available. Do not redirect the decision.

#### 8.9.3. APPROVE

**Check:** as section 8.3.1, step 3 — `comment` does not change requirements or the changes.

**Approval boundaries.** The approval is bound to the exact Scope elements (`artifact-rules.md` §5). By approving the Scope, which includes the `verification-report` with its Findings Disposition and `follow_up`, the human finally approves the classification of the NON_BLOCKING and FOLLOW_UP findings as listed there (`stages.md` §12, OD-11). The MVP does not support approving only a part of the FOLLOW_UP items: a human who does not accept the list uses `CANCEL` (`REQUEST_CHANGES` is not supported). The `backlog-entry` records are created later by INTEGRATION_CONFIRMATION (`workflow-policies.md` §7), not here. The approval does not merge and does not push.

**`state.yaml`:**

- `approvals.HUMAN_PR_APPROVAL` (`workflow-state.md` §4): `decision: APPROVE`, `decided_by: <by>`, `decided_at: <date -u>`, `scope` — the package elements of 8.9.1, step 4, `package: []` (all reviewed reports are part of the Scope of this gate), `comment`, `approval_state: VALID`, `previous: []`;
- `status: APPROVED` for the current `implementation-changeset` (the only document of the Scope); `version` and `currency` do not change;
- `pending_human_decision: null`;
- Quality Loops reset, as in section 8.3.2, for the producer and review stages of the Scope (IMPLEMENTATION, CODE_REVIEW, SECURITY_REVIEW, RECONCILIATION, VERIFICATION, PR_PREPARATION);
- `current_stage: MERGE_READINESS_CHECK` (the gate's `decisions.APPROVE`), `story_status: ACTIVE`, `execution_state: READY`, `updated_at`; `last_stage_verdict`, `blocker`, `records`, `git` do not change.

**`history.jsonl`** — the `human_decision.applied` event, as in section 8.3.2: `stage: HUMAN_PR_APPROVAL`, `refs` — the elements of `package`, `details: { decision: APPROVE, to_stage: MERGE_READINESS_CHECK, scope, approval_state: VALID, comment, retry_reset, mode }`.

Do not start MERGE_READINESS_CHECK. The next `next` executes it (section 7.13.2).

**CANCEL** — section 8.5 with `current_stage: HUMAN_PR_APPROVAL` and the gate's `decisions.CANCEL`. The open PR is not closed by the Orchestrator; the human closes it.

### 8.10. HUMAN_MERGE

A human integrates the approved changes into the target branch (`stages.md` §12, Q5, Q8, OD-12 step 6). It is a delivery action (`type: human_action`), not an approval. **Only the human performs the merge**, outside the Orchestrator, with the strategy the human chooses; the Orchestrator never merges, pushes or edits the PR. `decide` only records the human's decision from the stage's `decisions`: `CONFIRM_MERGED`, `REQUEST_CHANGES`, `CANCEL`. Human decisions that are not an approval are recorded only in History (`workflow-state.md` §4). The stage is reached only after MERGE_READINESS_CHECK `PASS` (section 1, IMP-D01).

#### 8.10.1. Check

Instead of steps 3–4 of section 8.2. The first violation — explain and stop, changing nothing.

1. **Waiting state.** `current_stage: HUMAN_MERGE`; `pending_human_decision.kind: HUMAN_ACTION`, `stage: HUMAN_MERGE`; `decision` is in `options` and in the stage's `decisions`; `blocker: null`; the `refs` of the last `human_decision.requested` event match `package`.
2. **Preconditions** (`stages.md`): the last `stage.completed` event is MERGE_READINESS_CHECK with `verdict: PASS` or INTEGRATION_CONFIRMATION with `verdict: CHANGES_REQUIRED`, in both cases `to_stage: HUMAN_MERGE`; `approvals.HUMAN_PR_APPROVAL` is `VALID` and its `scope` contains the current `implementation-changeset` identity (`version`, `git.changeset_commit`), `records.verification-report` and `records.pull-request`.
3. **Merge Protection Status.** `git.merge_protection_status` is one of `ACTIVE`, `NOT_SUPPORTED`, `UNAVAILABLE`, `DISABLED` (`merge_protection_statuses`). `DISABLED` — stop (MERGE_READINESS_CHECK could not have passed). `UNAVAILABLE` — only with a human-approved integration order recorded in History. A missing or another value — stop; it is never read as `ACTIVE`.

#### 8.10.2. REQUEST_CHANGES and CANCEL

- `REQUEST_CHANGES` needs `target` and a non-empty `reason`. `target=MERGE_READINESS_CHECK` (the merge is impossible because the target branch changed, or a manual procedure found a change after the check): apply — `current_stage: MERGE_READINESS_CHECK`, `story_status: ACTIVE`, `execution_state: READY`, `pending_human_decision: null`; `approvals`, `artifacts`, `records`, `git` do not change; the `human_decision.applied` event with `details: { decision: REQUEST_CHANGES, to_stage: MERGE_READINESS_CHECK, reason, kind: HUMAN_ACTION, mode }`. MERGE_READINESS_CHECK is a check without outputs, so its repeated run needs no new versions. `target=PR_PREPARATION` or `IMPLEMENTATION` require re-entry with new records or versions, which is not implemented: reject before any changes and explain.
- `CANCEL` — section 8.5 with `current_stage: HUMAN_MERGE` and the stage's `decisions.CANCEL`. The PR is not closed by the Orchestrator.

#### 8.10.3. CONFIRM_MERGED

A human's claim alone is not a confirmation. Before applying, check with read-only queries against `pull-request.repository` (as in section 8.9.1, step 3):

- the PR is merged: `gh pr view <number> --repo <repository> --json state,mergedAt,mergeCommit,baseRefName,headRefOid` — `state: MERGED`, `baseRefName` = the record's `target_branch`, `headRefOid` = `git.changeset_commit` (the approved head was merged), and `mergeCommit` exists;
- the merge commit is on the remote `target_branch` (it is the tip or an ancestor of the tip read with `git ls-remote` / the platform API);
- `target_before_merge` is **not** derived here. The parent of the final merge result is the target's pre-merge tip only for a merge commit or a squash merge; for a rebase merge of several commits it is the previous rebased commit. The strategy-aware determination belongs to Integration Match in INTEGRATION_CONFIRMATION (`workflow-policies.md` §9, "Independence from the merge strategy"), which records it in `integration-record`.

Any check fails or cannot be performed — change nothing, explain, stop. The Orchestrator does not assume a merge commit and does not require that the merge commit equals `changeset_commit`: the strategy (merge commit, squash, rebase) may change commit identity. Whether exactly the approved Changeset Content Identity was integrated (Integration Match), the target identity at the moment of the merge and whether Post-Merge Validation is needed are decided by INTEGRATION_CONFIRMATION, not here (`workflow-state-model.md` §7, §9; OD-16). For a controlled manual procedure (`NOT_SUPPORTED` or `UNAVAILABLE` with an approved order), the human's `CONFIRM_MERGED` also states that the target branch had not changed after the check; INTEGRATION_CONFIRMATION verifies it by Base Comparison.

**`state.yaml`:** `current_stage: INTEGRATION_CONFIRMATION` (the stage's `decisions.CONFIRM_MERGED`), `story_status: ACTIVE`, `execution_state: READY`, `pending_human_decision: null`, `updated_at`; `approvals`, `artifacts`, `records`, `git` and `last_stage_verdict` do not change.

**`history.jsonl`** — the `human_decision.applied` event: `actor: { kind: human, id: <by> }`, `stage: HUMAN_MERGE`, `refs` — the elements of `package`, `details: { decision: CONFIRM_MERGED, kind: HUMAN_ACTION, to_stage: INTEGRATION_CONFIRMATION, repository, pr_number, merge_commit, target_branch, merged_at, merge_protection_status, mode }` (`stage-io-map.yaml`: CONFIRM_MERGED and the merge commit — in History).

Do not start INTEGRATION_CONFIRMATION. The next `next` with `workspace=<path>` executes it (section 7.13.4).

### 8.11. Recovery `RESUME` — SOLUTION_DESIGN, Escalation Required (minimal)

The only Recovery implemented in the MVP (`workflow-state-model.md` §9, Recovery Actions and Procedure). It applies when SOLUTION_DESIGN returned `BLOCKED` (Escalation Required, `workflow-policies.md` §2) and the human has recorded the escalation decision in the canonical Foundation document. The human decides; the Orchestrator checks the preconditions and applies the decision. A decision does not remove a blocker by itself: the evidence must exist and be committed. The decision in the pilot is a simulation, as at a gate.

#### 8.11.1. Check

Instead of steps 2–4 of section 8.2. The first violation — explain and stop, changing nothing; the Story remains `BLOCKED` (`workflow-state-model.md` §9, Procedure).

1. **Arguments.** `decision=RESUME`, non-empty `by`, `reason` and `evidence`; `target` and `comment` are not passed. Another decision — explain that only `RESUME` is implemented for this Recovery, and stop.
2. **State.** `story_status: BLOCKED`, `pending_human_decision: null`, `current_stage: SOLUTION_DESIGN`, `execution_state: COMPLETED`, `last_stage_verdict: BLOCKED`; `blocker.source: STAGE`, `blocker.reason` begins with `Escalation Required`. Any other blocking reason, source or stage — Recovery for it is not implemented: explain and stop.
3. **History.** The last two events are `stage.completed` SOLUTION_DESIGN with `details.verdict: BLOCKED` and `story.blocked` SOLUTION_DESIGN whose `details.reason` equals `blocker.reason`. `artifacts.solution-design` is absent (the blocked attempt wrote no file).
4. **The blocker is resolved** (`workflow-state-model.md` §9, step 3: "the escalation decision is recorded in the canonical source"):
   - `evidence` is under `docs/architecture/` or `docs/product/`, is tracked (`git ls-files --error-unmatch <evidence>`) and has no uncommitted changes (`git status --porcelain -- <evidence>` is empty);
   - its last commit `git log -1 --format="%h|%cI" -- <evidence>` is later than the `ts` of the `story.blocked` event: the decision was committed after the blocking;
   - the file contains the decision that answers `blocker.needed` (for a dependency — an entry naming the dependency, its purpose and this Story). The Orchestrator only confirms that such an entry exists and quotes its location; it does not judge whether the decision is sufficient — the new SOLUTION_DESIGN attempt assesses the design again and may return `BLOCKED` again.
5. **Approvals and inputs.** `approvals.HUMAN_SPEC_APPROVAL` is `VALID` and its `story-spec` version is the current one; the inputs of SOLUTION_DESIGN pass section 7.2, steps 5 and 7 (the change of the Foundation must not have changed a recorded source of a registered artifact). Otherwise — stop: Revalidation and RECONFIRM are not implemented.
6. **The reason does not change requirements** — as in section 8.7.3.

#### 8.11.2. Application

One Orchestrator step (section 8.2, last paragraphs), with one decision time.

**`state.yaml`:** `story_status: ACTIVE`, `execution_state: READY`, `blocker: null`, `updated_at`; `current_stage` (SOLUTION_DESIGN), `last_stage_verdict` (`BLOCKED`: a human's decision is not a Stage Verdict), `pending_human_decision` (`null`), `artifacts`, `records`, `approvals`, `retry` and `git` do not change. `RESUME` does not reset counters (`workflow-state-model.md` §10).

**`history.jsonl`** — append one `human_decision.applied` event; the `stage.completed` and `story.blocked` events stay unchanged:

```json
{"ts":"<date -u>","story_id":"US-001","actor":{"kind":"human","id":"<by>"},"event":"human_decision.applied","stage":"SOLUTION_DESIGN","refs":["story-spec@1"],"details":{"decision":"RESUME","kind":"RECOVERY","to_stage":"SOLUTION_DESIGN","resolved_blocker":"<blocker.reason verbatim>","evidence":[{"path":"<evidence>","commit":"<short commit>"}],"reason":"<reason verbatim>","mode":"pilot"}}
```

`refs` — the current `story-spec@<N>` (and `impact-analysis@<N>` if it is passed according to section 7.2, step 7).

Do not start SOLUTION_DESIGN. The next `next` executes a new attempt: the entry trigger is this event (section 7.2, step 6), `attempt` is the count of `stage.started` SOLUTION_DESIGN + 1, `version` stays `1`, and `reason` is passed verbatim (section 7.3).

## 9. Constraints

- The Orchestrator writes only the pilot's `state.yaml` and `history.jsonl`: in `init`, at steps 7.2 (only `RUNNING` → `INTERRUPTED`), 7.4, 7.5.2, 7.7, 7.8, 8.3.2, 8.4.2, 8.5.3, 8.7.3, 8.8.3 (HUMAN_TECHNICAL_APPROVAL), 8.9.3 (HUMAN_PR_APPROVAL), 8.10.2–8.10.3 (HUMAN_MERGE), 8.11.2 (Recovery `RESUME`), 7.11.3, 7.11.6 (IMPLEMENTATION, CODE_REVIEW) 7.12.6 (SECURITY_REVIEW, RECONCILIATION, VERIFICATION) and 7.13 (PR_PREPARATION, MERGE_READINESS_CHECK, INTEGRATION_CONFIRMATION, `DONE`). The only record it writes itself is `cancellation-record` in 8.5.2. `status` writes nothing.
- An artifact file is written only by the Stage Skill during 7.5. The Orchestrator does not create, edit or delete it.
- Write nothing to `docs/stories/`, `docs/contracts/`, `docs/workflow/`, the Foundation or `.claude/skills/`.
- Do not invent statuses, events, state fields or transition rules. Report to the human what is missing in the contracts.
- History is append-only: lines are not changed or deleted.
- Do not fix the state silently and do not restore it automatically.
- Start only the stages from table 7.1. CLARIFICATION is executed inline in the main session, without a subagent. SPECIFICATION, SPEC_REVIEW, IMPACT_ANALYSIS, SOLUTION_DESIGN, API_DESIGN, DB_DESIGN, DESIGN_REVIEW, IMPLEMENTATION_PLANNING, PLAN_REVIEW, IMPLEMENTATION, CODE_REVIEW, SECURITY_REVIEW, RECONCILIATION and VERIFICATION are each executed in one separate `general-purpose` subagent; every review and verification stage in a fresh one, never the IMPLEMENTATION subagent. Do not pass a historical version of `impact-analysis` after `NOT_APPLICABLE` as an input (section 7.1), do not pass `impact-analysis` to API_DESIGN at all, and pass `impact`, `api` and `db` to DB_DESIGN, DESIGN_REVIEW, IMPLEMENTATION_PLANNING and PLAN_REVIEW only according to section 7.2, steps 7–9. Do not start IMPLEMENTATION_PLANNING or PLAN_REVIEW without the Testing Strategy (section 7.2, steps 10–11). Do not start IMPLEMENTATION or CODE_REVIEW outside the pilot worktree checks of section 7.11.1, and IMPLEMENTATION without `allow_commit=IMPLEMENTATION`. Do not start VERIFICATION without a verification workspace that passes section 7.12.3. Do not start PR_PREPARATION without `allow_remote=PR_PREPARATION repository=<owner/name>`, or INTEGRATION_CONFIRMATION without a workspace that passes section 7.13.4. Do not start additional subagents.
- Apply the human's decisions only in `decide` (section 8), by the human's decision from the command arguments. Do not simulate or guess the human's decisions, do not substitute `by`. Do not apply decisions of other gates (only HUMAN_SPEC_APPROVAL, HUMAN_TECHNICAL_APPROVAL and HUMAN_PR_APPROVAL), RECONFIRM, `REQUEST_CHANGES` at HUMAN_TECHNICAL_APPROVAL and HUMAN_PR_APPROVAL, `ROUTE_BACK` and any Recovery other than section 8.11. The Orchestrator does not merge, push or close PRs itself; a push and one PR creation happen only inside PR_PREPARATION under section 7.13.1. At HUMAN_MERGE the Orchestrator only records `CONFIRM_MERGED`, `REQUEST_CHANGES` (to MERGE_READINESS_CHECK) or `CANCEL` after the checks of section 8.10. Do not perform a Loopback through a gate automatically.
- A decision in the pilot is a simulation of a Human Gate. Do not present it as an approval or cancellation of the Story in the official Workflow.
- Do not invoke Stage Skills with the Skill tool.
- Change `state.yaml` only according to section 7.10.
- The Orchestrator itself does not run `git add`, `git commit`, `git push`, `checkout`, `switch`, `worktree`, `branch`, `reset` or other writing or destructive Git operations; it uses only the read-only commands of section 7.11.1. Commits are made only by the IMPLEMENTATION subagent under section 7.11.2.
