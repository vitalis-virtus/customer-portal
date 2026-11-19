# Workflow Policies — Workflow Definition

**Document status:** `APPROVED` (revision 3). Approved by a human on 2026-11-16 after the final Review.

## 1. Purpose

This document defines the normative policies of the Story Workflow: the boundaries of agent autonomy, Human Gates, Reviews, Optional Stages, Foundation changes, Findings, the Definition of Done, Git integration, execution modes and Story isolation.

Routes are defined by [stage-map.yaml](stage-map.yaml). Stages are described in [stages.md](stages.md). The state model — in [workflow-state-model.md](workflow-state-model.md).

The policies do not depend on the project's technologies. Technology rules (language, frameworks, stores, specific Security Rules) belong to the Project Foundation and are applied by stages by referencing it. These policies describe that the Foundation must be complied with, but do not duplicate its rules.

The normative rules of this document do not yet have programmatic verification. They will be enforced by the Orchestrator, Stage Skills and Harness Validation at the next stages.

## 2. Controlled Autonomy

The agent makes technical decisions within the approved documents (Q7): the Foundation, the approved specification, the approved design and the Implementation Plan.

**The agent decides on its own:**

- implementation details that do not contradict the approved design and the Foundation;
- the internal structure of the code within the Foundation rules;
- the composition and form of tests within the plan's testing approach;
- the wording of the artifacts its stage creates.

**Escalation to a human is needed:**

| Situation | How it is handled |
|---|---|
| A change to the Product Foundation | Foundation Change Request (section 6) |
| A change to the Architecture Foundation | Foundation Change Request (section 6) |
| A significant new dependency (criteria below) | `BLOCKED` (Escalation Required). If the Foundation requires a document update for a new technology — Foundation Change Request |
| Weakening or violation of a mandatory Foundation rule, in particular Security Rules | `BLOCKED` (Foundation Rule Violation). Only a Foundation Change Request or an allowed exception (section 7). Escalation and Risk Acceptance do not weaken the rules |
| A change to approved requirements | A Loopback through a gate with a human's decision |
| Acceptance of a significant risk that does not violate mandatory Foundation rules | `BLOCKED` (Escalation Required) → Risk Acceptance by a human (section 7) |

Additional rules:

- If a requirement or an important architectural decision is not defined and there is no answer in the documentation, the agent explains the uncertainty and asks the human for a decision. The agent does not invent requirements.
- The human's decision on an escalation is recorded in the canonical artifact it concerns. A record only in History is insufficient.
- The agent does not propose Risk Acceptance for a violation of a mandatory Foundation rule (`stage-map.yaml` RR-14).
### Risk-Based Dependency Approval (OD-09)

A new dependency or a change of an existing one requires a human's approval if it significantly affects at least one of:

- **architecture** — a new technology in the role of a system component, a replacement of an existing component, a new runtime service;
- **security** — cryptography, authentication, parsing of untrusted data, network interaction, access to secrets;
- **operational requirements** — a new service or infrastructure to run, significant resource or environment requirements;
- **production credentials** — new credentials, keys or secrets are needed for production;
- **license obligations** — a license with obligations for the project or a non-standard or commercial license;
- **compatibility** — a change of the language or runtime version, a major update of an existing dependency, breaking changes.

Rules:

- Simple auxiliary dependencies that do not meet any criterion may be proposed by the agent in `solution-design` or `implementation-plan` within Controlled Autonomy. The human sees them at HUMAN_TECHNICAL_APPROVAL.
- An unplanned dependency that appeared during IMPLEMENTATION is recorded in the changeset. CODE_REVIEW checks its classification. A significant unplanned dependency → `BLOCKED` (Escalation Required).
- If the classification is ambiguous, the dependency is considered significant.
- Stricter rules of the Project Foundation take precedence. For Customer Portal, the Architecture Foundation rule on new technologies applies.
- The Workflow does not provide for a formal risk assessment of dependencies (scoring).

## 3. Human Gates

Three Human Gates approve artifacts (Q5):

| Gate | What it approves |
|---|---|
| HUMAN_SPEC_APPROVAL | The Story requirements: `story-spec`, `clarification-record` |
| HUMAN_TECHNICAL_APPROVAL | The technical design and the Implementation Plan together |
| HUMAN_PR_APPROVAL | A specific version of the Story's changes in the PR |

HUMAN_MERGE is a separate delivery action that a human confirms. It is not an approval of artifacts.

Rules:

