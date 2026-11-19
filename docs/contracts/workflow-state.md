# Workflow State

**Status:** `APPROVED (MVP)`, Simplified Artifact Contracts MVP. Approved on 2026-11-17 as the design of the MVP contracts; boundaries of the approval — [README.md](README.md).

This document defines the format of the Story state (`state.yaml`), the event log (`history.jsonl`) and the stage result envelope. The status values are taken from the approved Workflow Definition: `stage-map.yaml` and `workflow-state-model.md`. The document does not introduce new statuses.

## 1. Who Writes

- `state.yaml` and `history.jsonl` are changed **only by the Orchestrator**.
- A Stage Skill writes the files of its artifacts (in the draft status, [artifact-rules.md](artifact-rules.md), section 3) and returns a result envelope (section 6). The Skill does not change the state.
- Human decisions are recorded by the Orchestrator: in `approvals` or `pending_human_decision` → History.
- Chat history is not a source of state.

## 2. Files

```
docs/stories/<story_id>/
├── state.yaml        # the current state, mutable
├── history.jsonl     # the event log, append-only
├── artifacts/…       # document versions
└── records/…         # supporting records
```

## 3. `state.yaml`

Example: the Story is waiting for HUMAN_SPEC_APPROVAL.

```yaml
story_id: US-001
story_type: FEATURE                 # from story-spec after its registration; null before that
story_status: WAITING_FOR_HUMAN     # ACTIVE | WAITING_FOR_HUMAN | BLOCKED | DONE | CANCELLED
current_stage: HUMAN_SPEC_APPROVAL  # Stage ID from stage-map.yaml
execution_state: AWAITING_HUMAN     # READY | RUNNING | AWAITING_HUMAN | COMPLETED | INTERRUPTED | FAILED | INVALID_RESULT
last_stage_verdict: PASS            # the verdict of the last completed attempt of the current/previous stage
execution_mode: STEP                # STEP | AUTO
updated_at: 2026-11-18T11:20:00Z

artifacts:                          # only the current version of each type
  clarification-record: { version: 1, status: REVIEWED, currency: CURRENT }
  story-spec:           { version: 2, status: REVIEWED, currency: CURRENT }

records:                            # the last valid record by purpose
  review-report:
    SPEC_REVIEW: review-report/0002
  applicability-decision: {}        # <STAGE>: { PLANNED: <record_id>, ENTRY: <record_id> }
  revalidation-record: []           # all valid revalidation records

approvals:
  HUMAN_SPEC_APPROVAL: null         # see section 4

pending_human_decision:
  kind: GATE                        # GATE | HUMAN_ACTION | LOOPBACK_CONFIRMATION | RECOVERY | CLARIFICATION_ANSWERS
  stage: HUMAN_SPEC_APPROVAL
  options: [APPROVE, REQUEST_CHANGES, CANCEL]
  package:                          # what the human must review: the Approval Scope and the reports
    - { type: clarification-record, version: 1 }
    - { type: story-spec, version: 2 }
    - { record: review-report/0002 }
  requested_at: 2026-11-18T11:20:00Z

blocker: null                       # { reason, source: STAGE | ORCHESTRATOR, details, needed }

retry:
  technical_attempts: {}            # <STAGE>: n — reset after an accepted result
  quality_loops:                    # separately for the target and the initiator; reset only by a full APPROVE of the corresponding gate
    target: { SPECIFICATION: 1 }
    initiator: { SPEC_REVIEW: 1 }
  repeated_failure: { F-002: 1 }    # occurrences of a Finding ID within the Story; not reset
  extensions: []                    # { counter, key, extra, decided_by, decided_at }

git:
  feature_branch: story/US-001
  target_branch: main
  changeset_commit: null            # from the current version of implementation-changeset
  verified_target_base: null        # from the current verification-report
  merge_protection_status: null     # ACTIVE | NOT_SUPPORTED | UNAVAILABLE | DISABLED
```

### Artifact Statuses

- `status` — the Version Status of the current version: `DRAFT`, `REVIEWED` or `APPROVED`.
- Each version with a number lower than the current one is `SUPERSEDED`. Such versions are not listed separately.
- `currency` — `CURRENT` or `STALE`, computed according to the rule of [artifact-rules.md](artifact-rules.md), section 4.
- The status and Currency are stored **only here**. They are not in the artifact files, so they cannot diverge.
- Version Status does not go backward. `REVIEWED` is set when a review stage has returned `PASS` for this version. `APPROVED` — when a human has made APPROVE at a gate whose Scope includes the version.

