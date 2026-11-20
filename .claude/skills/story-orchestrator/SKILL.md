---
name: story-orchestrator
description: Story Orchestrator MVP (Phase 6, increments 6.4.1–6.4.3; Phase 7, increments 7.4.1–7.4.2; Phase 8, increments 8.3.2, 8.4 and the API_DESIGN and DB_DESIGN integration). Initializes a pilot Story outside the registered Workflow, shows its status with a State and History check, executes one stage per run (CLARIFICATION inline, SPECIFICATION, SPEC_REVIEW, IMPACT_ANALYSIS, SOLUTION_DESIGN, API_DESIGN and DB_DESIGN in a subagent) in Step Mode up to DESIGN_REVIEW, supports re-entry into IMPACT_ANALYSIS after a Loopback from SOLUTION_DESIGN, applies the human's decisions at HUMAN_SPEC_APPROVAL (APPROVE, REQUEST_CHANGES, CANCEL) and at the Loopback confirmation from IMPACT_ANALYSIS, SOLUTION_DESIGN and API_DESIGN (RESUME, CANCEL). Started only manually with mode=pilot.
argument-hint: "init|status|next story=<ID> mode=pilot | decide story=<ID> mode=pilot decision=APPROVE|REQUEST_CHANGES|CANCEL|RESUME|ROUTE_BACK by=<id> [target=<stage>] [reason=\"...\"] [comment=\"...\"]"
disable-model-invocation: true
user-invocable: true
---

# Story Orchestrator — MVP (increments 6.4.1–6.4.3, 7.4.1–7.4.2, 8.3.2, 8.4, API_DESIGN, DB_DESIGN)

## 1. Purpose

The Orchestrator maintains the Workflow State and History of one Story: `state.yaml` and `history.jsonl` are changed **only** by it (`docs/contracts/workflow-state.md` §1). Routes, verdicts, loopbacks and gates are defined only by `docs/workflow/stage-map.yaml`; the state format — by `docs/contracts/workflow-state.md`; paths — by `docs/contracts/artifact-registry.yaml`. These files are authoritative; in case of a discrepancy, they apply, not this Skill.

The Skill works in the main session.

**Language** (`AGENTS.md`):
- everything addressed to the human — explanations, status output, requests for decisions, reports — is in Ukrainian;
- everything written to files — `state.yaml`, `history.jsonl`, `cancellation-record` and the generated Story artifacts — is in English;
- text provided by the human (`reason`, `comment`, `by`, interview answers) is recorded verbatim in its original language (section 7.10).

**Boundaries.** Commands: `init`, `status`, `next`, `decide`.

- `next` executes one stage per run (Step Mode): CLARIFICATION, SPECIFICATION, SPEC_REVIEW, IMPACT_ANALYSIS, SOLUTION_DESIGN, API_DESIGN or DB_DESIGN.
- After `PASS` of SPEC_REVIEW, the Story stops at HUMAN_SPEC_APPROVAL and waits for the human's decision.
- After `PASS` or `NOT_APPLICABLE` of IMPACT_ANALYSIS, the Story moves to SOLUTION_DESIGN.
- After `PASS` of SOLUTION_DESIGN, the Story moves to API_DESIGN.
- After `PASS` or `NOT_APPLICABLE` of API_DESIGN, the Story moves to DB_DESIGN.
- After `PASS` or `NOT_APPLICABLE` of DB_DESIGN, the Story moves to DESIGN_REVIEW. `next` does not execute that stage yet.
- `decide` applies the human's decisions:
  - `APPROVE`, `REQUEST_CHANGES` or `CANCEL` at HUMAN_SPEC_APPROVAL (section 8);
  - `RESUME` or `CANCEL` at the Loopback confirmation (`LOOPBACK_CONFIRMATION`), when IMPACT_ANALYSIS, SOLUTION_DESIGN, API_DESIGN or DB_DESIGN returned `CHANGES_REQUIRED` with a target through a gate (section 8.7). `ROUTE_BACK` is rejected without changes (section 8.7.2).
- Re-entry after a return is supported:
  - for SPECIFICATION — after a loopback from SPEC_REVIEW or REQUEST_CHANGES at the gate (section 7.3.1);
  - for IMPACT_ANALYSIS — after a loopback from SOLUTION_DESIGN (section 7.3.2).
- `REQUEST_CHANGES` with `target=CLARIFICATION` is rejected without changes (section 8.4.1).
- A Loopback without a gate from API_DESIGN (to SOLUTION_DESIGN or IMPACT_ANALYSIS) or from DB_DESIGN (to API_DESIGN, SOLUTION_DESIGN or IMPACT_ANALYSIS) is routed, but the target is not executed (section 7.8, API-D01, DB-D01).
- Not implemented: execution of other stages, re-entry into CLARIFICATION and SOLUTION_DESIGN with a new version, re-entry into IMPACT_ANALYSIS after a Loopback from API_DESIGN or DB_DESIGN, re-entry into API_DESIGN after `PASS` (only `api-design` v1), re-entry into DB_DESIGN after `PASS` (only `db-design` v1), DESIGN_REVIEW, other Human Gates, RECONFIRM, a Loopback through a gate (`ROUTE_BACK`), Recovery and Auto Mode. If the human asks for something else, explain that it does not exist yet, and stop.

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

