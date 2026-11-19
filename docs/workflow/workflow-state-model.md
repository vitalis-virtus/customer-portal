# Workflow State Model — Workflow Definition

**Document status:** `APPROVED` (revision 3). Approved by a human on 2026-11-16 after the final Review.

## 1. Purpose

This document defines the logical model of the Story Workflow state: what the state consists of, how it changes, how Recovery, Retry, approval validity, Staleness, History and recovery after an interrupted session work.

Transition routes are defined by [stage-map.yaml](stage-map.yaml). Stage descriptions — [stages.md](stages.md). Policies — [workflow-policies.md](workflow-policies.md).

### Out of Scope

- Technical schemas of the current state file (YAML) and the history file (JSONL): fields, types, paths, event format. They are defined by the Artifact Contracts stage.
- The mechanism for computing artifact versions and checking dependencies. A software Dependency Engine is not created at this stage.
- The Orchestrator implementation.

The document defines **what** must be stored and **which** rules apply. How exactly it is stored is defined by the next stages.

## 2. Components of the Workflow State

Each Story has its own independent Workflow State (Q12).

| Component | Meaning |
|---|---|
| Story Status | The overall state of the Story (section 4) |
| Story Type | The Story type from `story_types` in `stage-map.yaml` (OD-14). Recorded in `story-spec` |
| Current Stage | The stage from `stage-map.yaml` at which the Story is |
| Execution State | The state of the current attempt to execute the stage (section 3) |
| Last Stage Verdict | The last verdict of the current stage, if the attempt is completed |
| Pending Human Decision | The decision the Story is waiting for: a gate, a human action, Recovery, a Loopback through a gate, an escalation, answers to questions |
| Artifact Registry | The Story's controlled artifacts, their versions, Version Status, Currency and recorded dependencies (section 5) |
| Evidence Records | The Story's supporting records: `review-report`, `revalidation-record`, `applicability-decision` and others (section 5) |
| Approvals | Human Gate decisions bound to the Approved Content Identity of the Scope artifacts, and their Approval State (section 7) |
| Optional Stage Applicability | Valid `applicability-decision` records for Optional Stages |
| Retry Counters | Retry Policy counters (section 10) |
| Blocker | The reason for `BLOCKED`, if the Story is blocked (section 9) |
| Execution Mode | Step Mode or Auto Mode of the current run ([workflow-policies.md](workflow-policies.md#11-step-mode-and-auto-mode)) |
| Git Binding | The Story's feature branch, Changeset Content Identity and Target Base Identity to which checks and approval are bound (OD-12) |

**Sources of truth (Q9, Q18):**

- The current state is stored in the Story's state file in YAML format. This is the canonical source of the current state.
- The history of events and transitions is stored append-only in JSONL format. This is the canonical source of what happened.
- Approved human decisions are recorded in canonical artifacts (for example, in `clarification-record`, an approval in the state). A record only in History is insufficient.
- Chat history and Claude Code session memory are not a source of state, decisions or requirements.

**Who changes the state.** The Workflow State and History are changed only by the Orchestrator. A Stage Skill returns a result (verdict, outputs, a Loopback proposal, a justification), but does not change the state directly.

## 3. Execution State

The Execution State describes the technical course of one attempt to execute a stage. It is not an assessment of the result (Q10).

| Execution State | Meaning |
|---|---|
| `READY` | The stage is current, the attempt has not yet started |
| `RUNNING` | The attempt has started, there is no result yet |
| `AWAITING_HUMAN` | The stage or the Orchestrator is waiting for a human's decision or answer |
| `COMPLETED` | The attempt returned an accepted result with a Stage Verdict |
| `INTERRUPTED` | The attempt did not return a result: the session was interrupted, the process was stopped |
| `FAILED` | A technical failure of a tool or the environment during the attempt |
| `INVALID_RESULT` | A result was returned, but the Orchestrator rejected it: mandatory outputs are missing, the verdict is not allowed for the stage, the Loopback target is not allowed |

Rules:

- `INTERRUPTED`, `FAILED` and `INVALID_RESULT` are not the `BLOCKED` verdict. They are handled as technical attempts within the Retry Policy (section 10).
- For `human_gate` and `human_action`, the Execution State is `AWAITING_HUMAN` until the human makes a decision.
- Waiting for the human's answers during CLARIFICATION is `AWAITING_HUMAN`, not `BLOCKED`.

## 4. Stage Verdict and Story Status

### Stage Verdict

A Stage Verdict is an assessment of the stage result. It is returned by an agent stage after a completed attempt (Q10).

| Verdict | Meaning | Mandatory accompaniment |
|---|---|---|
| `PASS` | The stage fulfilled its responsibility, the mandatory outputs are created | The stage outputs |
| `CHANGES_REQUIRED` | Changes are needed in an artifact of this stage or of an upstream stage | Findings and the proposed Loopback target |
| `BLOCKED` | The stage cannot continue without a human's decision or action | The reason for blocking and what is needed for recovery |
| `NOT_APPLICABLE` | The Optional Stage is not applicable for this Story | A justification according to the applicability condition |

Rules:

- `NOT_APPLICABLE` is allowed only for stages with `optional: true` in `stage-map.yaml`.
- A human's decision at a Human Gate is a **Gate Decision**, not a Stage Verdict. A Reviewer `PASS` does not equal a Human Approval.
- A producer stage returns `CHANGES_REQUIRED` when it has found a problem in an upstream artifact. A review stage — when it has found BLOCKING findings in the checked artifact or higher.

### Story Status

| Story Status | Meaning |
|---|---|
| `ACTIVE` | The Story can be executed without a human's decision |
| `WAITING_FOR_HUMAN` | The Story is waiting for a Gate Decision, a human action, the confirmation of a Loopback through a gate or the human's answers |
| `BLOCKED` | The Story is blocked. It can be exited only through the Recovery Protocol (section 9) |
| `DONE` | Terminal. The Story is completed according to the Definition of Done |
| `CANCELLED` | Terminal. The Story is cancelled by a human's decision |

The Story moves to `BLOCKED` in the following cases:

- a stage returned the `BLOCKED` verdict;
- the Retry Budget is exhausted (section 10);
- the Orchestrator detected a state integrity violation (section 12) or the Review Binding Guard fired (`stage-map.yaml` RR-10);
- an escalation or a Foundation Change Request is needed ([workflow-policies.md](workflow-policies.md));
- INTEGRATION_CONFIRMATION detected an Integration Mismatch or Post-Merge Validation did not pass.

For each blocking, the **source** is recorded: a Stage Verdict or the Orchestrator. This way a stage blocking is not confused with a technical or process stop.

## 5. Artifact Status

### Artifact Classes (Q11)

- **Controlled artifacts** go through approval or can become stale. They have versions, Version Status, Currency and recorded dependencies on the versions of upstream artifacts. The list is in [stages.md](stages.md#5-logical-artifacts).
- **Supporting artifacts** (`review-report`, `revalidation-record`, `applicability-decision`, `reconciliation-report`, `verification-report`, `pull-request`, `integration-record`, `post-merge-validation-record`, `risk-acceptance-record`, `foundation-exception-record`, `cancellation-record`) do not have their own Lifecycle. They are **immutable records**: after creation, a record is not edited, and a correction creates a new record. Each record is bound to the versions of the artifacts it checked or described, and to the versions of the upstream artifacts against which the check was performed. A record regarding one version does not count for another.

The Product Foundation, Architecture Foundation and Workflow Definition are upstream sources for all Stories. Their versions are taken into account in the dependencies of controlled artifacts, but they change only through their own approval processes, not through the Story Workflow.

### Approved Content Identity (WF-03)

**Invariant:** every artifact covered by a Human Approval has a stable identity of the approved content — the **Approved Content Identity**. It uniquely determines the content the human saw and makes it possible to later verify that the content has not changed, in particular during Git Integration.

- For a controlled artifact, the Approved Content Identity is its version.
- For a supporting record — the identity of the immutable record.
- For `implementation-changeset` — the **Changeset Content Identity**: the identity of the content of the Story's changes, independent of the Git SHA of individual commits. Separately, the **Target Base Identity** is recorded — the version of the target branch against which the check was performed (OD-12).
- For `pull-request` — the repository, the PR identifier, the source and target branch and the Changeset Content Identity that the PR contains. PR metadata (title, description, labels) is not part of it (`stage-map.yaml` RR-12).

The enforcement mechanism (immutable snapshot, content hash or another) is defined by the Artifact Contracts stage. The Workflow only requires that the identity be stable and verifiable.

### Version Status

Version Status describes facts about a specific immutable version of a controlled artifact.

| Version Status | Meaning |
|---|---|
| `DRAFT` | The version is created, there is no review for it yet or the review did not pass |
| `REVIEWED` | The corresponding review stage returned `PASS` for this version |
| `APPROVED` | A human approved this version (APPROVE) at a Human Gate |
| `SUPERSEDED` | A newer version of this artifact exists |

### Currency

Currency is a separate dimension of the freshness of a version relative to upstream artifacts. It does not change the Version Status.

| Currency | Meaning |
|---|---|
| `CURRENT` | Each upstream dependency of the version either matches the current version of the upstream or is covered by a `revalidation-record` for the current version of the upstream |
| `STALE` | At least one upstream dependency has changed, and there is no `revalidation-record` for the new version of the upstream |

### Versioning Rules (WF-01)

- **Version immutability.** The content of a version and the dependencies recorded at its creation never change. Any change of content creates a new version with the `DRAFT` status, and the previous one becomes `SUPERSEDED`.
- **Revalidation does not change the version.** Confirmation of the freshness of an unchanged version relative to new upstream versions is recorded in a separate `revalidation-record`. The historical content and dependencies of the approved version are not rewritten.
- **Version Status and Currency are independent.** A version can be `APPROVED` and `STALE` at the same time. This means: the human approved exactly this content, but its freshness relative to the changed upstream has not yet been confirmed.
- **`SUPERSEDED` takes precedence.** For a `SUPERSEDED` version, Currency is not evaluated. It cannot be used as an input or as a basis for an approval.
- **Version Status does not go backward.** There are no transitions `APPROVED` → `REVIEWED` or `REVIEWED` → `DRAFT`. The validity of an approval is determined by the Approval State (section 7), not by the Version Status.
- Artifacts without their own gate (`implementation-changeset` before HUMAN_PR_APPROVAL, `clarification-record` before HUMAN_SPEC_APPROVAL) receive `APPROVED` through the gate whose Scope they are part of.
- A change of an artifact outside the Workflow (manual editing, a change in Git without the Orchestrator's participation) is detected as a new version. Then the Rework Protocol applies (section 14).

### Use of `STALE` Artifacts

- Regular execution of a stage does not use `STALE` or `SUPERSEDED` versions as valid inputs.
- The Revalidation procedure (section 8) and the Change Impact Assessment may read a `STALE` version, its previous upstream versions and new upstream versions to assess the impact. The result of such reading is only a `revalidation-record` or a new version, but not other outputs of the stage.

## 6. Transitions

A transition is performed by the Orchestrator according to the following procedure:

1. **Obtain the result** of the stage or the human's decision.
2. **Check the result:** the mandatory outputs are present; the verdict is allowed for the type and optionality of the stage; for `NOT_APPLICABLE` there is a justification; for `CHANGES_REQUIRED` the Loopback target is in the `loopbacks` of the current stage.
3. **Determine the route** only from `stage-map.yaml`. The Stage Skill proposes the Loopback target, but does not determine the route (Q13).
4. **Check the preconditions** of the target stage: the input artifacts have Currency `CURRENT` and are not `SUPERSEDED`; the required approvals are in the `VALID` state; for stages after CODE_REVIEW, the Review Binding Guard is performed (`stage-map.yaml` RR-10); for routes with a `condition`, the condition is confirmed.
5. **Apply the transition:** update the Workflow State and record an event in History. If the transition is a return to a previous stage, the Rework Protocol applies (section 14).

Rules:

- A disallowed transition is not performed. The Orchestrator records a rejection event, and the Execution State becomes `INVALID_RESULT`.
- A failed Review Binding Guard is not an `INVALID_RESULT` of the stage: the Story moves to `BLOCKED` (Integrity Error), because the content of the changes does not correspond to the passed reviews.
- A Loopback with `crosses_gates` first moves the Story to `WAITING_FOR_HUMAN`. The transition is performed only after the human's decision, because it changes an approved artifact. The human chooses one of the `loopback_confirmation_decisions` in `stage-map.yaml`: `ROUTE_BACK` (the proposed or another allowed target), `RESUME` (the Loopback is rejected, a new attempt of the current stage taking the decision into account) or `CANCEL`. The decision is recorded in History.
- A new Story starts from `initial_state` in `stage-map.yaml`.
- An `applicability-decision` is bound to the **Applicability Basis** — all relevant facts on which the decision was based (WF-V2-02): artifact versions, Changeset Content Identity, Workflow State facts, relevant History events, Retry Counters, human decisions, the Story Risk Assessment. Which categories of facts are relevant for each Optional Stage is defined by `basis_facts` in `stage-map.yaml`. Before using a decision (on entry to the stage, in the `na_checked_by` check, in the Review Binding Guard, before a Human Gate), it is checked that no relevant fact has changed (AP-08). If one has changed, the decision is re-evaluated, and the previous `NOT_APPLICABLE` is not carried over automatically. Events outside `basis_facts` do not invalidate the decision (AP-09).
- After a return, the Story goes through all subsequent stages in order. Each stage on re-entry performs Revalidation (section 8): it either creates a `revalidation-record` for an unchanged version or creates a new version.
- Review stages on re-entry check new versions of artifacts and the `revalidation-record` for unchanged versions. A report regarding a previous version or regarding other upstream versions does not count.

## 7. Approval Validity

A Human Approval is bound to the Approved Content Identity of each artifact of the **Approval Scope** of the corresponding gate (`approval_scope` in `stage-map.yaml`) (Q16, WF-03).

### Approval State

The Approval State is computed relative to the **Approval Basis** — the upstream versions valid at the moment of the human's last decision regarding this approval (APPROVE or RECONFIRM). It is not derived from Currency: a `revalidation-record` makes an artifact `CURRENT`, but does not by itself restore the approval.

| Approval State | Condition | Consequence |
|---|---|---|
| `VALID` | Each artifact of the Scope has the approved Approved Content Identity, and the upstream has not changed since the Approval Basis | The approval can be relied on for transitions |
| `SUSPENDED` | The identity has not changed, but at least one upstream has changed since the Approval Basis | The approval cannot be relied on until the human performs RECONFIRM |
| `INVALIDATED` | At least one artifact of the Scope has received a new version or a new record instead of the approved one, or Revalidation has shown that a new version is needed | The approval is no longer valid. A full APPROVE of the new versions is needed |

Rules:

- Moving from `SUSPENDED` to `VALID` is possible only through the human's RECONFIRM. After it, the Approval Basis is the current upstream versions, and this is recorded by the RECONFIRM event.

- `INVALIDATED` does not return to `VALID`. A new human decision creates a new approval.
- An approval never becomes valid for changed content without the human's APPROVE decision.
- The human's decision and its binding to the identity are recorded in the Workflow State and History.

### Reconfirmation (OD-06)

A Human Gate is a mandatory stop point during every pass. During a repeated pass of a gate, the human chooses one of two decisions:

- **APPROVE** — full approval. Mandatory if at least one artifact of the Scope has a new identity or the approval is in the `INVALIDATED` state.
- **RECONFIRM** — a shortened re-confirmation. Available only when, simultaneously:
  - there is a previous approval in the `VALID` or `SUSPENDED` state for the gate;
  - each artifact of the Scope has the same Approved Content Identity as at the previous APPROVE;
  - each upstream change after the Approval Basis is covered by a `revalidation-record` for each affected artifact of the Scope, and a review stage has checked these records.

RECONFIRM is recorded as a separate event. It references the original approval and the `revalidation-record` items the human accepted. The original approval is not replaced, and RECONFIRM is not considered an approval of new content. The decision package explicitly shows that the content has not changed and which upstream changes have been checked.

Auto Mode performs neither APPROVE nor RECONFIRM.

**Approvals of previous gates that the Story will not pass again.** If the upstream changed when the Story had already passed a gate (for example, a Foundation Change Request during IMPLEMENTATION), the approval of this gate may become `SUSPENDED`. Then:

- the required `revalidation-record` items are created by Out-of-Stage Revalidation (section 8), without returning the Story to previous stages;
- the human makes the RECONFIRM for such an approval within a Recovery or Foundation Change Request decision, separately for each approval, with the same conditions;
- `RESUME` is impossible as long as at least one approval of previous gates is in the `SUSPENDED` state without RECONFIRM or in the `INVALIDATED` state. For `INVALIDATED`, a `ROUTE_BACK` to the producer stage of the corresponding artifact is needed.

**The target branch after the merge (OD-16).** After CONFIRM_MERGED, changes to the target branch, in particular a difference of the Target Base Identity at the moment of the merge from the checked one, are handled only by Post-Merge Validation in INTEGRATION_CONFIRMATION. They do not change the Currency of `verification-report` and the Approval State of HUMAN_PR_APPROVAL. `post-merge-validation-record` is not part of any Approval Scope and does not replace `verification-report`. HUMAN_PR_APPROVAL remains valid as long as the Changeset Content Identity has not changed. Before the merge, a change of the target branch is handled as usual: MERGE_READINESS_CHECK → a repeated VERIFICATION → a new `verification-report` → `INVALIDATED`.

## 8. Staleness and Revalidation

### Dependencies

Each version of a controlled artifact records the versions of the upstream artifacts on which it is based (Q20). For example, `solution-design` depends on a version of `story-spec`, and `implementation-plan` — on versions of the design artifacts. The specific list of dependencies of each type and the way they are recorded is defined by the Artifact Contracts stage.

Dependencies are recorded when a version is created and do not change afterward (section 5). Supporting records also record the upstream versions against which the check was performed.

### Staleness

When an upstream artifact receives a new version, each dependent version that references the previous version of the upstream and does not have a `revalidation-record` for the new one receives Currency `STALE`. The Version Status does not change. Approvals whose Scope includes a `STALE` artifact move to `SUSPENDED`. This is a "check needed" marker, not an automatic reset of all approvals.

A review report performed against upstream versions other than the current ones does not count as `PASS` for further transitions.

### Change Impact Assessment and Revalidation

**Change Impact Assessment** is an assessment of how a change of an upstream artifact affects dependent artifacts. It is a procedure, not a stage. It should not be confused with the IMPACT_ANALYSIS stage, which assesses the Story's impact on the existing system.

- The Assessment is performed by the stage that changed the upstream artifact, or by the Orchestrator within a Foundation Change Request.
- The result of the Assessment is part of the package of the nearest human decision (gate, Recovery, Foundation Change Request). Therefore, the human sees which approved artifacts the change has affected.

**Revalidation** is performed by the owner stage of the dependent artifact during re-entry. It may read a `STALE` version (section 5).

| Revalidation result | Consequence |
|---|---|
| The change does not affect the artifact | A `revalidation-record` is created: version X remains current relative to upstream versions Y′, with a justification. The content, dependencies and Version Status of version X do not change. Currency becomes `CURRENT`. The Approval State does not change: a `SUSPENDED` approval becomes `VALID` only after the human's RECONFIRM (section 7) |
| The change affects the artifact | The stage creates a new version (`DRAFT`). The previous one becomes `SUPERSEDED`. Approvals whose Scope includes the previous version become `INVALIDATED` |

A `revalidation-record` is an immutable supporting record. It is checked by the artifact's review stage on a repeated pass. Finally, the human sees it during APPROVE or RECONFIRM (section 7).

**Supporting records in the Approval Scope** (`review-report`, `applicability-decision`, `verification-report` and others) are subject to the same rule. If the conclusion of an immutable record remains valid relative to new upstream versions, the author of the record (the stage that created it) records this in a `revalidation-record` rather than creating a new record. A new record is created only when the conclusion has changed, and then the approval whose Scope included the previous record becomes `INVALIDATED`.

### Out-of-Stage Revalidation

When the upstream has changed and the Story is at a stage after the artifact that needs to be checked (for example, a Foundation Change Request during IMPLEMENTATION), Revalidation can be performed without returning the Story:

1. The artifact's owner role performs Revalidation and creates a `revalidation-record` or reports that a new version is needed.
2. The review role of this artifact checks the `revalidation-record`.
3. The Current Stage does not change. The event is recorded in History.

If a new version is needed, Out-of-Stage Revalidation ends, and the human chooses `ROUTE_BACK` to the producer stage of this artifact. Out-of-Stage Revalidation is started only within a human's decision regarding Recovery or a Foundation Change Request.

## 9. BLOCKED and Recovery Protocol

Each `BLOCKED` has a defined recovery path (Q14).

### Blocking Reasons

| Reason | Example |
|---|---|
| Missing Information | Information that only a human can provide is needed, and the stage cannot continue |
| Escalation Required | A decision from the list of Controlled Autonomy escalations ([workflow-policies.md](workflow-policies.md#2-controlled-autonomy)) |
| Foundation Change Required | The Story requires a change to the Product or Architecture Foundation |
| External Dependency | An external system, access or permission is unavailable |
| Retry Budget Exhausted | The limit of attempts or Loopbacks is exhausted (section 10) |
| Upstream Changed | During Session Recovery or between runs, the upstream (Foundation, an upstream artifact) changed, and Revalidation is needed before the Story continues |
| Integrity Error | The state, History or artifacts contradict each other (section 12), or the Review Binding Guard is not satisfied (`stage-map.yaml` RR-10) |
| Integration Mismatch | The merge introduced into the target branch changes other than those approved by the human, or unapproved content together with them |
| Post-Merge Validation Failed | The Target Base Identity changed before the merge, and the check of the actual integrated result did not pass (OD-16). Diagnostics — in `post-merge-validation-record` |
| Foundation Rule Violation | A mandatory Foundation rule is violated that cannot be eliminated through Risk Acceptance (WF-V2-01). Resolution — a fix, a Foundation Change Request or an allowed exception |
| Merge Protection Unavailable | The protection mechanism is supported but objectively unavailable (`merge_protection_statuses.UNAVAILABLE`, FR-02). The human in Recovery approves an acceptable integration order |
| Merge Protection Disabled | The mechanism is available and mandatory under the Merge Protection Policy, but disabled (`merge_protection_statuses.DISABLED`, FR-02). `RESUME` only after protection is enabled or under a valid `foundation-exception-record` |

The list of reasons is a working one. The final names are defined by the Artifact Contracts stage.

### Recovery Actions

| Action | Consequence |
|---|---|
| `RESUME` | The Story returns to `ACTIVE` at the same stage and starts a new attempt. The human may at the same time allow additional attempts beyond the Retry Budget and perform RECONFIRM for `SUSPENDED` approvals (section 7) |
| `ROUTE_BACK` | The Story moves to a target from `recovery_route_back_targets` in `stage-map.yaml`. The human's decision is at the same time a confirmation of the return through gates. Then — the Rework Protocol |
| `CANCEL` | The Story moves to `CANCELLED` through the Cancellation Protocol ([workflow-policies.md](workflow-policies.md#17-cancellation-protocol-od-13)) |
| `KEEP_BLOCKED` | The Story remains `BLOCKED`. The decision and the reason are recorded |

**Integration Mismatch and Post-Merge Validation Failed.** The agent does not perform an automatic fix of an already performed merge: no merge, no revert, no force-push, no new commits in the target branch. The human chooses:

| Action | When |
|---|---|
| `RESUME` | The human has fixed the target branch outside the Story (themselves or through another Story). INTEGRATION_CONFIRMATION repeats the checks |
| `KEEP_BLOCKED` | The correction is still in progress, for example through a new related BUG Story |
| `CANCEL` | `cancellation-record` records that the changes have already been integrated and the check has not passed |

`ROUTE_BACK` is possible only to HUMAN_MERGE (`recovery_route_back_targets`).

Integration Match always evaluates the merge result recorded by the last CONFIRM_MERGED, not the current state of the target branch. Post-Merge Validation checks the current state of the target branch. `RESUME` repeats the checks for the same merge. A repeated merge after a correction goes through `ROUTE_BACK` → HUMAN_MERGE → a new CONFIRM_MERGED.

**Completing the Story after an external fix (OD-17).** An external fix by itself does not complete the Story.

- **Post-Merge Validation Failed.** The Story can move to DONE only when, simultaneously:
  1. the original Integration Match = `PASS`;
  2. the cause of the failure was fixed by another Story or a human;
  3. the human allowed `RESUME` through the Recovery Protocol;
  4. the repeated Post-Merge Validation on the current state of the target branch = `PASS`;
  5. all other conditions of the Definition of Done are met.
- **Integration Mismatch.** A successful execution of another Story does not justify the original integration. DONE is possible only with proof that the approved Changeset Content Identity of this Story has been integrated properly, and with all DoD conditions met. Such proof is a new merge of exactly the approved changes after a correction by a human: `ROUTE_BACK` → HUMAN_MERGE → a new CONFIRM_MERGED → Integration Match `PASS`. `RESUME` for the original non-matching merge gives only a repeated `BLOCKED`. If this cannot be confirmed, the Story remains `BLOCKED` or ends as `CANCELLED` by a human's decision.

These rules are also set by `post_merge_validation.resume_rules` in `stage-map.yaml`.

### Procedure

1. **Record the blocking:** the reason, the source (Stage Verdict / Orchestrator), evidence, what is needed for recovery.
2. **Obtain the human's decision** from the list of Recovery actions. The agent does not make the Recovery decision itself.
3. **Check the preconditions:** for `RESUME` — the cause of the blocking is eliminated or the escalation decision is recorded in the canonical source, the Review Binding Guard is satisfied for the current stage, and the approvals on which the current stage relies are in the `VALID` state (after RECONFIRM, if needed); for `ROUTE_BACK` — the target is allowed by `recovery_route_back_targets`.
4. **Change the state** according to the action.
5. **Record the Recovery event** in History: the decision, who made it, the justification.

If a precondition is not met, the action is not applied, and the Story remains `BLOCKED`.

## 10. Retry Policy

The Retry Budget cannot be infinite (Q15). The Retry Policy is configurable (OD-08). The approved initial configuration values are set by `retry_policy` in `stage-map.yaml`: Technical Attempts — 3, Quality Loops — 3, Repeated Failure — 3. These are not immutable global constants: they can be changed only by a change of the Workflow Definition approved by a human.

### Accounting

| Counter | What it counts | Accounting key |
|---|---|---|
| Technical Attempts | Consecutive attempts of a stage that ended with `INTERRUPTED`, `FAILED` or `INVALID_RESULT` | The stage |
| Quality Loops | Returns through `CHANGES_REQUIRED` from agent stages | Separately: the target producer stage and the initiator stage of the return. The limit applies to each key |
| Repeated Failure | The number of occurrences of the same **Finding Identity**. The limit of 3 is reached at the third occurrence of the problem, that is, at the second repetition after it was marked as fixed | The Finding Identity within the whole Story, regardless of the stage that detected it |

**Finding Identity** is the logical identity of a problem: what is violated (an Acceptance Criterion, a rule, a design requirement) and where. The format is defined by the Artifact Contracts stage. The same problem detected by another stage or after a return to another stage has the same Finding Identity. An identical technical error (the same type of failure of the same stage) is accounted for in the same way.

Accounting by two Quality Loops keys makes it impossible to bypass the limit: a review stage that returns the Story to different producer stages exhausts the initiator stage's limit, and a producer stage to which different reviews return the Story exhausts the target's limit.

### Reset

- Technical Attempts is reset when the stage returns an accepted result (`COMPLETED`).
- Quality Loops for producer stages whose artifacts are part of a gate's Approval Scope, and for review stages before this gate, are reset only by the human's full APPROVE decision at this gate.
- Repeated Failure is not reset within a Story.
- RECONFIRM, `RESUME`, `ROUTE_BACK`, REQUEST_CHANGES and a new session do not reset counters.

### Exhaustion and Extension

- After any limit is exhausted, the Story moves to `BLOCKED` with the reason Retry Budget Exhausted. Repetition of a Finding Identity up to the Repeated Failure limit leads to an early escalation, without waiting for Quality Loops to be exhausted.
- The agent does not increase the budget. An extension is possible only by a human's decision through Recovery: `RESUME` with a specified number of additional attempts for a specific counter or `ROUTE_BACK` to another route. The extension is recorded in History.
- Waiting for a human's decision does not consume the Retry Budget.
- The human's REQUEST_CHANGES decision at a gate does not count toward the agent counters. Subsequent returns from agent stages count as usual.
- A change of the default values is a change of the Workflow Definition approved by a human.
- The history of attempts, counters and budget extensions is not deleted. A counter reset is an event in History, not a deletion of previous records.

## 11. Event History

History is an append-only log of Story events (Q9). An event is not changed or deleted.

Events are recorded for:

- Story creation and Execution Mode changes;
- the start, completion, interruption, failure and result rejection of a stage attempt;
- applied and rejected transitions and Loopbacks;
- Human Gate requests and decisions (APPROVE, RECONFIRM, REQUEST_CHANGES, CANCEL), human actions and confirmations of a Loopback through a gate;
- each entry into the Rework Protocol: mechanism, source, target, reason (section 14);
- the creation of versions of controlled artifacts, Currency changes, `revalidation-record`, Approval State changes;
- `applicability-decision` records and their re-evaluation;
- the Review Binding Guard firing and PR metadata changes;
- blockings, Recovery decisions, Retry Budget exhaustion and extension;
- a Foundation Change Request and its resolution;
- the check before merge (in particular the Merge Protection Status), the confirmation of the merge by a human and the result of INTEGRATION_CONFIRMATION, in particular `post-merge-validation-record`;
- `risk-acceptance-record` and `foundation-exception-record`;
- the creation of a `backlog-entry` for FOLLOW_UP;
- session recovery and detected integrity violations;
- the completion (`DONE`) and cancellation (`CANCELLED`) of the Story, in particular `cancellation-record`.

Each Gate Decision and Recovery event records who made the decision and regarding which Approved Content Identity. The set of event fields is defined by the Artifact Contracts stage.

## 12. Session Recovery

The process state does not depend on the Claude Code session (Q9). A new session resumes work only from the Workflow State, History and canonical artifacts.

During recovery, the Orchestrator:

1. **Loads the state** of the selected Story.
2. **Checks integrity:** the state is consistent with the latest History events; the artifacts referenced by the state exist and have recorded versions; the feature branch corresponds to the Git Binding. A discrepancy is not fixed automatically: the Story moves to `BLOCKED` with the reason Integrity Error.
3. **Handles an incomplete attempt:** if the Execution State is `RUNNING`, but there is no result, the attempt receives `INTERRUPTED`. This is not the `BLOCKED` verdict. Partial results of an interrupted attempt are not considered completed outputs. A new attempt may use them only after a check.
4. **Checks Staleness and changes outside the Workflow:** whether the Foundation, upstream artifacts, the content of artifacts, the Changeset Content Identity or the Target Base Identity have changed since the last run. Detected changes are handled by the Rework Protocol (section 14).
5. **Continues work** according to the Execution Mode: a new attempt of the stage within Technical Attempts or waiting for a human's decision.

## 13. What the Next Stages Define

| Topic | Stage |
|---|---|
| Schemas of the state file and the History file, paths, event identifiers | Artifact Contracts |
| The way artifacts are versioned and dependencies are recorded | Artifact Contracts |
| The mechanism of Approved Content Identity, Changeset Content Identity, Target Base Identity | Artifact Contracts |
| The format of Finding Identity, Applicability Basis, `revalidation-record`, `applicability-decision`, `post-merge-validation-record`, `risk-acceptance-record`, `foundation-exception-record`, `cancellation-record` | Artifact Contracts |
| Detection of changes of the Applicability Basis, Merge Protection Status | Story Orchestrator, Stage Skills |
| The list of dependencies for each artifact type | Artifact Contracts |
| The order of writing the state and History and the detection of discrepancies | Story Orchestrator |
| Locking of concurrent work of several sessions with one Story | Story Orchestrator |
| Computation of Currency, Approval State and the Review Binding Guard | Story Orchestrator |
| Tests of transitions, gates, Recovery, Retry, Staleness, Session Recovery, Review Binding Guard | Harness Validation |

## 14. Rework Protocol

The Rework Protocol is the single procedure for any return of the Story to a previous stage or a repeated check after changes (WF-04, `stage-map.yaml` RR-11). Individual mechanisms do not have their own invalidation rules.

### Procedure

1. **Check the route** for the mechanism: the target is allowed by `stage-map.yaml` (loopbacks, gate decisions, `recovery_route_back_targets`), and the route's `condition` is met.
2. **Obtain the human's decision** if the return crosses a Human Gate or the mechanism is by definition a human's decision.
3. **Record the event** of entry into the Rework Protocol: mechanism, source, target, reason, who made the decision.
4. **Move to the target.** The target stage performs the work. If it creates a new version of an artifact, the previous one becomes `SUPERSEDED`.
5. **Update the Currency** of dependent versions: those that reference the previous version of the upstream and do not have a `revalidation-record` for the new one become `STALE` (section 8).
6. **Update the Approval State:** approvals whose Scope includes a `SUPERSEDED` version or a replaced supporting record become `INVALIDATED`; approvals for which the upstream has changed since the Approval Basis become `SUSPENDED` (section 7).
7. **Go through the subsequent stages** in order. Each stage performs Revalidation (section 8). Review stages check new versions and new `revalidation-record` items. Optional Stages re-evaluate applicability if any relevant fact of the Applicability Basis has changed (section 6), in particular through this very return, a human's decision or a change of Retry Counters. The Review Binding Guard applies to stages after CODE_REVIEW.
8. **At each Human Gate**, the human makes APPROVE (if the approval is `INVALIDATED`) or may make RECONFIRM (if the conditions of section 7 are met). Auto Mode stops.

### Application to Mechanisms

| Mechanism | Steps 1–2 (route and human decision) | Steps 3–8 |
|---|---|---|
| Stage Loopback (`CHANGES_REQUIRED`) | Target from `loopbacks`. A human — if there are `crosses_gates` | The same |
| REQUEST_CHANGES at a Human Gate | Target from the gate's decisions. The human's decision | The same |
| REQUEST_CHANGES at HUMAN_MERGE | Target from the HUMAN_MERGE decisions. The human's decision | The same |
| `loopback_confirmation` ROUTE_BACK | Target from `loopbacks`. The human's decision | The same |
| Recovery ROUTE_BACK | Target from `recovery_route_back_targets`. The human's decision | The same |
| Recovery RESUME | Target — the current stage. The human's decision. The preconditions of section 9 | Steps 5–6 for changes that occurred during the blocking. For approvals of previous gates — Out-of-Stage Revalidation and RECONFIRM within the Recovery decision (sections 7, 8) |
| Recovery after a Foundation Change Request | `RESUME` or `ROUTE_BACK` according to the result of the Change Impact Assessment. The human's decision | The same. The Foundation is upstream for all artifacts. For `RESUME` — Out-of-Stage Revalidation and RECONFIRM |
| Staleness detected during Session Recovery | The Orchestrator records the change. The Story moves to `BLOCKED` (Upstream Changed), then — Recovery | Steps 5–6 immediately, then — the same |
| A change of an artifact outside the Workflow | Detected as a new version or a new Changeset Content Identity. The Story moves to `BLOCKED` (Integrity Error). The human's decision through Recovery | The same. A new Changeset Content Identity requires a ROUTE_BACK to IMPLEMENTATION or earlier (RR-10) |