## 4. Approvals

A record is created or replaced when a human makes a decision at a gate.

**Approval Scope.** `scope` contains **each** corresponding element of this gate's `approval_scope` from `stage-map.yaml`, expanded to the exact identity:

- a controlled document — `{ type, version }`;
- a supporting record — `{ record: <record_id> }`;
- `implementation-changeset` — `{ type, version, changeset_commit }`.

An element with the condition "if the stage is applicable" is replaced, for a non-applicable stage, by a record `{ record: <applicability-decision> }` with `NOT_APPLICABLE`. An element of the Scope must not be omitted silently. Reports that the human reviewed but that are not part of the Scope (for example, the SPEC_REVIEW `review-report` for HUMAN_SPEC_APPROVAL) are stored separately in `package`.

```yaml
approvals:
  HUMAN_SPEC_APPROVAL:
    decision: APPROVE
    decided_by: human-architect            # who: the human's identifier
    decided_at: 2026-11-18T12:05:00Z      # when
    scope:                                # what exactly is approved: all elements of the gate's approval_scope
      - { type: clarification-record, version: 1 }
      - { type: story-spec, version: 2 }
    package: [review-report/0002]         # reviewed, but not part of the Scope
    comment: "OD-01: варіант A"
    approval_state: VALID                 # VALID | SUSPENDED | INVALIDATED
    previous: []                          # previous approvals of this gate that became INVALIDATED

  HUMAN_PR_APPROVAL:                      # an example of a Scope with records
    scope:
      - { type: implementation-changeset, version: 3, changeset_commit: 4f2a9c1 }
      - { record: review-report/0007 }            # CODE_REVIEW
      - { record: applicability-decision/0009 }   # SECURITY_REVIEW ENTRY: NOT_APPLICABLE
      - { record: applicability-decision/0010 }   # RECONCILIATION ENTRY: NOT_APPLICABLE
      - { record: verification-report/0002 }
      - { record: pull-request/0001 }
```

**Replacement of a record.** A record in the Scope is considered replaced when a new record for the same purpose becomes valid. For example, a new `verification-report` after a repeated VERIFICATION or a new CODE_REVIEW `review-report` for a new changeset version (the reference in `state.yaml` → `records` changes). A `revalidation-record` does not replace a record: it only confirms that the conclusion is still valid.

Computation of `approval_state` (`workflow-state-model.md`, section 7) — only in one direction:

| Transition | Who and when |
|---|---|
| → `VALID` | **Only** a human's APPROVE decision. It creates a new approval record, and the previous one moves to `previous` |
| `VALID` → `SUSPENDED` | The Orchestrator: the Scope identities have not changed, but after `decided_at` the upstream of at least one element has changed. The document became `STALE`; for a record — the version from its `subjects` is no longer current, and there is no new record yet |
| `VALID` / `SUSPENDED` → `INVALIDATED` | The Orchestrator: at least one element of the Scope has received a newer version, a new `changeset_commit` or has been replaced by a new record |

Exception from the Workflow: `post-merge-validation-record` is not part of any Scope and does not change HUMAN_PR_APPROVAL (`stage-map.yaml`, `approval_effect`).

The Orchestrator never returns an approval to `VALID`. In particular, a `revalidation-record` returns Currency `CURRENT` to the artifact, but the approval remains `SUSPENDED` (`workflow-state-model.md`, section 7).

**MVP limitation.** RECONFIRM is not supported. An approval in the `SUSPENDED` or `INVALIDATED` state becomes valid again only through a new full APPROVE. This is stricter than the Workflow Definition (see README.md).

Human decisions that are not an approval (REQUEST_CHANGES, CANCEL, CONFIRM_MERGED, Recovery decisions and Loopback confirmation) are recorded only in History (section 5). The consequence of each of them is visible in the state: `current_stage`, `story_status`, `blocker`.

## 5. `history.jsonl`

One line — one JSON event. The file is append-only: lines are not changed or deleted.

```json
{"ts":"2026-11-18T12:05:00Z","story_id":"US-001","actor":{"kind":"human","id":"human-architect"},"event":"human_decision.applied","stage":"HUMAN_SPEC_APPROVAL","refs":["clarification-record@1","story-spec@2","review-report/0002"],"details":{"decision":"APPROVE","to_stage":"IMPACT_ANALYSIS","comment":"OD-01: варіант A"}}
```