git:                                # the pilot has no feature branch; branch naming is not defined
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
   - if the last routing event is `stage.completed` API_DESIGN or DB_DESIGN with `verdict: CHANGES_REQUIRED` and a `to_stage` without `crosses_gates` (`API_DESIGN`, `SOLUTION_DESIGN` or `IMPACT_ANALYSIS`) — the marker «повернення з <stage> без підтримки повторного входу в MVP: `next` цю стадію не виконає» (section 7.8, API-D01, DB-D01);
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
- `execution_state: RUNNING` ⇔ the last event is `stage.started` or `human_decision.applied` for `current_stage`. The exception is `human_decision.applied` with `details.decision: RESUME` (`LOOPBACK_CONFIRMATION`, section 8.7.3): after it, `execution_state: READY`;
- `execution_state: AWAITING_HUMAN` ⇔ the last event is `human_decision.requested` for `current_stage`.

**Files** (`workflow-state-model.md` §12, step 2):

- each version from `artifacts` exists at the pilot path, and its front matter has the same `artifact_type`, `version` and `story_id`;
- each record from `records` exists, and its front matter has the same `record_id` and `story_id`. For `records.applicability-decision`, this is each value `<STAGE>: { ENTRY: <record_id> }`; its front matter also has `target_stage: <STAGE>` and `kind: ENTRY`.

**Git Binding** is not checked while all `git` fields are `null` (a pilot without a feature branch). Say so in the status.

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

The output and the owner are from `artifact-registry.yaml` (`produced_by`), the inputs — from `stage-io-map.yaml`. SPEC_REVIEW checks both inputs (`stage-map.yaml` → `reviews: [CLARIFICATION, SPECIFICATION]`). `codebase`, `architecture-foundation` and `product-foundation` for IMPACT_ANALYSIS, SOLUTION_DESIGN, API_DESIGN and DB_DESIGN are sources, not registered artifacts: the Stage Skill reads them itself (`impact-analysis/SKILL.md` §2, `solution-design/SKILL.md` §2, `api-design/SKILL.md` §2, `db-design/SKILL.md` §2). Allowed verdicts and routes — only from `stage-map.yaml` (section 7.6).

**Conditional input `impact-analysis`.** The condition `when: applicable` (`stage-io-map.yaml`) is determined by the valid `applicability-decision` of the stage, that is, `records.applicability-decision.IMPACT_ANALYSIS.ENTRY` with `result: APPLICABLE`, not by the existence of the artifact. If the valid decision is `NOT_APPLICABLE`, a previously registered version of `impact-analysis` remains in `artifacts` as a historical one, but it is not an input of any stage, and the Orchestrator does not pass it (section 7.2, step 7).

### 7.2. Pre-Run Check

1. **Check the arguments** and consistency (section 6). A discrepancy — stop, as in `status`.
2. **`story_status` must be `ACTIVE`.** Otherwise — show the state and stop. For `pending_human_decision.kind: GATE` at HUMAN_SPEC_APPROVAL and for `kind: LOOPBACK_CONFIRMATION` at IMPACT_ANALYSIS, SOLUTION_DESIGN, API_DESIGN or DB_DESIGN, say that the human makes the decision with the `decide` command (sections 8, 8.7). For `CANCELLED` — the Story is cancelled in the pilot, the state is terminal and is not reopened. Other human decisions and Recovery are not implemented yet.
   A special case — `WAITING_FOR_HUMAN` with `pending_human_decision.kind: CLARIFICATION_ANSWERS`: a new `next` run means that the interview of this attempt was interrupted together with the session. **MVP limitation:** continuing the interview in a new session is not supported, and the contract does not define a transition from `AWAITING_HUMAN` to `INTERRUPTED` during session recovery (`workflow-state-model.md` §12 describes only `RUNNING`). Change nothing: explain this to the human and stop.