- **A gate cannot be bypassed automatically.** No execution mode, verdict or configuration replaces a human's decision.
- **A Reviewer PASS is not a Human Approval.** A gate opens only after `PASS` of the corresponding review stages, but even then it requires a human's decision.
- **An approval is bound to the Approved Content Identity** of each artifact of the Approval Scope (`approval_scope` in `stage-map.yaml`). Validity rules — [workflow-state-model.md](workflow-state-model.md#7-approval-validity).
- **Decision package.** The human receives the Scope artifacts, review reports, `applicability-decision`, the Findings Disposition, risks and escalations. During a repeated pass — the changes relative to the approved identity, the results of the Change Impact Assessment and `revalidation-record`.
- **Decision.** The human chooses one of the gate's decisions in `stage-map.yaml`. REQUEST_CHANGES must contain a target from those allowed for this gate and a reason. Then the Rework Protocol applies.
- **Repeated pass (OD-06).** A gate is a mandatory stop during every pass. If the approved content has not changed, the approval is in the `VALID` or `SUSPENDED` state, and each upstream change after the human's last decision is covered by a checked `revalidation-record`, the human may perform a shortened RECONFIRM. A `SUSPENDED` approval does not become `VALID` without a human's decision. If the approved identity has changed or the approval is in the `INVALIDATED` state, a full APPROVE is needed. RECONFIRM is not an approval of changed content and is recorded as a separate event with a reference to the original approval.
- **Auto Mode** does not perform APPROVE or RECONFIRM and does not choose a decision instead of the human.
- **Boundary of the gate's authority (WF-V2-01).** A human's decision at a gate cannot accept a violation of a mandatory Foundation rule, a violation of an Acceptance Criterion or an unmet DoD condition. APPROVE is impossible in the presence of such a BLOCKING finding. The possible decisions are REQUEST_CHANGES, CANCEL or a separate Foundation Change Request.
- The human's decision is recorded in the Workflow State and History together with the Approved Content Identity to which it belongs.

## 4. Review Policies

**Mandatoriness (Q28).** A Review is mandatory if the producer stage it checks is applicable. The producer → review mapping is set by the `reviewed_by` field in `stage-map.yaml`. A Review cannot be bypassed through `NOT_APPLICABLE`:

- mandatory review stages (SPEC_REVIEW, DESIGN_REVIEW, PLAN_REVIEW, CODE_REVIEW) do not have `optional: true` and cannot return `NOT_APPLICABLE`;
- each mandatory producer stage is checked by a mandatory review stage or a Human Gate;
- SECURITY_REVIEW is an additional specialized check. Its `NOT_APPLICABLE` does not exempt the changes from DESIGN_REVIEW, CODE_REVIEW and the basic security checks in VERIFICATION.
- A change of the Changeset Content Identity after CODE_REVIEW requires a new CODE_REVIEW, and if SECURITY_REVIEW is applicable — also a new SECURITY_REVIEW. This is required by the Review Binding Guard (`stage-map.yaml` RR-10). A change of only PR metadata does not require a new review (RR-12).

**Two-level security check (OD-05):**

1. DESIGN_REVIEW checks security decisions at the architecture level for every Story.
2. SECURITY_REVIEW, if applicable, performs an in-depth check of the implementation after CODE_REVIEW.

There is no separate Security Design Review Stage.

**Independence (Q6) — a hybrid model:**

- The Producer and the Reviewer have different logical roles and responsibilities.
- The Reviewer does not fix the artifact it checks. It records findings and a verdict. The fix is made by the producer stage after the return.
- The Reviewer receives canonical artifacts, not the working context of the producer stage.

**Context isolation (OD-10).** An isolated context means that the reviewer is executed separately from the producer stage and receives only canonical artifacts, `revalidation-record` and Foundation sources. It does not see the working context, reasoning and drafts of the producer stage. The implementation mechanism (for example, Subagents) is defined by the next stage.

- Isolation is always mandatory for DESIGN_REVIEW, CODE_REVIEW and SECURITY_REVIEW.
- For SPEC_REVIEW and PLAN_REVIEW, isolation is mandatory if at least one elevated-risk trigger from `review_context_isolation.risk_triggers` in `stage-map.yaml` fires: Story Type SECURITY or INFRASTRUCTURE; changes to authentication, authorization, sessions, credentials or sensitive data; SECURITY_REVIEW is planned (for PLAN_REVIEW); the artifact was created after a return through a Human Gate; Repeated Failure fired or the Retry Budget was extended; a human requires isolation.
- If the applicability of a trigger is ambiguous, isolation is needed.

**Binding to versions.** A review report is valid only for the versions of artifacts and upstream versions against which the check was performed. After a new version, a new review is needed. For an unchanged version with a new `revalidation-record`, the review checks this record.

## 5. Optional Stage Policies

The Workflow is one for all Stories (Q21). The complexity of a Story affects only the applicability of Optional Stages: IMPACT_ANALYSIS, API_DESIGN, DB_DESIGN, SECURITY_REVIEW, RECONCILIATION.

Rules:

The applicability conditions of each Optional Stage are defined in `applicability_conditions` in `stage-map.yaml` as a list of triggers (OD-07). The common rules AP-01…AP-07 are defined there as well. In short:

- A stage is applicable if at least one trigger is met. If applicability is ambiguous, the stage is considered applicable.
- `NOT_APPLICABLE` requires an `applicability-decision`: for each trigger — why it is not met, with a reference to the source.
- An `applicability-decision` is bound to the Applicability Basis — all relevant facts of the decision: artifact versions, Changeset Content Identity, Workflow State facts, relevant History events, Retry Counters, human decisions, the Story Risk Assessment (WF-V2-02, AP-04). The relevant categories for each stage are set by `basis_facts`.
- If a relevant fact has changed, the decision is re-evaluated before use (AP-08). In particular, a return through a Human Gate, Repeated Failure, a Retry Budget extension, new security findings, a human's decision to apply a Review, Recovery after BLOCKED or a change of the Story Risk Assessment do not allow reusing a previous `NOT_APPLICABLE` for SECURITY_REVIEW or RECONCILIATION. This is checked by the stage on entry, the stage from `na_checked_by` and the Orchestrator in the Review Binding Guard.
- Events outside a stage's `basis_facts` do not invalidate its decision (AP-09).
- The planned decision in the Implementation Plan is a forecast based only on the plan's artifacts. After entry into the stage, the decision on entry is valid (AP-10).
- The `NOT_APPLICABLE` justification is checked by the stage from the `na_checked_by` field and seen by the human at the nearest Human Gate.
- The Implementation Plan declares the expected applicability of SECURITY_REVIEW and RECONCILIATION. The human approves it at HUMAN_TECHNICAL_APPROVAL and may require the application of a stage.
- A stage approved as applicable cannot later be declared `NOT_APPLICABLE` without a human's decision. The agent may declare a non-applicable stage applicable without additional approval.
- There is no automatic Risk Scoring Engine: triggers are evaluated by the stage, and checked by the reviewer and the human.

**SECURITY_REVIEW (Q22).** For changes that meet the applicability condition, the stage is mandatory. Basic security checks remain part of VERIFICATION for all Stories.

**RECONCILIATION (Q27).** A full Reconciliation is performed for Stories that meet the applicability condition. The basic check of compliance with requirements in VERIFICATION is mandatory for all Stories.

## 6. Foundation Change Management

A Stage Skill does not change the approved Product or Architecture Foundation (Q19).

If a Story requires a change to the Foundation:

1. **The Story stops.** The stage returns `BLOCKED` with the reason Foundation Change Required.
2. **A Change Request is produced** (`foundation-change-request`): what to change, why, which approved decisions it affects, alternatives within the current Foundation.
3. **A human reviews the change** and approves or rejects it.
4. **The Foundation is updated** after approval. The update is separate work outside the Story Workflow, approved by a human.
5. **A Change Impact Assessment is performed** for the artifacts of this Story. Other non-terminal Stories check Staleness during their next run (section 10, [Session Recovery](workflow-state-model.md#12-session-recovery)), because one Orchestrator run works only with one Story.
6. **Dependent results go through Revalidation** ([workflow-state-model.md](workflow-state-model.md#8-staleness-and-revalidation)). For unchanged versions, `revalidation-record` items are created; for affected ones — new versions.
7. **The Story is resumed** through the Recovery Protocol: `RESUME` or `ROUTE_BACK` according to the rules of the Rework Protocol ([workflow-state-model.md](workflow-state-model.md#14-rework-protocol)). For `SUSPENDED` approvals of previous gates, Out-of-Stage Revalidation is performed ([workflow-state-model.md](workflow-state-model.md#8-staleness-and-revalidation)), after which the human confirms them through RECONFIRM within this decision. If an approval is in the `INVALIDATED` state, a `ROUTE_BACK` to the producer stage of the corresponding artifact is needed.

If the change is rejected, the Story either adapts within the current Foundation (`ROUTE_BACK`) or is cancelled (`CANCEL`).

The agent reports a contradiction between Foundation documents and does not resolve it on its own in favor of one of the documents.

## 7. Findings Disposition

Each finding of a review or verification stage has a class (Q23):

| Class | Meaning | Consequence |
|---|---|---|
| BLOCKING | A violation that does not allow moving forward | Verdict `CHANGES_REQUIRED` or `BLOCKED`. Must be eliminated before `PASS` |
| NON_BLOCKING | A shortcoming that does not violate requirements and rules | Fixed or accepted with a justification in the Findings Disposition |
| FOLLOW_UP | Work outside the Story's scope | Requires a human's approval at HUMAN_PR_APPROVAL. An approved FOLLOW_UP becomes a `backlog-entry` in the canonical Backlog |

Rules:

- A violation of a mandatory Acceptance Criterion is always BLOCKING. It cannot be moved to FOLLOW_UP.
- A violation of a mandatory rule of the Product, Architecture or Security Foundation is always BLOCKING until it is properly resolved (see below, `stage-map.yaml` RR-14).
- The class is proposed by the reviewer. The agent does not lower the BLOCKING class. A human may lower the class only for a finding that is not a violation of an Acceptance Criterion, a mandatory Foundation rule or a DoD condition. The decision is recorded.
- BLOCKING is not moved to FOLLOW_UP if this would allow bypassing a mandatory requirement. There is no automatic lowering of the class.
- The Findings Disposition (how each finding is resolved) is part of `verification-report` and the HUMAN_PR_APPROVAL package.

### Risk Acceptance and Foundation Rules (WF-V2-01)

| Case | What is possible | Record |
|---|---|---|
| **A. Risk Acceptance without a Foundation violation** | A human explicitly accepts a justified risk that does not contradict mandatory rules of the Product, Architecture or Security Foundation. The finding may receive the Disposition "risk accepted" | `risk-acceptance-record` (Finding Identity, risk, justification, who decided, boundaries) and the Findings Disposition. An event in History |
| **B. A violation of a mandatory Foundation rule** | Risk Acceptance and a decision at a gate do not eliminate the violation. Options: fix the implementation; an approved Foundation Change Request with a Foundation update (section 6); a controlled exception, only if that specific Foundation rule explicitly allows an exception — under the procedure defined in the Foundation | `foundation-change-request` or `foundation-exception-record` with a reference to the Foundation provision that allows the exception. An event in History |

Rules:

- If a Foundation rule does not explicitly allow an exception, a Human Approval cannot create one. Only a fix or a Foundation Change Request remain.
- A case B finding remains BLOCKING until there is a fix, an updated Foundation or a valid `foundation-exception-record`.
- For Customer Portal, the Foundation does not contain a general exception procedure. Individual rules define the conditions of their own exception (for example, "only by separate approval"). The Workflow does not extend these conditions.
- If it is not obvious whether a risk violates a mandatory Foundation rule, case B applies.
- Each finding has a Finding Identity ([workflow-state-model.md](workflow-state-model.md#10-retry-policy)). A repeated occurrence of the same problem is the same finding, even if another stage detected it.

### FOLLOW_UP Backlog (OD-11)

- Each FOLLOW_UP approved by a human creates a trackable `backlog-entry` record in the project's canonical Backlog.
- A `backlog-entry` references the original Story and the Finding Identity.
- The records are created by INTEGRATION_CONFIRMATION before the Story moves to DONE. The Story does not become DONE until there is a `backlog-entry` for each approved FOLLOW_UP.
- A `backlog-entry` lives independently of the Story: the Story's transition to DONE or CANCELLED does not close or delete it.
- The Backlog is a shared project artifact, not a Story artifact. Creating a `backlog-entry` is an allowed exception from Story isolation (section 10).
- There is no canonical Backlog yet. Its existence is a prerequisite (section 15). The record structure is defined by the Artifact Contracts stage.

## 8. Definition of Done

A Story receives `DONE` only when all conditions are met (Q4, Q23).

**Checked by VERIFICATION:**

1. All mandatory stages have `PASS`, and Optional Stages — `PASS` or `NOT_APPLICABLE` with a checked `applicability-decision`.
2. The mandatory reviews have been performed for the current Changeset Content Identity and the current versions of artifacts (Review Binding Guard).
3. All controlled artifacts have Currency `CURRENT`, and review reports are bound to the current versions.
4. Each Acceptance Criterion has actual evidence of fulfillment.
5. The checks required by the project's Testing Strategy, the Story Risk Assessment and the plan's Testing Approach have been performed and passed (section 18).
6. There are no unresolved BLOCKING findings. Violations of mandatory Foundation rules have been resolved by a fix, a Foundation Change Request or a valid `foundation-exception-record`, not by Risk Acceptance. NON_BLOCKING and FOLLOW_UP have a Disposition.
7. Human decisions recorded in History are reflected in canonical artifacts. Only VERIFICATION checks this. RECONCILIATION does not duplicate it.

**Confirmed by a human:**

8. HUMAN_PR_APPROVAL is in the `VALID` state for the current Changeset Content Identity.
9. HUMAN_MERGE has been performed.

**Confirmed by MERGE_READINESS_CHECK and INTEGRATION_CONFIRMATION:**

10. Before the merge, the freshness of the feature branch and the target branch has been checked (section 9).
11. The merge introduced exactly the approved changes into the target branch, without unapproved content.
12. If the Target Base Identity at the moment of the merge differed from the checked one, Post-Merge Validation has passed.
13. A `backlog-entry` has been created for each approved FOLLOW_UP (section 7).

Readiness for a PR or an open PR does not mean DONE.

## 9. Git Integration

Rules (Q8):

- Each Story works in a separate feature branch.
- The agent may create commits in the feature branch and prepare a PR to the main branch (target branch).
- Only a human performs the merge into the target branch.
- DONE is possible only after a confirmed integration of the approved changes.

### Identity of Changes

Checks and approvals are bound not only to Git SHA, but to two identities ([workflow-state-model.md](workflow-state-model.md#5-artifact-status)):

- **Changeset Content Identity** — the content of the Story's changes. Rebuilding commits (squash, rebase without conflicts, renaming commits) does not change it if the content of the changes is the same.
- **Target Base Identity** — the version of the target branch against which the integration result has been checked.

A new Changeset Content Identity is created only by IMPLEMENTATION. In particular, resolving a conflict with the target branch is a change of content that is performed in IMPLEMENTATION and goes through CODE_REVIEW, SECURITY_REVIEW (if applicable) and VERIFICATION (`stage-map.yaml` RR-10).

### Merge Validation Protocol (OD-12)

1. **Binding of the check.** VERIFICATION binds `verification-report` to the Changeset Content Identity and the Target Base Identity.
2. **The check before merge.** MERGE_READINESS_CHECK after HUMAN_PR_APPROVAL checks that the feature branch contains the approved Changeset Content Identity, the PR corresponds to the approved identity, and the target branch has not changed relative to the checked Target Base Identity.
3. **The target branch has changed.** If the content of the Story's changes is the same, a check of the integration result is needed: a return to VERIFICATION with the condition `CHANGESET_CONTENT_UNCHANGED`. A new `verification-report` changes the Approval Scope, so HUMAN_PR_APPROVAL becomes `INVALIDATED`.
4. **A code change is needed for integration.** A return to IMPLEMENTATION. Then CODE_REVIEW, SECURITY_REVIEW (if applicable), RECONCILIATION (if applicable) and VERIFICATION.
5. **A stale approval.** An `INVALIDATED` HUMAN_PR_APPROVAL requires a full APPROVE. RECONFIRM is possible only when no artifact of the Scope has changed.
6. **Only a human performs the merge** (HUMAN_MERGE). The agent does not perform merge, revert or push into the target branch.
7. **The confirmation after merge.** INTEGRATION_CONFIRMATION checks the actual merge result: the Story's changes in the target branch correspond to the approved Changeset Content Identity, there is no unapproved content. Then it compares the Target Base Identity at the moment of the merge with the checked one.
8. **DONE** — only after `PASS` of INTEGRATION_CONFIRMATION.

The check before merge (step 2) and the confirmation after merge (step 7) are different stages. The target branch may change between them. This gap is closed by two levels (OD-16): Merge Protection prevents unchecked integration, and Post-Merge Validation checks it in a controlled way if it has nevertheless happened.

### Merge Protection — Prevention (OD-16)

- If the Git platform supports target branch protection mechanisms (Branch Protection, Required Checks, prohibition of merging with stale checks or an equivalent), they are used in accordance with the project's **Merge Protection Policy** (`stage-map.yaml` RR-13). The Merge Protection Policy defines which mechanisms are mandatory. This is a project rule, so WF-V2-01 applies to it. The Workflow Core does not depend on a specific platform or merge strategy.
- MERGE_READINESS_CHECK records the **Merge Protection Status** and acts according to it (`merge_protection_statuses`, FR-02):

| Status | Case | Action |
|---|---|---|
| `ACTIVE` | The mechanism is enabled | A regular check before merge |
| `NOT_SUPPORTED` | The platform does not support the mechanism | A controlled manual procedure is allowed (below). The limitation is recorded. If the Merge Protection Policy requires exactly this mechanism, the Story is handled as `DISABLED` |
| `UNAVAILABLE` | The mechanism is supported but objectively unavailable | The limitation is recorded. `BLOCKED` (Merge Protection Unavailable). The human in Recovery approves an acceptable integration order. The decision is recorded in the Workflow State and History |
| `DISABLED` | The mechanism is available and mandatory under the Merge Protection Policy, but disabled | `BLOCKED` (Merge Protection Disabled). The merge remains blocked until protection is enabled or a valid foundation-exception-record exists, provided that the Merge Protection Policy explicitly permits the exception. A manual procedure does not replace this case.

A mechanism that is available but not mandatory under the policy and is disabled is handled as `UNAVAILABLE`.

- **A controlled manual procedure** (`NOT_SUPPORTED` or an approved order for `UNAVAILABLE`): the human performs the merge immediately after MERGE_READINESS_CHECK and, together with CONFIRM_MERGED, confirms that the target branch has not changed after the check. If it has changed, the human does not perform the merge, but chooses REQUEST_CHANGES → MERGE_READINESS_CHECK. A manual procedure does not bypass mandatory project rules.
- The agent does not change the repository protection settings without a human's permission.
- Merge Protection reduces the risk, but does not replace INTEGRATION_CONFIRMATION.

### Post-Merge Validation — Recovery (OD-16)

Post-Merge Validation is part of INTEGRATION_CONFIRMATION, not a separate stage. It uses the same stage results (`PASS` → DONE, `BLOCKED`, a technical failure → `FAILED`), so it does not require new routes, gates or Loopbacks. The order of checks in the stage is mandatory (`post_merge_validation.check_order` in `stage-map.yaml`):

1. **Integration Match.** The Story's merge result is evaluated, not a later state of the target branch. The merge had to introduce exactly the approved Changeset Content Identity without unapproved content. Otherwise — `BLOCKED` (Integration Mismatch), and Post-Merge Validation is not performed. This way the Story cannot become DONE with a non-matching changeset.
2. **Base Comparison.** If the Target Base Identity at the moment of the merge equals the checked one, Post-Merge Validation is not needed.
3. **Post-Merge Validation** — only if the base has changed:
   - the actual integrated result on the current state of the target branch is checked;
   - the relevant integration checks and tests are performed according to the Testing Strategy, the Story Risk Assessment and the Testing Approach;
   - the result and the checked state of the target branch are recorded in an immutable `post-merge-validation-record`;
   - `PASS` allows continuing, `FAIL` gives `BLOCKED` (Post-Merge Validation Failed) with diagnostics in `post-merge-validation-record`;
   - a technical failure to run the checks is `FAILED` within Technical Attempts, not a verdict.
4. **FOLLOW_UP Backlog.** `backlog-entry` items are created.

Rules:

- The agent does not change the target branch: it does not perform merge, revert or push and does not automatically fix already integrated code.
- Post-Merge Validation does not invalidate HUMAN_PR_APPROVAL if the Changeset Content Identity has not changed. `post-merge-validation-record` is not part of the Approval Scope ([workflow-state-model.md](workflow-state-model.md#7-approval-validity)).
- A mismatch of the integrated result or a change of approved code is handled by the current rules of Approval Validity, the Recovery Protocol and the Rework Protocol.

**Independence from the merge strategy.** The protocol does not depend on whether a merge commit, a squash merge or a rebase is used:

| Strategy | What INTEGRATION_CONFIRMATION checks |
|---|---|
| Merge commit | The integrated branch has the approved Changeset Content Identity, and the merge did not add other content |
| Squash merge | The changes of the squash commit relative to the Target Base Identity correspond to the approved Changeset Content Identity |
| Rebase | The cumulative changes of the moved commits correspond to the approved Changeset Content Identity |

The specific comparison method is defined by the Stage Skills and Artifact Contracts.

**Integration Mismatch and Post-Merge Validation Failed.** In both cases, INTEGRATION_CONFIRMATION returns `BLOCKED`, and the Story does not become DONE. The agent does not automatically fix an already performed merge. The decision is made by a human through the Recovery Protocol ([workflow-state-model.md](workflow-state-model.md#9-blocked-and-recovery-protocol)).

**Completion after an external fix (OD-17).** An external fix by itself does not complete the Story.

- **Post-Merge Validation Failed.** DONE is possible if the original Integration Match = `PASS`, the cause was fixed by another Story or a human, the human allowed `RESUME`, the repeated Post-Merge Validation = `PASS`, and all other DoD conditions are met.
- **Integration Mismatch.** A successful other Story does not justify the original integration. DONE is possible only with proof that the approved Changeset Content Identity of this Story has been integrated properly, and with all DoD conditions met. Otherwise, the Story remains `BLOCKED` or ends as `CANCELLED` by a human's decision. Proof — a new merge of the approved changes after a correction: `ROUTE_BACK` → HUMAN_MERGE → a new CONFIRM_MERGED. Integration Match always evaluates the merge recorded by the last CONFIRM_MERGED.

Branch naming and the format of commits and PRs are defined by the Stage Skills and Artifact Contracts.

## 10. Multiple Story Isolation

Rules (Q12):

- Each Story has an independent Workflow State, History, artifacts and feature branch.
- One Orchestrator run works only with one Story.
- Switching between Stories is allowed. The state of the Story switched away from is preserved, and it can be resumed later ([workflow-state-model.md](workflow-state-model.md#12-session-recovery)).
- A stage of one Story does not change the state, artifacts or branch of another Story. Reading other Stories is allowed (for example, for IMPACT_ANALYSIS).
- The only exception is creating a `backlog-entry` in the project's shared Backlog for an approved FOLLOW_UP (section 7). The existing records of other Stories are not changed.
- The sources shared by all Stories are the Foundation and the Workflow Definition. Their change triggers a Staleness check in other non-terminal Stories during their next run.
- Automatic parallel orchestration of several Stories is not part of the current version.

## 11. Step Mode and Auto Mode

| Mode | Behavior |
|---|---|
| Step Mode | One stage is executed. After the transition, the run stops |
| Auto Mode | Stages are executed sequentially until a mandatory stop |

**Mandatory stops of Auto Mode** (Q25):

- a Human Gate, including a repeated pass at which RECONFIRM is possible;
- a human action;
- a Loopback that crosses a Human Gate;
- waiting for the human's answers;
- `BLOCKED`, in particular Retry Budget exhaustion, escalation, Foundation Change Request, Foundation Rule Violation, an integrity violation, the Review Binding Guard, Merge Protection Unavailable or Disabled, Integration Mismatch, Post-Merge Validation Failed;
- a terminal state.

Auto Mode does not bypass Human Gates and does not make human decisions: neither APPROVE, nor RECONFIRM, nor Recovery decisions. A Loopback without crossing a gate within the Retry Budget is performed by Auto Mode without stopping.

## 12. Context Management

- **Progressive Context Loading (Q17).** Each stage has a minimum set of sources ([stages.md](stages.md#2-context-sources)). Additional documents are loaded only as needed.
- **Persistent Artifacts (Q18).** Subsequent stages obtain context from the canonical artifacts of previous stages. Chat history is not a source of state, requirements or approved decisions.

## 13. Mapping to Workflow Invariants

| # | Invariant | Where it is ensured |
|---|---|---|
| 1 | There is no transition without an allowed route | `stage-map.yaml` RR-01; state model §6 |
| 2 | A Human Gate cannot be bypassed automatically | §3, §11; `stage-map.yaml` RR-07 (including RECONFIRM) |
| 3 | Reviewer PASS ≠ Human Approval | §3; state model §4 |
| 4 | An approval is bound to the current artifacts | §3; state model §5 (Approved Content Identity), §7 |
| 5 | A human's decision is in the canonical source | §2; state model §2 |
| 6 | Stale artifacts are not used without Revalidation | state model §5, §8, §14 |
| 7 | A disallowed Loopback is not performed | `stage-map.yaml` RR-03; state model §6 |
| 8 | BLOCKED has a Recovery Protocol | state model §9 |
| 9 | The Retry Budget is finite | state model §10 |
| 10 | NOT_APPLICABLE requires a justification | §5; `stage-map.yaml` RR-02 |
| 11 | Stage Skills do not change the Workflow State | state model §2 |
| 12 | DONE only after Git Integration | §8, §9; `stage-map.yaml` RR-09, RR-10, RR-13, `post_merge_validation` |
| 13 | Chat History is not a source of truth | §12; state model §2 |
| 14 | The Workflow Core does not depend on Go, Gin, PostgreSQL | §1; [stages.md](stages.md#2-context-sources) (the binding to the project is placed separately) |
| 15 | Technology rules are in the Foundation and Stage Skills | §1 |

A normative definition of the invariants is not their programmatic verification. Verification will be provided by the Orchestrator and Harness Validation.

## 14. OD Decisions

### Approved Decisions OD-01…OD-17

All seventeen decisions are approved by a human: OD-01…OD-15 — after Review Draft v1, OD-16 — after Review Draft v2, OD-17 — after the final Review Draft v3. The status of each is `APPROVED_DECISION`. The OD-* identifiers remain stable and are used in all Workflow Definition documents for traceability.

| ID | Status | Decision | Normative rules | Where applied |
|---|---|---|---|---|
| OD-01 | `APPROVED_DECISION` | A mandatory universal SOLUTION_DESIGN stage: the overall technical structure of the solution — Domain, Application, Integration and Infrastructure Design, if the Story needs it | The stage does not return `NOT_APPLICABLE`. DESIGN_REVIEW is always applicable | `stage-map.yaml` (SOLUTION_DESIGN, DESIGN_REVIEW); stages.md §8 |
| OD-02 | `APPROVED_DECISION` | A separate DELIVERY phase separates the technical readiness of the Story from the integration of changes | Phase 7 in `phases` | `stage-map.yaml` (`phases`); stages.md §4, §12 |
| OD-03 | `APPROVED_DECISION` | There is no separate Clarification Review Stage. SPEC_REVIEW checks `clarification-record` and `story-spec` | `reviewed_by: SPEC_REVIEW` for CLARIFICATION | `stage-map.yaml`; stages.md §6, §7 |
| OD-04 | `APPROVED_DECISION` | A separate PLAN_REVIEW stage for an independent check of the Implementation Plan | PLAN_REVIEW is mandatory | `stage-map.yaml`; stages.md §9 |
| OD-05 | `APPROVED_DECISION` | A two-level security check: DESIGN_REVIEW — architecture; SECURITY_REVIEW — an in-depth check of the implementation after CODE_REVIEW, if applicable | There is no separate Security Design Review | §4; `stage-map.yaml` (DESIGN_REVIEW, SECURITY_REVIEW); stages.md §8, §11 |
| OD-06 | `APPROVED_DECISION` | Gates are mandatory stops. Unchanged content after Revalidation → a shortened RECONFIRM. Changed content or `INVALIDATED` → a full APPROVE. Auto Mode does not perform Reconfirmation | `gate_decisions.RECONFIRM`, RR-07 | §3, §11; state model §7; `stage-map.yaml` |
| OD-07 | `APPROVED_DECISION` | Explicit normative applicability conditions for Optional Stages. Ambiguity → applicable. `NOT_APPLICABLE` with a justification and a check. Without a Risk Scoring Engine | `applicability_conditions`, AP-01…AP-07 | §5; `stage-map.yaml` |
| OD-08 | `APPROVED_DECISION` | A configurable Retry Policy with approved initial values 3 / 3 / 3 (Technical Attempts, Quality Loops, Repeated Failure). Accounting and reset rules. Extension — only by a human through Recovery | `retry_policy` | state model §10; `stage-map.yaml` |
| OD-09 | `APPROVED_DECISION` | Risk-Based Dependency Approval. Simple auxiliary dependencies — within Controlled Autonomy. Stricter Foundation rules take precedence | Significance criteria | §2 |
| OD-10 | `APPROVED_DECISION` | Context isolation is mandatory for DESIGN_REVIEW, CODE_REVIEW, SECURITY_REVIEW. For SPEC_REVIEW and PLAN_REVIEW — according to risk triggers | `review_context_isolation`, ISO-01…ISO-06 | §4; `stage-map.yaml` |
| OD-11 | `APPROVED_DECISION` | Approved FOLLOW_UP create a `backlog-entry` in the canonical Backlog with a reference to the Story and the Finding | DoD condition 12 | §7, §8, §10, §15; stages.md §12 |
| OD-12 | `APPROVED_DECISION` | Merge Validation Protocol: binding to the feature and target branch, the check before merge, a check of the integration result, repeated reviews on code changes, full approval of a stale approval, merge only by a human, the confirmation after merge | RR-10, RR-12, `route_conditions`, MERGE_READINESS_CHECK | §9; state model §5, §9; `stage-map.yaml`; stages.md §12 |
| OD-13 | `APPROVED_DECISION` | Cancellation Protocol. CANCELLED is a terminal state, the Story is not reopened. Nothing is deleted without explicit permission | RR-08 | §17; `stage-map.yaml` |
| OD-14 | `APPROVED_DECISION` | One Workflow for the types FEATURE, BUG, TECHNICAL, INFRASTRUCTURE, SECURITY. Clarification and Specification adapt to the type. Without separate Workflow Profiles | `story_types` | §16; stages.md §6, §7; `stage-map.yaml` |
| OD-15 | `APPROVED_DECISION` | Checks are defined by the project's Testing Strategy, the Story Risk Assessment and the plan's Testing Approach. Technology testing rules are not part of the Workflow Core. The absence of a Testing Strategy is a Foundation Gap | FG-01 | §8, §15, §18; stages.md §9, §11 |

| OD-16 | `APPROVED_DECISION` | Option C: Merge Protection (Prevention) and Post-Merge Validation (Recovery). Post-Merge Validation is part of INTEGRATION_CONFIRMATION, without a new stage. The agent does not change the target branch and protection settings | RR-13, `post_merge_validation`, DoD 11–12 | §9; state model §7, §9; `stage-map.yaml` (INTEGRATION_CONFIRMATION); stages.md §12 |
| OD-17 | `APPROVED_DECISION` | Completion after an external fix: Post-Merge Validation Failed → DONE only with the original Integration Match PASS, the human's RESUME, a repeated Post-Merge Validation PASS and the DoD met. Integration Mismatch → DONE only with proof of the proper integration of the approved Changeset Content Identity, otherwise BLOCKED or CANCELLED | `post_merge_validation.resume_rules` | §9; state model §9; stages.md §12 |

### Approved Review Findings

| ID | Status | Essence | Where applied |
|---|---|---|---|
| WF-01 | `RESOLVED` | Version immutability, Currency, `revalidation-record` | state model §5, §7, §8 |
| WF-02 | `RESOLVED` | Review Binding Guard | `stage-map.yaml` RR-10, RR-12 |
| WF-03 | `RESOLVED` | Approved Content Identity for the Approval Scope | state model §5; `stage-map.yaml` |
| WF-04 | `RESOLVED` | A single Rework Protocol | state model §14; RR-11 |
| WF-05 | `RESOLVED` | Distribution between RECONCILIATION and VERIFICATION | stages.md §11 |
| WF-V2-01 | `APPROVED_DECISION` | Risk Acceptance does not eliminate violations of mandatory Foundation rules | §2, §3, §7, §8; RR-14; stages.md §8, §11 |
| WF-V2-02 | `APPROVED_DECISION` | An Applicability Decision is bound to all relevant facts (Applicability Basis) | §5; AP-04, AP-08…AP-10, `basis_facts`, RR-10; state model §6, §14 |

### Final Review Decisions

| ID | Status | Decision | Where applied |
|---|---|---|---|
| FR-01 | `APPROVED_DECISION` | The former `DERIVED` elements are approved: CODE_REVIEW, PR_PREPARATION, MERGE_READINESS_CHECK and the specific Applicability Triggers | `stage-map.yaml` (`definition_status`, `applicability_conditions`); stages.md §4 |
| FR-02 | `APPROVED_DECISION` | Merge Protection distinguishes `NOT_SUPPORTED` (a manual procedure), `UNAVAILABLE` (a human approves the integration order) and `DISABLED` under a mandatory policy (the merge is blocked). A manual procedure does not bypass mandatory rules | RR-13, `merge_protection_statuses`; §9; state model §9; stages.md §12 |
| FR-03 | `APPROVED_DECISION` | SEC-08 applies only to changes that directly affect security controls or behavior governed by specialized Security Rules. Basic security checks remain in DESIGN_REVIEW and VERIFICATION | `stage-map.yaml` (SEC-08) |

### Open Questions

There are no open or blocking questions. New questions are registered here with the next identifier OD-18 and go through separate approval.

## 15. Prerequisites and Foundation Gaps

Prerequisites without which the Workflow cannot be applied to a real Story. They do not change the approved Foundation and are not created within the Workflow Definition.

| ID | Prerequisite | Type | When needed | Consequence if absent |
|---|---|---|---|---|
| FG-01 | The project's Testing Strategy. The Customer Portal Architecture Foundation references it, but the document does not exist yet | Foundation Gap | Before the first real Story | The first real Story is not registered in the Workflow until the Testing Strategy is created and approved by separate work outside the Story Workflow |
| PRQ-01 | The project's canonical Backlog | Prerequisite | Before the first Story whose source is not the Product Foundation, and before the first approved FOLLOW_UP | A Story of this kind is not registered. INTEGRATION_CONFIRMATION cannot create a `backlog-entry`, so the Story does not become DONE |
| PRQ-02 | Merge Protection Configuration: the project's Merge Protection Policy (which mechanisms are mandatory) and the actual state of the mechanisms on the platform (OD-16, FR-02) | Prerequisite | Before the first HUMAN_MERGE | Without a defined policy, no mechanism is considered mandatory, and the `NOT_SUPPORTED` / `UNAVAILABLE` cases apply (section 9). The limitation is recorded in each Story. Only a human changes the settings |

## 16. Story Types (OD-14)

One canonical Workflow supports all types from `story_types` in `stage-map.yaml`. There are no separate Workflow Profiles. The type affects the content of CLARIFICATION and SPECIFICATION and the triggers of Optional Stages and review isolation.

| Story Type | Story source | Mandatory content of the specification |
|---|---|---|
| FEATURE | Product Foundation (for Customer Portal — Epic Map) or Backlog | User or role, need, behavior, scope |
| BUG | Backlog | Observed and expected behavior, reproduction conditions, affected scope, regression check |
| TECHNICAL | Backlog, in particular deferred Foundation decisions | Technical Goal, Scope, Constraints |
| INFRASTRUCTURE | Backlog, in particular deferred Foundation decisions | Technical Goal, Scope, Constraints, environments, impact on startup and rollback |
| SECURITY | Backlog or security findings | Risk or threat, security goal, Scope, Constraints |

Rules:

- Each Story, regardless of type, has verifiable Acceptance Criteria and a Verification Method for each of them.
- For TECHNICAL, INFRASTRUCTURE and SECURITY, there is no need to create an artificial User Persona.
- The type is proposed by CLARIFICATION. The type is recorded in `story-spec`, and the human approves it at HUMAN_SPEC_APPROVAL.
- A change of the type after approval is a change of `story-spec`, that is, a return through the Rework Protocol.

## 17. Cancellation Protocol (OD-13)

`CANCELLED` is a terminal state. A cancelled Story is not reopened. If the work needs to continue, a new Story is created with a reference to the cancelled one.

Procedure:

1. **The human's decision.** Only a human makes CANCEL: at a gate, at a human action, in Recovery or in a decision regarding a Loopback through a gate.
2. **Cancellation Record.** The Orchestrator creates a `cancellation-record` that records:
   - the reason for cancellation;
   - the last state of the Story: stage, Story Status, Execution State;
   - the created artifacts, their versions and statuses;
   - the state of the feature branch: whether it exists, whether there are commits outside the target branch, whether there are uncommitted changes;
   - the presence of an open PR;
   - the human's decisions regarding working resources: what to keep, what to close or delete;
   - decisions regarding approved FOLLOW_UP: which ones become a `backlog-entry`.
3. **Working resources.** The agent does not delete branches, does not close PRs and does not discard uncommitted changes without the human's explicit permission for each action. The permission is recorded in `cancellation-record`.
4. **Transition** to `CANCELLED` and an event in History.

If the Story's changes have already been integrated into the target branch (CANCEL after HUMAN_MERGE), `cancellation-record` explicitly records this. Cancellation does not roll back integrated changes.

## 18. Testing Requirements (OD-15)

The mandatory checks of a Story are defined by three sources:

1. **The project's Testing Strategy** — part of the Project Foundation (FG-01).
2. **The Story Risk Assessment** — a section of `implementation-plan`. It defines the risk categories that the Story touches, based on `solution-design`, `impact-analysis` and the Story Type. It is checked by PLAN_REVIEW and approved by a human at HUMAN_TECHNICAL_APPROVAL.
3. **The Testing Approach** — a section of `implementation-plan` that defines the checks for each Acceptance Criterion and each risk category.

Risky changes receive additional checks. The Workflow Core defines risk categories, not testing technologies:

| Risk category | Example of an additional check |
|---|---|
| Authentication, authorization | Security tests |
| Persistence | Integration tests with the store |
| Concurrency | Concurrent scenario / race tests |
| Infrastructure | Configuration / deployment validation |

The specific kinds, tools and scope of tests are defined by the project's Testing Strategy. VERIFICATION checks that all checks from the three sources have been performed and have passed.