| Field | Meaning |
|---|---|
| `ts` | The event time, ISO-8601 UTC |
| `actor` | `{kind: agent \| human \| orchestrator, id}`. For an agent — the stage or Skill name |
| `event` | The event type from the table below |
| `stage` | The stage to which the event belongs |
| `refs` | References: `<type>@<version>` for documents, `<record_id>` for records |
| `details` | Brief event data: verdict, decision, route, reason |

| `event` | When |
|---|---|
| `story.registered` | The Story is registered, the state is initialized from `initial_state` |
| `stage.started` | A stage attempt has started |
| `stage.completed` | The Orchestrator accepted the result envelope: verdict, registered outputs, route |
| `stage.failed` | The attempt is `INTERRUPTED`, `FAILED` or `INVALID_RESULT`, with a reason |
| `human_decision.requested` | The Story is waiting for a human (`pending_human_decision`) |
| `human_decision.applied` | A human's decision: who, what, when, regarding which versions, route |
| `staleness.detected` | An upstream change outside a stage transition, in particular of the Foundation |
| `story.blocked` | Story → `BLOCKED`: reason and source |
| `retry.extended` | A human extended the Retry Budget |
| `story.done` / `story.cancelled` | A terminal state |

There is no separate event for a mode change: `execution_mode` changes in the `details` of any event initiated by a human.

## 6. Stage Result Envelope

A Stage Skill completes its work by returning the following block to the Orchestrator:

```yaml
result:
  story_id: US-001
  stage: SPEC_REVIEW
  attempt: 1
  verdict: CHANGES_REQUIRED          # PASS | CHANGES_REQUIRED | BLOCKED | NOT_APPLICABLE
  outputs:                           # the files the stage wrote
    - { record: review-report/0003 }
  loopback_target: SPECIFICATION     # only for CHANGES_REQUIRED; a target from the stage's loopbacks
  blocker: null                      # for BLOCKED: { reason, details, needed }
  summary: "1 BLOCKING finding (F-002)"
```

The Orchestrator accepts the result only when everything in the list is met:

- the verdict is allowed for the stage (`NOT_APPLICABLE` — only for `optional: true` together with an `applicability-decision`);
- all mandatory outputs from [stage-io-map.yaml](stage-io-map.yaml) exist and have correct metadata;
- `loopback_target` is in the stage's `loopbacks`.

Otherwise, `execution_state` becomes `INVALID_RESULT` and a `stage.failed` event is emitted.

## 7. How the Orchestrator Records a Transition

1. Check the result envelope or the human's decision (section 6).
2. Determine the route **only** from `stage-map.yaml`.
3. Register new versions and records in `state.yaml`, update `status`, `currency` and `approval_state`.
4. Update the stage fields, `pending_human_decision`, `blocker`, `retry`.
5. Append one or more events to `history.jsonl`.

A History line and `state.yaml` are changed in one Orchestrator step. There is no durable transaction protocol and no replay. If, during session recovery, `state.yaml`, History and the files contradict each other, the Story moves to `BLOCKED` (Integrity Error), and the discrepancy is resolved by a human. There is no automatic correction.

## 8. Checking the Link Between Review and Code Changes

For stages with an order greater than CODE_REVIEW (RR-10), the Orchestrator checks that all conditions are met:

1. the CODE_REVIEW `review-report` (as well as SECURITY_REVIEW and `reconciliation-report`, if they were applied) has the current version of `implementation-changeset` in `subjects`;
2. after `changeset_commit`, only Story files have changed in the feature branch:

   ```
   git diff --quiet <changeset_commit> HEAD -- . ':(exclude)docs/stories/<story_id>'
   ```

3. outside `docs/stories/<story_id>/` there are no uncommitted or untracked changes (`git status --porcelain` is empty for these paths).

The same rule applies to the PR head in MERGE_READINESS_CHECK: between `changeset_commit` and the PR head, only Story files have changed.

If at least one condition is not met, the Story moves to `BLOCKED` (Integrity Error). A rebase or squash changes `changeset_commit`, so it requires a new version of the changeset through IMPLEMENTATION. This is stricter than the Workflow Definition (see README.md).