3. **`current_stage` must be in table 7.1.** Another stage — "not supported yet", stop.
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
6. **Approval and entry trigger (IMPACT_ANALYSIS, SOLUTION_DESIGN, API_DESIGN and DB_DESIGN).** The precondition of these stages is an approved `story-spec` (`stages.md`, IMPACT_ANALYSIS and SOLUTION_DESIGN; `workflow-state-model.md` §6, step 4). For API_DESIGN and DB_DESIGN, `solution-design` must also be current: it is checked as an input in step 5:
   - `approvals.HUMAN_SPEC_APPROVAL` is not `null`, `decision: APPROVE`, `approval_state: VALID`;
   - the current version of `story-spec` in `artifacts` has `status: APPROVED` and matches the `story-spec` version in `approvals.HUMAN_SPEC_APPROVAL.scope`;
   - **the entry trigger** is the last History event with `details.to_stage`, as in section 7.3.1. Its `to_stage` must equal `current_stage`. Allowed triggers:

     | Stage | Trigger |
     |---|---|
     | IMPACT_ANALYSIS | `human_decision.applied` HUMAN_SPEC_APPROVAL with `decision: APPROVE` (first entry); `stage.completed` SOLUTION_DESIGN with `verdict: CHANGES_REQUIRED` and `loopback_target: IMPACT_ANALYSIS` (re-entry, section 7.3.2); `human_decision.applied` IMPACT_ANALYSIS with `decision: RESUME` (section 8.7.3) |
     | SOLUTION_DESIGN | `stage.completed` IMPACT_ANALYSIS with `verdict: PASS` or `NOT_APPLICABLE`; `human_decision.applied` SOLUTION_DESIGN with `decision: RESUME` |
     | API_DESIGN | `stage.completed` SOLUTION_DESIGN with `verdict: PASS`; `human_decision.applied` API_DESIGN with `decision: RESUME` |
     | DB_DESIGN | `stage.completed` API_DESIGN with `verdict: PASS` or `NOT_APPLICABLE`; `human_decision.applied` DB_DESIGN with `decision: RESUME` |

     Another trigger means a route that this MVP does not execute. In particular, `stage.completed` API_DESIGN or DB_DESIGN with `verdict: CHANGES_REQUIRED` and a `loopback_target` without `crosses_gates` (section 7.8, API-D01, DB-D01): explain that re-entry into the target after a Loopback from that stage is not implemented in the MVP, and that the Story cannot continue with `next`;
   - SOLUTION_DESIGN: `artifacts.solution-design` is absent. The MVP does not support re-entry into SOLUTION_DESIGN with a new version (`solution-design/SKILL.md` §3, step 1);
   - API_DESIGN: `artifacts.api-design` is absent. The MVP supports only `api-design` v1 (`api-design/SKILL.md` §3, step 1);
   - DB_DESIGN: `artifacts.db-design` is absent. The MVP supports only `db-design` v1 (`db-design/SKILL.md` §3, step 1).

   Not met — change nothing, explain to the human and stop.
7. **IMPACT_ANALYSIS applicability (SOLUTION_DESIGN and DB_DESIGN).** Determines whether `impact-analysis` is passed (section 7.1, "Conditional input"):
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
8. **API_DESIGN applicability (only DB_DESIGN).** Determines whether `api-design` is passed, by the same rules as step 7:
   - `records.applicability-decision.API_DESIGN.ENTRY` exists, and the record file exists;
   - its `subjects` are the current versions of `story-spec` and `solution-design`;
   - the last `stage.completed` event of the API_DESIGN stage has `refs` with this `<record_id>` and a `verdict` consistent with the record's `result`: `PASS` ⇔ `APPLICABLE`, `NOT_APPLICABLE` ⇔ `NOT_APPLICABLE`;
   - **`APPLICABLE`:** `artifacts.api-design` exists; the `refs` of the same event contain `api-design@1`; step 5 is performed for it; `api` = the path of this version;
   - **`NOT_APPLICABLE`:** `api` is not passed; an `api-design` file, if any, is not an input.

   Any other combination is a state discrepancy. Change nothing, explain to the human what exactly does not match (file, field, event), and stop.

### 7.3. Attempt Parameters

Computed by the Orchestrator, not by the Stage Skill:

| Parameter | How it is determined |
|---|---|
| `attempt` | The number of `stage.started` events for `current_stage` in History + 1 |
| `version` | For a document: the current version of the output type in `artifacts` + 1; if there is none — `1` |
| `record_id` | For a record: `<record_type>/<NNNN>`, where `record_type` is `review-report` (SPEC_REVIEW) or `applicability-decision` (IMPACT_ANALYSIS, API_DESIGN and DB_DESIGN; one numbering of the type for the whole Story), and `NNNN` is the next number of the type in the Story (`artifact-rules.md` §2): the highest number `<record_type>/…` in the `refs` of the `stage.completed` events of History + 1, four digits; if there are none — `0001` |
| `output` | The path of the output type according to the registry with the pilot prefix (section 3). For SPEC_REVIEW — the record `records/review-report/<NNNN>.md`. For IMPACT_ANALYSIS — the document `artifacts/impact-analysis/v<version>.md`. For SOLUTION_DESIGN — `artifacts/solution-design/v<version>.md`. For API_DESIGN — `artifacts/api-design/v1.md`. For DB_DESIGN — `artifacts/db-design/v1.md` |
| `record_output` | Only IMPACT_ANALYSIS, API_DESIGN and DB_DESIGN: `records/applicability-decision/<NNNN>.md` under the pilot root |
| `clarification`, `spec` | The paths of the current versions of `clarification-record` and `story-spec` (for SPECIFICATION — only `clarification`; for IMPACT_ANALYSIS, SOLUTION_DESIGN, API_DESIGN and DB_DESIGN — only `spec`) |
| `design` | Only API_DESIGN and DB_DESIGN: the path of the current version of `solution-design` |
| `api` | Only DB_DESIGN and only according to section 7.2, step 8 (`APPLICABLE`): the path of `api-design` v1 |
| `impact` | Only SOLUTION_DESIGN and DB_DESIGN, and only according to section 7.2, step 7 (`APPLICABLE`): the path of the current version of `impact-analysis` |
| `previous_reports` | For SPEC_REVIEW: the paths of the previous `review-report` files of this stage from History (`stage.completed` of the SPEC_REVIEW stage); if there are none — do not pass |
| `previous`, `reason` | SPECIFICATION: only for `version > 1` — according to section 7.3.1. IMPACT_ANALYSIS — according to section 7.3.2 |
| `reason` (SOLUTION_DESIGN, API_DESIGN, DB_DESIGN) | Only if the entry trigger (section 7.2, step 6) is `human_decision.applied` of the current stage with `details.decision: RESUME` (section 8.7.3). The value is the `details.reason` of this event verbatim. Otherwise do not pass. `version` is always `1` |

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

**Unregistered Working Draft.** Before starting any attempt, check whether the `output` file (for IMPACT_ANALYSIS, API_DESIGN and DB_DESIGN — also `record_output`) exists while it is not in `state.yaml` → `artifacts` or `records`. If it exists, it is a Working Draft of a previous attempt (`artifact-rules.md` §3, item 1). Do not overwrite, delete or register it. Change nothing in the state: show the path, explain that an unregistered Working Draft was found, and stop. The human makes the decision outside the Skill.

> **Temporary MVP limitation.** The contract allows a repeated attempt to overwrite a Working Draft. This Skill deliberately does not do that until Recovery is implemented. The Working Draft contract does not change.

### 7.4. Attempt Start

One Orchestrator step:

- `state.yaml`: `execution_state: RUNNING`, `updated_at`;
- `history.jsonl`: the `stage.started` event.

```json
{"ts":"<date -u>","story_id":"US-001","actor":{"kind":"orchestrator","id":"story-orchestrator"},"event":"stage.started","stage":"SPECIFICATION","refs":["clarification-record@1"],"details":{"attempt":1,"version":1,"output":"docs/pilots/orchestrator/US-001/artifacts/story-spec/v1.md","mode":"pilot"}}
```

`refs` are the input artifacts of the attempt (`<type>@<version>`); for CLARIFICATION — `[]`. For SPEC_REVIEW, `details` contain `record_id` instead of `version`. For IMPACT_ANALYSIS, `refs: [story-spec@<N>]`, and for a re-entry with `version > 1` — also `impact-analysis@<N-1>`. `details` contain `version`, `output`, `record_id` and `record_output`, as well as `previous` and `reason`, if they exist (section 7.3.2). For SOLUTION_DESIGN, `refs` consist of `story-spec@<N>`, `impact-analysis@<M>` (only if `impact` is passed) and the valid IMPACT_ANALYSIS `applicability-decision` that determined the input. `details` contain `version`, `output` and `reason`, if it exists. For API_DESIGN, `refs: [story-spec@<N>, solution-design@<M>]`; `details` contain `version`, `output`, `record_id`, `record_output` and `reason`, if it exists. For DB_DESIGN, `refs` consist of `story-spec@<N>`, `solution-design@<M>`, `impact-analysis@<L>` and `api-design@1` (each only if passed) and the valid IMPACT_ANALYSIS and API_DESIGN `applicability-decision` records that determined the inputs; `details` as for API_DESIGN. For a re-entry into SPECIFICATION (7.3.1), `refs` additionally contain the previous version (`story-spec@1`), and for a loopback from SPEC_REVIEW — also the trigger record (`review-report/0001`). `details` contain `previous` and `reason`. For the "human decision" trigger there is no record: the decision is identified by `reason` (`actor.id`, `ts`).

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

#### 7.5.3. SPECIFICATION, SPEC_REVIEW, IMPACT_ANALYSIS, SOLUTION_DESIGN, API_DESIGN and DB_DESIGN — Subagent

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

3. Wait for the **actual completion** of the subagent and its final response (for a background run — the completion notification). A background start does not mean that the subagent has finished. While it is running:
   - do not process its result and do not guess it;
   - do not change `state.yaml`, `history.jsonl` and the pilot files;
   - do not start other subagents and stages.
4. Only after completion, take `git status --porcelain --untracked-files=all` again (snapshot B) and the `git hash-object` of the same files. The difference A→B must consist exactly of the new files that correspond to the elements of the envelope's `outputs`; if `outputs` are empty, there is no difference. For SPECIFICATION, SPEC_REVIEW and SOLUTION_DESIGN, this is the `output` file. For IMPACT_ANALYSIS, API_DESIGN and DB_DESIGN, `{ record: <record_id> }` corresponds to `record_output`, and `{ type: impact-analysis | api-design | db-design, version }` — to `output`. The hashes of registered files do not change (`artifact-rules.md` §3, item 2). Any other change — the result is rejected (`INVALID_RESULT`) with a list of files. Do not roll anything back: the human makes the decision.
5. The envelope is the last `result:` block in the subagent's response. A failure of the Agent tool — `FAILED`. A response without an envelope — `INVALID_RESULT`.

### 7.6. Result Envelope Check

The Orchestrator accepts the result only when **everything** is met (`workflow-state.md` §6):

- there is a `result` block with the fields `story_id`, `stage`, `attempt`, `verdict`, `outputs`, `loopback_target`, `blocker`, `summary`;
- `story_id`, `stage`, `attempt` match the attempt parameters;
- `verdict` is allowed for the stage according to `stage-map.yaml`: `optional: false` → not `NOT_APPLICABLE`; `CHANGES_REQUIRED` — only if the stage's `loopbacks` are not empty. CLARIFICATION: `PASS | BLOCKED`. SPECIFICATION and SPEC_REVIEW: `PASS | CHANGES_REQUIRED | BLOCKED`. IMPACT_ANALYSIS, API_DESIGN and DB_DESIGN (`optional: true`): `PASS | NOT_APPLICABLE | CHANGES_REQUIRED | BLOCKED`. SOLUTION_DESIGN (`optional: false`): `PASS | CHANGES_REQUIRED | BLOCKED`;
- `loopback_target`: for `CHANGES_REQUIRED` — a target from the stage's `loopbacks` (SPECIFICATION → `CLARIFICATION`; SPEC_REVIEW → `SPECIFICATION` or `CLARIFICATION`; IMPACT_ANALYSIS → `SPECIFICATION` or `CLARIFICATION`, both with `crosses_gates`; SOLUTION_DESIGN → `IMPACT_ANALYSIS` without `crosses_gates` or `SPECIFICATION` or `CLARIFICATION` with `crosses_gates`; API_DESIGN → `SOLUTION_DESIGN` or `IMPACT_ANALYSIS` without `crosses_gates` or `SPECIFICATION` or `CLARIFICATION` with `crosses_gates`; DB_DESIGN → `API_DESIGN`, `SOLUTION_DESIGN` or `IMPACT_ANALYSIS` without `crosses_gates` or `SPECIFICATION` or `CLARIFICATION` with `crosses_gates`); otherwise `null`;
- for SPEC_REVIEW — section 7.6.1, for IMPACT_ANALYSIS — section 7.6.2, for API_DESIGN — section 7.6.3, for DB_DESIGN — section 7.6.4 instead of the document rules below;
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

### 7.7. Attempt Failure

`INTERRUPTED`, `FAILED` or `INVALID_RESULT`, one step:

- `execution_state` — the corresponding value, `retry.technical_attempts.<stage>` + 1, `updated_at`;
- the `stage.failed` event with `details: { attempt, execution_state, reason }`;
- the attempt file is not registered: it remains a Working Draft;
- if the counter has reached `stage-map.yaml` → `retry_policy.defaults.technical_attempts`: also `story_status: BLOCKED`, `blocker: { reason: "Retry Budget Exhausted", source: ORCHESTRATOR, details, needed: "a human's decision through Recovery" }` and the `story.blocked` event.

### 7.8. Applying an Accepted Result

The route — only from `stage-map.yaml` (`workflow-state.md` §7). One step: first `state.yaml` (section 7.10), then the events in `history.jsonl`. In all cases, `retry.technical_attempts.<stage>` is deleted (a reset after an accepted result, `workflow-state-model.md` §10) and `updated_at` is updated.

**SPEC_REVIEW record.** If `outputs` contain a record, it is registered with any verdict: `records.review-report.SPEC_REVIEW: <record_id>` (`workflow-state.md` §3). The `refs` of the `stage.completed` events contain `<record_id>`.

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

The next `next` will stop at step 7.2, item 3: DESIGN_REVIEW is not supported yet.

**`CHANGES_REQUIRED`** of IMPACT_ANALYSIS, SOLUTION_DESIGN, API_DESIGN or DB_DESIGN with `loopback_target` `SPECIFICATION` or `CLARIFICATION`. These targets have `crosses_gates: [HUMAN_SPEC_APPROVAL]`, so the Loopback is performed only after the human's decision (RR-04, `workflow-state-model.md` §6). The Orchestrator does not perform it, it only requests confirmation (D-1). Below, `<stage>` is `IMPACT_ANALYSIS`, `SOLUTION_DESIGN`, `API_DESIGN` or `DB_DESIGN`:

- for IMPACT_ANALYSIS, API_DESIGN and DB_DESIGN: the record — according to the "applicability record" rule above. SOLUTION_DESIGN does not create records;
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
      - { record: <record_id> }                   # only IMPACT_ANALYSIS, API_DESIGN and DB_DESIGN: the applicability-decision of this attempt
    requested_at: <date -u>
  ```
- events:
  - `stage.completed` with `details: { attempt, verdict: CHANGES_REQUIRED, outputs, loopback_target, crosses_gates: [HUMAN_SPEC_APPROVAL], summary, mode }`, without `to_stage`;
  - `human_decision.requested` (`stage: <stage>`, `refs` — the elements of `package`, `details: { kind: LOOPBACK_CONFIRMATION, options, loopback_target, mode }`).

Then `next` stops at step 7.2 (`story_status` is not `ACTIVE`). The human makes the decision with the `decide` command (section 8.7).

**`CHANGES_REQUIRED`** with a Loopback without `crosses_gates` → `loopback_target`: SPECIFICATION → `CLARIFICATION`; SPEC_REVIEW → `SPECIFICATION` or `CLARIFICATION`; SOLUTION_DESIGN → `IMPACT_ANALYSIS`; API_DESIGN → `SOLUTION_DESIGN` or `IMPACT_ANALYSIS`; DB_DESIGN → `API_DESIGN`, `SOLUTION_DESIGN` or `IMPACT_ANALYSIS`. These loopbacks do not have `crosses_gates`, so the human's decision is not needed (`stage-map.yaml` RR-03, RR-04):

- for API_DESIGN and DB_DESIGN: the record — according to the "applicability record" rule above;
- `last_stage_verdict: CHANGES_REQUIRED`; the Version Status of artifacts does not change;
- `retry.quality_loops.target.<loopback_target>` + 1 and `retry.quality_loops.initiator.<stage>` + 1 (`workflow-state-model.md` §10);
- for SPEC_REVIEW: `retry.repeated_failure.<F-NNN>` + 1 for each `OPEN` `BLOCKING` finding of the report (`workflow-state.md` §3);
- if any counter has reached the corresponding value of `retry_policy.defaults` (`quality_loops`, `repeated_failure`) — `story_status: BLOCKED`, `blocker: { reason: "Retry Budget Exhausted", source: ORCHESTRATOR, details, needed }`, `execution_state: COMPLETED`, `current_stage` does not change, the events `stage.completed` and `story.blocked`;
- otherwise `current_stage: <loopback_target>`, `execution_state: READY`, `story_status: ACTIVE`, the `stage.completed` event with `details: { attempt, verdict, loopback_target, to_stage, summary, mode }`.

Do not start the target stage. The next `next` will perform its re-entry according to section 7.3.1 if it is SPECIFICATION, or according to section 7.3.2 if it is IMPACT_ANALYSIS after a Loopback from SOLUTION_DESIGN. For CLARIFICATION, it will stop (MVP limitation, 7.3.1).

> **MVP limitation (API-D01, DB-D01).** After a Loopback from API_DESIGN (to SOLUTION_DESIGN or IMPACT_ANALYSIS) or from DB_DESIGN (to API_DESIGN, SOLUTION_DESIGN or IMPACT_ANALYSIS), the route and the counters are applied, but re-entry into the target is not implemented: the next `next` stops at section 7.2, step 6, and changes nothing. In the report to the human, do not present the Story as executable by `next`: say that the Story is at `<loopback_target>` without supported re-entry, and that `decide` is not available, because the Story is not waiting for a human decision. The human decides outside the Skill.

Re-entry does not change the counters: Quality Loops have already been accounted for during `CHANGES_REQUIRED`.

**`BLOCKED`** → there is no route (`stage-map.yaml` RR-05):

- for IMPACT_ANALYSIS, API_DESIGN and DB_DESIGN: the record, if it is in `outputs` — according to the "applicability record" rule above;
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

## 8. Command `decide`

Applies the human's decision for the current `pending_human_decision`. Works in the main session, without a subagent. Two kinds of waiting are supported:

| `pending_human_decision.kind` | Stage | Decisions in the MVP | Sections |
|---|---|---|---|
| `GATE` | HUMAN_SPEC_APPROVAL | `APPROVE`, `REQUEST_CHANGES`, `CANCEL` | 8.2–8.5 |
| `LOOPBACK_CONFIRMATION` | IMPACT_ANALYSIS, SOLUTION_DESIGN, API_DESIGN, DB_DESIGN | `RESUME`, `CANCEL`; `ROUTE_BACK` is rejected without changes | 8.2, 8.5, 8.7 |

The human makes the decision, and the Orchestrator only checks and applies it (`workflow-state.md` §1).

The command is non-interactive: the decision, the human's identifier, the target, the reason and the comment come only from the arguments. The Orchestrator does not conduct a dialogue about the content of the artifacts, does not clarify anything and does not propose decisions. A gate approves the existing versions; it does not create new product decisions (D-03).

A decision in the pilot is a simulation of a Human Gate. It applies only within the pilot and is not an approval or cancellation of the Story in the official Workflow.

### 8.1. Arguments

| Argument | Decision | Rule |
|---|---|---|
| `decision` | all | For `GATE`: `APPROVE`, `REQUEST_CHANGES` or `CANCEL` (`gate_decisions` in `stage-map.yaml`); `RECONFIRM` is not offered in the MVP (`docs/contracts/README.md`, "Deferred. An Approved Simplification of the Workflow Is Needed"). For `LOOPBACK_CONFIRMATION`: `ROUTE_BACK`, `RESUME` or `CANCEL` (`loopback_confirmation_decisions`). Any other value is an error |
| `by` | all, mandatory | A non-empty identifier of the human who made the decision. In the pilot, this is a declared identifier, not verified authentication. The Orchestrator does not invent it, does not substitute it for the human and does not take the identifier of an agent or Skill |
| `target` | only `REQUEST_CHANGES`, mandatory | The target stage of the return (section 8.4.1) |
| `reason` | `REQUEST_CHANGES`, `CANCEL` and `RESUME`, mandatory | A non-empty reason. Recorded verbatim according to the rules for the human's text in section 7.10 (`workflow-policies.md` §3, §17). For `RESUME`, the next attempt of the stage receives it verbatim (section 7.3) |
| `comment` | only `APPROVE`, optional | Recorded verbatim according to the rules of section 7.10. It is not a source of requirements (section 8.3.1, step 3) |

If a mandatory argument is missing, there is an argument not allowed for this decision, or the `decision` is not allowed — explain how to repeat the command, and stop. `state.yaml` and `history.jsonl` do not change.

### 8.2. Common Check

For each decision. Check everything in order. The first violation — explain to the human what exactly is wrong and where (file, field), and stop. The state, History and files do not change.

1. **Consistency** of State, History and files — section 6. A discrepancy — as in `status`.
2. **Kind of waiting.** `story_status: WAITING_FOR_HUMAN`, `execution_state: AWAITING_HUMAN`, `pending_human_decision.stage` = `current_stage`. Then — according to `pending_human_decision.kind`:
   - `LOOPBACK_CONFIRMATION` — the check of section 8.7.1 instead of steps 3–4 below;
   - `GATE` — steps 3–4 below;
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

Application is one Orchestrator step (`workflow-state.md` §7): first the record file, if any (only CANCEL), then `state.yaml` according to section 7.10, then the events in `history.jsonl`. `reason`, `comment` and `by` are serialized in all files according to the rules for the human's text (section 7.10). In all decisions, the `actor` of the `human_decision.applied` event is the human from `by` (`{kind: human, id: <by>}`), `stage` is `pending_human_decision.stage` (HUMAN_SPEC_APPROVAL, IMPACT_ANALYSIS, SOLUTION_DESIGN, API_DESIGN or DB_DESIGN), `refs` are the elements of `package`. Artifact and review report files do not change (`artifact-rules.md` §3, item 2).

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

The procedure is the same for both kinds of waiting: `GATE` at HUMAN_SPEC_APPROVAL and `LOOPBACK_CONFIRMATION` at IMPACT_ANALYSIS, SOLUTION_DESIGN, API_DESIGN or DB_DESIGN (`loopback_confirmation_decisions.CANCEL`). The examples below are given for the gate. For `LOOPBACK_CONFIRMATION`:
- the place of the stage in the record and the events is taken by `current_stage` (`IMPACT_ANALYSIS`, `SOLUTION_DESIGN`, `API_DESIGN` or `DB_DESIGN`);
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
- `current_stage` does not change (`HUMAN_SPEC_APPROVAL`, `IMPACT_ANALYSIS`, `SOLUTION_DESIGN`, `API_DESIGN` or `DB_DESIGN`): `CANCELLED` is a terminal state, not a Stage ID (`stage-map.yaml` → `terminal_states`). The consequence of the decision is visible in `story_status` (`workflow-state.md` §4);
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

After 8.3.2, 8.4.2, 8.5.3 or 8.7.3, stop and show the summary with the `status` command. Do not start the next stage.

### 8.7. LOOPBACK_CONFIRMATION (IMPACT_ANALYSIS, SOLUTION_DESIGN, API_DESIGN, DB_DESIGN)

Confirmation of a Loopback through a gate (`stage-map.yaml` RR-04, `loopback_confirmation_decisions`; `workflow-state-model.md` §6). In the MVP, it is created by a `CHANGES_REQUIRED` of IMPACT_ANALYSIS, SOLUTION_DESIGN, API_DESIGN or DB_DESIGN with the target `SPECIFICATION` or `CLARIFICATION` (section 7.8). Below, `<stage>` is `current_stage`. The human makes the decision, the Orchestrator only checks and applies it. A decision in the pilot is a simulation, as at a gate.

#### 8.7.1. Check

Instead of steps 3–4 of section 8.2. The first violation — explain and stop, changing nothing.

1. **State:**
   - `current_stage` is `IMPACT_ANALYSIS`, `SOLUTION_DESIGN`, `API_DESIGN` or `DB_DESIGN`; `pending_human_decision.kind: LOOPBACK_CONFIRMATION`, `pending_human_decision.stage` = `current_stage`;
   - `last_stage_verdict: CHANGES_REQUIRED`;
   - `decision` is in `pending_human_decision.options` and in `loopback_confirmation_decisions` in `stage-map.yaml`.
2. **Package.** `pending_human_decision.package` contains exactly:
   - `{ type: story-spec, version: <current version> }`;
   - only for IMPACT_ANALYSIS, API_DESIGN and DB_DESIGN — `{ record: <record_id> }`, where `<record_id>` is `records.applicability-decision.<stage>.ENTRY`.

   The `refs` of the last `human_decision.requested` event match `package`.
3. **The cause of the Loopback.** The last `stage.completed` event has `stage: <stage>`, `verdict: CHANGES_REQUIRED`, a `loopback_target` from the stage's `loopbacks` with `crosses_gates`, and no `to_stage`. For IMPACT_ANALYSIS, API_DESIGN and DB_DESIGN, its `refs` contain `<record_id>`.
4. **The approval has not changed** (only for `RESUME`; `CANCEL` is available from any non-terminal state, RR-08). `approvals.HUMAN_SPEC_APPROVAL` has `approval_state: VALID`, and the `story-spec` version in its `scope` is the current one (section 7.2, step 6).

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

- `stage` and `to_stage` are `<stage>`. For SOLUTION_DESIGN, `refs` are `package`, that is, only `story-spec@<N>`. For API_DESIGN and DB_DESIGN, `refs` are `story-spec@<N>` and `applicability-decision/<NNNN>`, as for IMPACT_ANALYSIS;
- `rejected_loopback_target` is the `loopback_target` of the event from step 3 of section 8.7.1.

Do not start the stage. The next `next` will execute a new attempt and pass `reason`: for IMPACT_ANALYSIS — according to section 7.3.2, for SOLUTION_DESIGN, API_DESIGN and DB_DESIGN — verbatim according to section 7.3. `version` does not change, because the artifact of this attempt was not registered. For IMPACT_ANALYSIS, API_DESIGN and DB_DESIGN, `record_id` is the next number, and the previous `applicability-decision` will be replaced by the new record after an accepted result (section 7.8).

## 9. Constraints

- The Orchestrator writes only the pilot's `state.yaml` and `history.jsonl`: in `init`, at steps 7.2 (only `RUNNING` → `INTERRUPTED`), 7.4, 7.5.2, 7.7, 7.8, 8.3.2, 8.4.2, 8.5.3 and 8.7.3. The only record it writes itself is `cancellation-record` in 8.5.2. `status` writes nothing.
- An artifact file is written only by the Stage Skill during 7.5. The Orchestrator does not create, edit or delete it.
- Write nothing to `docs/stories/`, `docs/contracts/`, `docs/workflow/`, the Foundation or `.claude/skills/`.
- Do not invent statuses, events, state fields or transition rules. Report to the human what is missing in the contracts.
- History is append-only: lines are not changed or deleted.
- Do not fix the state silently and do not restore it automatically.
- Start only the stages from table 7.1. CLARIFICATION is executed inline in the main session, without a subagent. SPECIFICATION, SPEC_REVIEW, IMPACT_ANALYSIS, SOLUTION_DESIGN, API_DESIGN and DB_DESIGN are each executed in one separate `general-purpose` subagent. Do not pass a historical version of `impact-analysis` after `NOT_APPLICABLE` as an input (section 7.1), do not pass `impact-analysis` to API_DESIGN at all, and pass `impact` and `api` to DB_DESIGN only according to section 7.2, steps 7 and 8. Do not start other stages, in particular DESIGN_REVIEW, and additional subagents.
- Apply the human's decisions only in `decide` (section 8), by the human's decision from the command arguments. Do not simulate or guess the human's decisions, do not substitute `by`. Do not apply decisions of other gates, RECONFIRM and `ROUTE_BACK`. Do not perform a Loopback through a gate automatically.
- A decision in the pilot is a simulation of a Human Gate. Do not present it as an approval or cancellation of the Story in the official Workflow.
- Do not invoke Stage Skills with the Skill tool.
- Change `state.yaml` only according to section 7.10.
- Do not run `git add`, `git commit`, `git push` or destructive operations.
