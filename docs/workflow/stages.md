# Stages — Workflow Definition

**Document status:** `APPROVED` (revision 3). Approved by a human on 2026-11-16 after the final Review.

## 1. Purpose

This document explains each stage of the Story Lifecycle: why it exists, what it needs as input, what it creates and when it is considered complete.

Routing rules are not repeated here. The order of stages, `next`, Loopbacks, Human Gate decisions, Optional Stages and terminal states are defined by [stage-map.yaml](stage-map.yaml). If this document and `stage-map.yaml` diverge, `stage-map.yaml` is canonical, and the discrepancy must be reported.

Related documents:

- [workflow-state-model.md](workflow-state-model.md) — Workflow State, Execution State, Stage Verdict, Artifact Status, Recovery, Retry, Staleness, History.
- [workflow-policies.md](workflow-policies.md) — Human Gates, Reviews, Controlled Autonomy, Optional Stages, Findings, Definition of Done, Git.

### Out of Scope

- Artifact schemas, front matter, Markdown structure and file paths. They are defined by the Artifact Contracts stage. Here, artifacts are named only by logical identifiers.
- Stage execution instructions. They are defined by future Stage Skills.
- Technology rules (language, frameworks, stores). They belong to the Project Foundation.

### Provenance Markers

The `APPROVED_DECISION` and `DERIVED` markers have the same meaning as in `stage-map.yaml`: `APPROVED_DECISION` follows directly from Q1–Q28, OD-01…OD-17, approved Review Findings or the final Review decisions FR-01…FR-03; `DERIVED` follows from approved decisions, but was not approved separately. In the approved version there are no `DERIVED` elements (FR-01). The OD decisions are described in [workflow-policies.md](workflow-policies.md#14-od-decisions).

## 2. Context Sources

The Workflow Core references logical sources. For Customer Portal, they are bound as follows:

| Logical source | Customer Portal |
|---|---|
| Product Foundation | [docs/product/](../product/) — Product Vision, Personas, Business Glossary, Business Rules, Non-Functional Requirements, Epic Map |
| Architecture Foundation | [docs/architecture/](../architecture/) — Architecture, Package Map, API, Persistence and Security Conventions |
| Workflow Definition | [docs/workflow/](./) |

Each stage has a **minimum set of sources** (the "Inputs" field). Other documents are loaded only when the stage cannot fulfill its responsibility without them (Progressive Context Loading). Chat history is not an input of any stage: stages obtain context only from canonical artifacts and the Workflow State.

## 3. General Rules for All Stages

These rules apply to every stage, so they are not repeated in the stage descriptions.

**General preconditions:**

- The Story is at this stage according to the Workflow State.
- All input controlled artifacts exist, have Currency `CURRENT` and are not `SUPERSEDED` ([workflow-state-model.md](workflow-state-model.md#5-artifact-status)). The exception is the Revalidation procedure: it may read `STALE` versions to assess the impact of changes, but its result is only a `revalidation-record` or a new version.
- The approvals on which the stage relies are in the `VALID` state.
- For stages after CODE_REVIEW, the Review Binding Guard is performed (`stage-map.yaml` RR-10).
- There are no open Foundation questions marked as blocking for this Story.

**General failure handling:**

- Information that only a human can provide is missing, or a decision from the list of escalations is needed → `BLOCKED` with a reason.
- The Story requires a change to the Product or Architecture Foundation → `BLOCKED` with the reason Foundation Change Required. Then the Foundation Change Request applies ([workflow-policies.md](workflow-policies.md)).
- A problem in an upstream artifact → `CHANGES_REQUIRED` with a proposed Loopback to one of the allowed targets. The stage does not change the upstream artifact itself.
- An interrupted session or a technical failure is not a Stage Verdict. They are handled by the Execution State.

**General completion conditions:**

- The stage returned one Stage Verdict with a justification.
- For `PASS`, all mandatory outputs of the stage are created. New versions of controlled artifacts have recorded dependencies on the versions of upstream artifacts.
- For `NOT_APPLICABLE`, an `applicability-decision` is created according to the applicability condition from `stage-map.yaml`.
- On re-entry after a return, the stage performs Revalidation: it creates a `revalidation-record` for an unchanged version or a new version (Rework Protocol, [workflow-state-model.md](workflow-state-model.md#14-rework-protocol)).

**Review responsibility:** a producer stage does not check itself as the final check. Its result is checked by the stage specified in `reviewed_by` in `stage-map.yaml`. Independence rules — in [workflow-policies.md](workflow-policies.md).

## 4. Stage Overview

| # | Stage ID | Phase | Type | Optional | Provenance |
|---|---|---|---|---|---|
| 1 | CLARIFICATION | CLARIFICATION | producer | no | Q1 |
| 2 | SPECIFICATION | SPECIFICATION | producer | no | Q1 |
| 3 | SPEC_REVIEW | SPECIFICATION | review | no | Q6, Q28, OD-03 |
| 4 | HUMAN_SPEC_APPROVAL | SPECIFICATION | human_gate | — | Q5, OD-06 |
| 5 | IMPACT_ANALYSIS | TECHNICAL_DESIGN | producer | yes | Q26, OD-07 |
| 6 | SOLUTION_DESIGN | TECHNICAL_DESIGN | producer | no | OD-01 |
| 7 | API_DESIGN | TECHNICAL_DESIGN | producer | yes | Q2, OD-07 |
| 8 | DB_DESIGN | TECHNICAL_DESIGN | producer | yes | Q2, OD-07 |
| 9 | DESIGN_REVIEW | TECHNICAL_DESIGN | review | no | Q2, OD-05 |
| 10 | IMPLEMENTATION_PLANNING | IMPLEMENTATION_PLANNING | producer | no | Q1, OD-15 |
| 11 | PLAN_REVIEW | IMPLEMENTATION_PLANNING | review | no | Q6, Q28, OD-04 |
| 12 | HUMAN_TECHNICAL_APPROVAL | IMPLEMENTATION_PLANNING | human_gate | — | Q5, OD-06 |
| 13 | IMPLEMENTATION | IMPLEMENTATION | producer | no | Q1, Q3 |
| 14 | CODE_REVIEW | IMPLEMENTATION | review | no | Q6, Q28, FR-01 |
| 15 | SECURITY_REVIEW | VERIFICATION | review | yes | Q22, OD-05 |
| 16 | RECONCILIATION | VERIFICATION | verification | yes | Q27 |
| 17 | VERIFICATION | VERIFICATION | verification | no | Q1, Q23, OD-15 |
| 18 | PR_PREPARATION | DELIVERY | producer | no | Q8, FR-01 |
| 19 | HUMAN_PR_APPROVAL | DELIVERY | human_gate | — | Q5, OD-06 |
| 20 | MERGE_READINESS_CHECK | DELIVERY | check | no | OD-12, OD-16, FR-01, FR-02 |
| 21 | HUMAN_MERGE | DELIVERY | human_action | — | Q5, Q8, OD-12 |
| 22 | INTEGRATION_CONFIRMATION | DELIVERY | check | no | Q4, Q8, OD-11, OD-12, OD-16, OD-17 |

The `DELIVERY` phase is approved (OD-02): it separates the technical readiness of the Story from the integration of changes.

## 5. Logical Artifacts

Artifacts are named by logical identifiers. Their structure is defined by the Artifact Contracts stage. Artifact classes, Approved Content Identity, Version Status and Currency are explained in [workflow-state-model.md](workflow-state-model.md#5-artifact-status). Supporting artifacts are immutable records: a correction creates a new record.

| Artifact | Class | Created by | Purpose |
|---|---|---|---|
| `clarification-record` | controlled | CLARIFICATION | Questions, answers and decisions of the human regarding the Story |
| `story-spec` | controlled | SPECIFICATION | The Story specification: Story Type, Acceptance Criteria and Verification Method |
| `impact-analysis` | controlled | IMPACT_ANALYSIS | The Story's impact on existing behavior, contracts, data and other Stories |
| `solution-design` | controlled | SOLUTION_DESIGN | The overall technical structure of the solution: Domain, Application, Integration, Infrastructure Design to the extent needed |
| `api-design` | controlled | API_DESIGN | Changes to the external interface |
| `db-design` | controlled | DB_DESIGN | Changes to the schema and structure of stored data |
| `implementation-plan` | controlled | IMPLEMENTATION_PLANNING | Implementation steps, Story Risk Assessment, Testing Approach, planned Optional Stages |
| `implementation-changeset` | controlled | only IMPLEMENTATION | Code and test changes in the feature branch. Identified by the Changeset Content Identity |
| `foundation-change-request` | controlled | any stage through BLOCKED | A request to change the Product or Architecture Foundation |
| `review-report` | supporting | review stages | Findings and verdict regarding specific versions of artifacts and upstream versions |
| `revalidation-record` | supporting | the artifact's owner stage | Confirmation that an unchanged version is current relative to new upstream versions |
| `applicability-decision` | supporting | Optional Stage | Assessment of applicability triggers, in particular the justification of `NOT_APPLICABLE` |
| `reconciliation-report` | supporting | RECONCILIATION | Traceability Acceptance Criteria → design → changeset → tests |
| `verification-report` | supporting | VERIFICATION | Actual check results, Findings Disposition, DoD state, Target Base Identity |
| `pull-request` | supporting | PR_PREPARATION | A PR from the feature branch to the target branch. Identity without metadata |
| `integration-record` | supporting | INTEGRATION_CONFIRMATION | Confirmation of the actual integration of the approved changes |
| `post-merge-validation-record` | supporting | INTEGRATION_CONFIRMATION | The Post-Merge Validation result: the checked state of the target branch, the checks performed, diagnostics in case of failure (OD-16). Not part of the Approval Scope |
| `risk-acceptance-record` | supporting | a human's decision on an escalation | A risk accepted by a human without violating mandatory Foundation rules (WF-V2-01) |
| `foundation-exception-record` | supporting | a human's decision under the Foundation procedure | A controlled exception, only if the Foundation rule explicitly allows it (WF-V2-01) |
| `cancellation-record` | supporting | Orchestrator during CANCEL | The reason and the Story state at the moment of cancellation, decisions regarding resources |
| `backlog-entry` | project-level | INTEGRATION_CONFIRMATION or the Cancellation Protocol | A canonical Backlog entry for an approved FOLLOW_UP with a reference to the Story and the Finding |

## 6. Stages of the CLARIFICATION Phase

### CLARIFICATION

**Purpose.** Eliminate the ambiguities of the Story before specification: understand the intent, boundaries, rules and constraints that apply to it.

**Preconditions.** The Story is registered in the source that corresponds to its type ([workflow-policies.md](workflow-policies.md#16-story-types-od-14)): FEATURE — in the Product Foundation (for Customer Portal — a Candidate Story in the Epic Map) or in the Backlog; other types — in the Backlog. Product dependencies that the Product Foundation marks as having to be resolved before Clarification of this Story are resolved. The prerequisites from [workflow-policies.md](workflow-policies.md#15-prerequisites-and-foundation-gaps) are met.

**Inputs.**
- Minimum: the Story description from its source; Business Glossary; Business Rules and NFRs related to the Story. For TECHNICAL and INFRASTRUCTURE — the corresponding decisions and constraints of the Architecture Foundation.
- As needed: Product Vision, Personas; Architecture Foundation — to identify technical constraints that affect the requirements.
- On re-entry: the previous version of `clarification-record` and the reason for the return.

**Responsibilities.**
- Propose a Story Type and clarify the content required for this type (OD-14). For technical types, do not create an artificial User Persona.
- Formulate questions for the human. Do not invent answers and business requirements.
- Record the human's answers and decisions in the canonical artifact, not only in History.
- Identify contradictions with the Foundation and open questions that block the Story.

**Expected Outputs.** `clarification-record`.

**Completion Conditions.** All questions without which specification is impossible have a human answer or are explicitly marked as a human's decision to defer them, with a defined impact on scope.

**Review Responsibilities.** There is no separate review stage (OD-03). `clarification-record` is checked by SPEC_REVIEW together with the specification.

**Optionality.** Mandatory.

**Failure Handling.** Waiting for the human's answers is a waiting state, not `BLOCKED` ([workflow-state-model.md](workflow-state-model.md#3-execution-state)). A contradiction with the Foundation → `BLOCKED` (Foundation Change Required or clarification of the Foundation by a human).

## 7. Stages of the SPECIFICATION Phase

### SPECIFICATION

**Purpose.** Produce a verifiable specification of the Story.

**Preconditions.** `clarification-record` has a current version.

**Inputs.**
- Minimum: `clarification-record`; the Story description; Business Glossary; related Business Rules and NFRs.
- As needed: other Product Foundation documents.

**Responsibilities.**
- Record the Story Type and the content that is mandatory for it ([workflow-policies.md](workflow-policies.md#16-story-types-od-14)). For technical types — Technical Goal, Scope and Constraints.
- Describe the behavior or technical result, the scope and out-of-scope of the Story.
- Formulate the Acceptance Criteria so that each criterion can be verified, and specify a Verification Method for each.
- Reference Foundation rules rather than rewrite them.
- Do not make technical decisions that belong to Technical Design.

**Expected Outputs.** `story-spec` with the Story Type, Acceptance Criteria and Verification Method.

**Completion Conditions.** Each decision from `clarification-record` is reflected in the specification or explicitly recognized as not affecting it. Each Acceptance Criterion is verifiable and has a Verification Method.

**Review Responsibilities.** SPEC_REVIEW.

**Optionality.** Mandatory.

**Failure Handling.** Missing or contradictory requirements → `CHANGES_REQUIRED` with a Loopback to CLARIFICATION.

### SPEC_REVIEW

**Purpose.** Independently check `clarification-record` and `story-spec` before approval by a human (OD-03). Context isolation — according to risk triggers (OD-10).

**Preconditions.** Current versions of `clarification-record` and `story-spec` exist.

**Inputs.**
- Minimum: `story-spec`; `clarification-record`; the Story description; related Business Rules and NFRs; Business Glossary.
- As needed: other Product Foundation documents.
- Not included: the working context of the producer stage.

**Responsibilities.**
- Check completeness, unambiguity, verifiability of the Acceptance Criteria and the presence of a Verification Method, consistency with the Product Foundation and with the decisions of `clarification-record`.
- Check that the Story Type is justified, and the content of the specification corresponds to this type.
- Check that the specification contains no invented requirements and does not make technical decisions.
- Classify findings ([workflow-policies.md](workflow-policies.md#7-findings-disposition)). Do not fix artifacts.

**Expected Outputs.** `review-report` bound to the checked versions.

**Completion Conditions.** A verdict is reached. `PASS` is possible only without BLOCKING findings.

**Review Responsibilities.** It is itself a Review. The human sees the result at HUMAN_SPEC_APPROVAL.

**Optionality.** Mandatory.

**Failure Handling.** BLOCKING findings → `CHANGES_REQUIRED` with a Loopback to SPECIFICATION or CLARIFICATION.

### HUMAN_SPEC_APPROVAL

**Purpose.** A human approves the Story requirements.

**Preconditions.** SPEC_REVIEW has `PASS` for the current versions of the Approval Scope artifacts.

**Inputs (decision package).** `story-spec`, `clarification-record`, the SPEC_REVIEW `review-report`. If the gate is passed again — a description of the changes relative to the approved identity, the results of the Change Impact Assessment and `revalidation-record`.

**Responsibilities.** The human makes one of the decisions declared in `stage-map.yaml`: APPROVE, RECONFIRM (only under the conditions of OD-06), REQUEST_CHANGES or CANCEL. The decision and its justification are recorded in the Workflow State and History and bound to the Approved Content Identity.

**Expected Outputs.** An approval bound to the Approved Content Identity of `story-spec` and `clarification-record`, or a RECONFIRM of the previous approval.

**Completion Conditions.** The human has made a decision. The agent cannot make it instead of the human.

**Optionality.** Mandatory. Gate rules — [workflow-policies.md](workflow-policies.md#3-human-gates).

**Failure Handling.** As long as there is no decision, the Story waits for the human. There are no timeouts and no automatic decisions.

## 8. Stages of the TECHNICAL_DESIGN Phase

### IMPACT_ANALYSIS

**Purpose.** Determine the Story's impact on existing behavior, contracts, data, modules and other Stories in progress.

**Preconditions.** `story-spec` is approved (HUMAN_SPEC_APPROVAL).

**Inputs.**
- Minimum: `story-spec`; the existing code and artifacts the Story touches.
- As needed: Architecture Foundation; the Workflow State of other active Stories (read-only).

**Responsibilities.** Determine the affected components and contracts, regression risks, dependencies on other Stories, the need to migrate existing data.

**Expected Outputs.** `impact-analysis`.

**Completion Conditions.** Each affected area has an impact assessment or an explicit justification of the absence of impact.

**Review Responsibilities.** DESIGN_REVIEW.

**Optionality.** Optional. Condition — `APPLICABILITY_IMPACT_ANALYSIS` in `stage-map.yaml`.

**Failure Handling.** The specification is infeasible or contradicts the existing system → `CHANGES_REQUIRED` with a Loopback to SPECIFICATION or CLARIFICATION (crosses HUMAN_SPEC_APPROVAL).

**Do not confuse** with the Change Impact Assessment — the procedure for assessing the impact of a change of an upstream artifact on dependent artifacts ([workflow-state-model.md](workflow-state-model.md#8-staleness-and-revalidation)).

### SOLUTION_DESIGN

A universal mandatory Design Stage (OD-01).

**Purpose.** Define the overall technical structure of the Story's solution to the extent it needs:
- **Domain Design** — entities, invariants, placement of business rules;
- **Application Design** — use cases, ports, transaction boundaries, data consistency between stores, error categories;
- **Integration Design** — interaction of modules, external systems and the contracts between them;
- **Infrastructure Design** — configuration, environments, runtime components, startup and rollback, if the Story changes them.

Sections that the Story does not need are marked as not affected with a short justification. The stage is not limited to the Domain and Application levels.

**Preconditions.** `story-spec` is approved. `impact-analysis` is current if IMPACT_ANALYSIS is applicable.

**Inputs.**
- Minimum: `story-spec`; `impact-analysis` (if any); the Architecture Foundation to the extent of the architectural style, modules and dependency rules.
- As needed: other Architecture Foundation conventions; the existing code of the affected modules.

**Responsibilities.** Make technical decisions within the Foundation. Reference Foundation rules rather than repeat them. Specify the analyses that the Foundation requires from Technical Design (for Customer Portal, for example, the Consistency Analysis from the Persistence Conventions). Describe security decisions at the architecture level. Specify new dependencies with a classification according to Risk-Based Dependency Approval (OD-09).

**Expected Outputs.** `solution-design`.

**Completion Conditions.** Each Acceptance Criterion has a place in the design that implements it. The decisions do not violate the Foundation's Architecture Rules.

**Review Responsibilities.** DESIGN_REVIEW.

**Optionality.** Mandatory (OD-01). The scope of the design may be minimal, but the stage does not return `NOT_APPLICABLE`.

**Failure Handling.** General.

### API_DESIGN

**Purpose.** Design the changes to the system's external interface.

**Preconditions.** `solution-design` is current.

**Inputs.**
- Minimum: `story-spec`; `solution-design`; external interface conventions from the Architecture Foundation (for Customer Portal — API Conventions).
- As needed: the Foundation's Security conventions; the current interface contract.

**Responsibilities.** Define the contract of the interface changes according to the Foundation conventions. Do not invent general rules that the Foundation leaves open.

**Expected Outputs.** `api-design`.

**Completion Conditions.** Each interface change is described according to the Foundation conventions. Each change is linked to Acceptance Criteria.

**Review Responsibilities.** DESIGN_REVIEW.

**Optionality.** Optional. Condition — `APPLICABILITY_API_DESIGN`.

**Failure Handling.** General.

### DB_DESIGN

**Purpose.** Design the changes to the schema and structure of stored data.

**Preconditions.** `solution-design` is current. `api-design` is current if API_DESIGN is applicable.

**Inputs.**
- Minimum: `story-spec`; `solution-design`; data storage conventions from the Architecture Foundation (for Customer Portal — Persistence Conventions).
- As needed: `api-design`; `impact-analysis`; the current data schema.

**Responsibilities.** Define the changes to the schema, constraints, indexes and migrations according to the Foundation conventions, in particular for existing data.

**Expected Outputs.** `db-design`.

**Completion Conditions.** Each schema change is justified by a requirement or the design. Migrations are described in accordance with the Foundation conventions.

**Review Responsibilities.** DESIGN_REVIEW.

**Optionality.** Optional. Condition — `APPLICABILITY_DB_DESIGN`.

**Failure Handling.** General.

### DESIGN_REVIEW

**Purpose.** Independently check the entire technical design of the Story as a whole. Context isolation is mandatory (OD-10). The first level of the security check is architecture (OD-05).

**Preconditions.** All applicable stages of the TECHNICAL_DESIGN phase have `PASS`, non-applicable ones — `NOT_APPLICABLE` with a justification.

**Inputs.**
- Minimum: `story-spec`; all applicable design artifacts; the `NOT_APPLICABLE` justifications; the Foundation's Architecture Rules.
- As needed: Foundation conventions that the design concerns; the existing code.
- Not included: the working context of the producer stages.

**Responsibilities.**
- Check the consistency of the design with the specification, the Architecture Foundation and the Architecture Rules.
- Check the consistency of the design artifacts with each other.
- Check the `applicability-decision` for each `NOT_APPLICABLE` in the TECHNICAL_DESIGN phase.
- Check security decisions at the architecture level for every Story, regardless of the applicability of SECURITY_REVIEW.
- Check the classification of new dependencies (OD-09). Check that the design does not violate or weaken mandatory Foundation rules, in particular the Security Rules. Such a violation is BLOCKING, and Risk Acceptance does not eliminate it (RR-14).
- Check the `revalidation-record` for unchanged versions of design artifacts during a repeated pass.
- Classify findings. Do not fix artifacts.

**Expected Outputs.** `review-report`.

**Completion Conditions.** A verdict is reached. `PASS` — without BLOCKING findings.

**Review Responsibilities.** It is itself a Review.

**Optionality.** Mandatory. Does not return `NOT_APPLICABLE`, because SOLUTION_DESIGN is mandatory (OD-01).

**Failure Handling.** BLOCKING findings → `CHANGES_REQUIRED` with a Loopback to the corresponding Design Stage or higher.

## 9. Stages of the IMPLEMENTATION_PLANNING Phase

### IMPLEMENTATION_PLANNING

**Purpose.** Plan the implementation of the approved design.

**Preconditions.** DESIGN_REVIEW has `PASS` for the current versions of the design artifacts.

**Inputs.**
- Minimum: `story-spec`; applicable design artifacts; the DESIGN_REVIEW `review-report`.
- As needed: the project structure from the Architecture Foundation; the existing code.

**Responsibilities.**
- Break the implementation down into steps linked to the Acceptance Criteria and the design.
- Compile a **Story Risk Assessment**: the risk categories that the Story touches (OD-15).
- Define the **Testing Approach** according to the project's Testing Strategy and the Story Risk Assessment, in particular where Test-First is appropriate (Q3) and which additional checks are needed for risky changes.
- Declare the applicability of SECURITY_REVIEW and RECONCILIATION in the form of an `applicability-decision`.
- Identify risks and external actions that will require a human's decision.

**Expected Outputs.** `implementation-plan` (with the Story Risk Assessment and Testing Approach); `applicability-decision` for the planned Optional Stages of the VERIFICATION phase.

**Completion Conditions.** Each Acceptance Criterion is covered by an implementation step and a check. Each risk category has a corresponding check.

**Review Responsibilities.** PLAN_REVIEW.

**Optionality.** Mandatory.

**Failure Handling.** The design is incomplete for planning → `CHANGES_REQUIRED` with a Loopback to the corresponding Design Stage.

### PLAN_REVIEW

**Purpose.** Independently check the `implementation-plan` (OD-04). Context isolation — according to risk triggers (OD-10).

**Preconditions.** A current version of `implementation-plan` exists.

**Inputs.**
- Minimum: `implementation-plan`; `story-spec`; applicable design artifacts.
- Not included: the working context of the producer stage.

**Responsibilities.** Check the coverage of the Acceptance Criteria and the design, the completeness of the Story Risk Assessment, the consistency of the Testing Approach with the project's Testing Strategy and the risk categories, the `applicability-decision` for the planned Optional Stages, the identified risks. Do not fix the plan.

**Expected Outputs.** `review-report`.

**Completion Conditions.** A verdict is reached. `PASS` — without BLOCKING findings.

**Optionality.** Mandatory.

**Failure Handling.** BLOCKING findings → `CHANGES_REQUIRED` with a Loopback to IMPLEMENTATION_PLANNING or higher.

### HUMAN_TECHNICAL_APPROVAL

**Purpose.** A human approves the technical design and the Implementation Plan (Q5).

**Preconditions.** DESIGN_REVIEW and PLAN_REVIEW have `PASS` for the current versions of the Approval Scope artifacts.

**Inputs (decision package).** The artifacts from `approval_scope` in `stage-map.yaml`, the DESIGN_REVIEW and PLAN_REVIEW `review-report`, `applicability-decision`, the declared risks and escalations. During a repeated pass — the changes relative to the approved identity, the results of the Change Impact Assessment and `revalidation-record`.

**Responsibilities.** The human makes a decision: APPROVE, RECONFIRM (only under the conditions of OD-06), REQUEST_CHANGES or CANCEL. The human may require the application of an Optional Stage that the plan marked as non-applicable.

**Expected Outputs.** An approval bound to the Approved Content Identity of the Approval Scope artifacts, or a RECONFIRM of the previous approval.

**Completion Conditions.** The human has made a decision.

**Optionality.** Mandatory.

**Failure Handling.** As for HUMAN_SPEC_APPROVAL.

## 10. Stages of the IMPLEMENTATION Phase

### IMPLEMENTATION

**Purpose.** Implement the approved plan: code and tests (Q3).

**Preconditions.** HUMAN_TECHNICAL_APPROVAL is valid for the current versions of the design and the plan. The work is done in the Story's feature branch.

**Inputs.**
- Minimum: `implementation-plan`; applicable design artifacts; `story-spec` (Acceptance Criteria); the Architecture Foundation rules that the changes concern.
- As needed: other Foundation conventions; the existing code.
- On re-entry: the findings that caused the Loopback.

**Responsibilities.**
- Implement the changes within the approved design. The agent makes technical decisions within these boundaries itself (Controlled Autonomy).
- Write tests as part of the implementation. Apply Test-First where the plan has determined it to be appropriate.
- Do not expand the scope. A deviation from the approved design is a Loopback or an escalation, not a silent change.
- IMPLEMENTATION is the only stage that creates a new Changeset Content Identity (`stage-map.yaml` RR-10). In particular, conflicts with the target branch are resolved here if integration requires changing the content of the changes.
- Record an unplanned dependency in the changeset. Escalate a significant one (OD-09).

**Expected Outputs.** `implementation-changeset` (code and tests in the feature branch) with a new Changeset Content Identity.

**Completion Conditions.** The plan steps are completed. The tests related to the changes are written, and they pass locally.

**Review Responsibilities.** CODE_REVIEW.

**Optionality.** Mandatory.

**Failure Handling.** The approved design or plan cannot be implemented → `CHANGES_REQUIRED` with a Loopback (crosses HUMAN_TECHNICAL_APPROVAL) or `BLOCKED` for escalation.

### CODE_REVIEW

Q6, Q28, FR-01.

**Purpose.** Independently check the implementation. Context isolation is mandatory (OD-10).

**Preconditions.** A current Changeset Content Identity exists. Each new identity requires a new CODE_REVIEW.

**Inputs.**
- Minimum: `implementation-changeset`; `implementation-plan`; applicable design artifacts; the Foundation's Architecture Rules.
- As needed: Foundation conventions that the changes concern.
- Not included: the working context of IMPLEMENTATION.

**Responsibilities.** Check the consistency with the design and the plan, the Architecture Rules, the quality of the tests and their consistency with the Testing Approach, the absence of out-of-scope changes, the classification of new dependencies (OD-09). Record security decisions or findings that require an in-depth check (trigger SEC-10). Do not fix the code.

**Expected Outputs.** `review-report` bound to the Changeset Content Identity.

**Completion Conditions.** A verdict is reached. `PASS` — without BLOCKING findings.

**Optionality.** Mandatory.

**Failure Handling.** BLOCKING findings → `CHANGES_REQUIRED` with a Loopback to IMPLEMENTATION or higher.

## 11. Stages of the VERIFICATION Phase

Distribution of responsibility in the phase (WF-05):

| Stage | What it checks | What it does not do |
|---|---|---|
| SECURITY_REVIEW | In-depth security of the implementation (the second level of OD-05) | Does not check the general code quality, because CODE_REVIEW does this |
| RECONCILIATION | Traceability: the links Acceptance Criteria → design → changeset → tests in both directions | Does not run tests and does not evaluate actual behavior |
| VERIFICATION | Actual behavior: results of tests and checks, basic security checks, Definition of Done | Does not rebuild traceability if there is a `reconciliation-report` |

All three stages bind their results to the current Changeset Content Identity (Review Binding Guard, `stage-map.yaml` RR-10).

### SECURITY_REVIEW

**Purpose.** An in-depth independent check of the security of the implementation (Q22). The second level of the two-level security check (OD-05). The first level is DESIGN_REVIEW. Context isolation is mandatory (OD-10).

**Preconditions.** CODE_REVIEW has `PASS` for the current Changeset Content Identity.

**Inputs.**
- Minimum: `implementation-changeset`; `solution-design` and applicable design artifacts; the DESIGN_REVIEW and CODE_REVIEW `review-report`; the Foundation's Security Rules and Security conventions.
- As needed: `story-spec`, NFRs.
- Not included: the working context of the producer stages.

**Responsibilities.**
- On entry, assess applicability according to `APPLICABILITY_SECURITY_REVIEW` against the current Changeset Content Identity and all facts of `basis_facts`, in particular new findings, human decisions, Recovery and Retry Counters (AP-08). A planned `applicability-decision` may only be expanded. A previous `NOT_APPLICABLE` with a changed Applicability Basis is not used.
- Check the implementation for compliance with the Foundation's Security Rules and the security decisions of the design, identify vulnerabilities and weakenings of security rules. Do not fix artifacts.

**Expected Outputs.** `review-report` bound to the Changeset Content Identity, or an `applicability-decision` with `NOT_APPLICABLE`.

**Completion Conditions.** A verdict is reached. `PASS` — without BLOCKING findings.

**Optionality.** Optional. Condition — `APPLICABILITY_SECURITY_REVIEW`. For changes that meet the condition, it is mandatory. After each new Changeset Content Identity, applicability and the result are assessed anew.

**Failure Handling.** BLOCKING findings → `CHANGES_REQUIRED` with a Loopback to IMPLEMENTATION or higher. A security risk that does not violate mandatory Security Rules → `BLOCKED` (Escalation Required) for possible Risk Acceptance by a human. A violation of a mandatory Security Rule → `BLOCKED` (Foundation Rule Violation): only a fix, a Foundation Change Request or an allowed exception (WF-V2-01).

### RECONCILIATION

**Purpose.** A full check of traceability between the Acceptance Criteria, the design, the implementation and the tests (Q27).

**Preconditions.** CODE_REVIEW has `PASS`, and SECURITY_REVIEW — `PASS` or `NOT_APPLICABLE` for the current Changeset Content Identity.

**Inputs.**
- Minimum: `story-spec`; applicable design artifacts; `implementation-plan`; `implementation-changeset`; review reports.
- As needed: `clarification-record`; the Story History (only to find the source of a gap).

**Responsibilities.** Build and check traceability in both directions:
- each Acceptance Criterion has a place in the design, changes in the changeset and tests or other checks that cover it;
- each change in the changeset has a basis in the requirements or the design.

The stage does not run tests. Whether human decisions are reflected in canonical artifacts is checked by VERIFICATION, not RECONCILIATION.

On entry, the stage assesses applicability according to `APPLICABILITY_RECONCILIATION` and all facts of `basis_facts`: returns through a Human Gate, Foundation Change Request, Repeated Failure, Retry Budget extension, human decisions (AP-08). A previous `NOT_APPLICABLE` with a changed Applicability Basis is not used.

**Expected Outputs.** `reconciliation-report` bound to the Changeset Content Identity and the artifact versions, or an `applicability-decision` with `NOT_APPLICABLE`.

**Completion Conditions.** Each Acceptance Criterion has a traceability chain or a BLOCKING finding. Each change has a basis or a finding.

**Optionality.** Optional. Condition — `APPLICABILITY_RECONCILIATION`.

**Failure Handling.** A traceability gap → `CHANGES_REQUIRED` with a Loopback to the stage where the gap arose.

### VERIFICATION

**Purpose.** An actual check of the Story result: behavior, test results, basic security checks and the Definition of Done.

**Preconditions.** CODE_REVIEW has `PASS`, and SECURITY_REVIEW and RECONCILIATION — `PASS` or `NOT_APPLICABLE` for the current Changeset Content Identity.

**Inputs.**
- Minimum: `story-spec`; `implementation-changeset`; `implementation-plan` (Story Risk Assessment and Testing Approach); the project's Testing Strategy; review reports; `reconciliation-report` (if any); `applicability-decision`; the Currency of controlled artifacts; Target Base Identity.
- As needed: the Foundation's Architecture Rules; design artifacts.

**Responsibilities.**
- Perform the checks from three sources — the project's Testing Strategy, the Story Risk Assessment and the Testing Approach (OD-15) — and record the results. Risky changes receive additional checks ([workflow-policies.md](workflow-policies.md#18-testing-requirements-od-15)).
- Check the integration result relative to the current Target Base Identity (OD-12).
- Perform the **basic check of compliance with requirements**: each Acceptance Criterion has actual evidence of fulfillment according to its Verification Method. The check is always mandatory. If there is a `reconciliation-report`, VERIFICATION uses its traceability and does not build it anew. If RECONCILIATION has `NOT_APPLICABLE`, a direct link "criterion → evidence" is sufficient (Q27).
- Perform basic security checks (Q22) even when SECURITY_REVIEW is not applicable.
- Check the `applicability-decision` for SECURITY_REVIEW and RECONCILIATION, in particular that the Applicability Basis of each `NOT_APPLICABLE` has not changed (AP-08).
- Check that violations of mandatory Foundation rules have not been resolved through Risk Acceptance (WF-V2-01).
- Check the Currency of all controlled artifacts and the binding of review reports to the current versions.
- Check that human decisions from History are reflected in canonical artifacts (DoD 7).
- Produce the Findings Disposition and the Definition of Done state ([workflow-policies.md](workflow-policies.md#8-definition-of-done)).

**Expected Outputs.** `verification-report` bound to the Changeset Content Identity and the Target Base Identity.

**Completion Conditions.** `PASS` only when all agent conditions of the DoD are met, and there are no unresolved BLOCKING findings.

**Optionality.** Mandatory.

**Failure Handling.** Tests do not pass or an Acceptance Criterion has no evidence → `CHANGES_REQUIRED` with a Loopback to IMPLEMENTATION or higher. Violations of Acceptance Criteria cannot be moved to FOLLOW_UP.

## 12. Stages of the DELIVERY Phase

The phase separates the technical readiness of the Story from the integration of changes (OD-02). The check before merge (MERGE_READINESS_CHECK) and the confirmation after merge (INTEGRATION_CONFIRMATION) are different stages (OD-12).

### PR_PREPARATION

Q8, FR-01.

**Purpose.** Prepare the Story's changes for approval by a human: commits in the feature branch and a PR to the target branch.

**Preconditions.** VERIFICATION has `PASS` for the current Changeset Content Identity and Target Base Identity.

**Inputs.** `verification-report`; review reports; `story-spec`; the state of the feature branch and the target branch.

**Responsibilities.**
- Prepare commits and a PR with a description, references to the Story, check results and the Findings Disposition.
- Change only the organization of commits and the PR metadata, but not the content of the changes. Rebuilding commits must not change the Changeset Content Identity (RR-12).
- Do not perform the merge.

**Expected Outputs.** `pull-request` bound to the Changeset Content Identity and the target branch.

**Completion Conditions.** The PR is open, and it contains exactly the Changeset Content Identity that VERIFICATION checked.

**Review Responsibilities.** HUMAN_PR_APPROVAL.

**Optionality.** Mandatory.

**Failure Handling.**
- A change of the content of the changes is needed (in particular a conflict with the target branch) → `CHANGES_REQUIRED` with a Loopback to IMPLEMENTATION. Then CODE_REVIEW and the rest of the checks.
- Only the target branch has changed, the content of the changes is the same → a Loopback to VERIFICATION with the condition `CHANGESET_CONTENT_UNCHANGED`.
- There is no permission or access to the remote repository → `BLOCKED`.

### HUMAN_PR_APPROVAL

**Purpose.** A human approves a specific version of the Story's changes before integration (Q5).

**Preconditions.** A `pull-request` is open for the Changeset Content Identity that VERIFICATION checked.

**Inputs (decision package).** The `approval_scope` artifacts in `stage-map.yaml`: `implementation-changeset`, the CODE_REVIEW and SECURITY_REVIEW (if applicable) `review-report`, `reconciliation-report` (if any), `verification-report` with the Findings Disposition and FOLLOW_UP, `applicability-decision`, `pull-request`. During a repeated pass — what has changed relative to the approved identity.

**Responsibilities.** The human makes a decision: APPROVE, RECONFIRM (only under the conditions of OD-06), REQUEST_CHANGES or CANCEL. The human finally approves the classification of NON_BLOCKING and FOLLOW_UP findings.

**Expected Outputs.** An approval bound to the Approved Content Identity of the Scope artifacts.

**Completion Conditions.** The human has made a decision.

**Optionality.** Mandatory.

**Failure Handling.** As for HUMAN_SPEC_APPROVAL.

### MERGE_READINESS_CHECK

OD-12 (steps 2–5), OD-16, FR-01, FR-02.

**Purpose.** Immediately before the merge, check that the approved changes can still be integrated without a new check.

**Preconditions.** HUMAN_PR_APPROVAL is in the `VALID` state.

**Inputs.** The HUMAN_PR_APPROVAL approval; `pull-request`; the current state of the feature branch and the target branch; `verification-report` (Target Base Identity).

**Responsibilities.**
- Check that the feature branch contains the approved Changeset Content Identity.
- Check that the PR corresponds to the approved identity: the same target branch and the same content of changes.
- Check that the target branch has not changed relative to the Target Base Identity from `verification-report`.
- Record the Merge Protection Status of the target branch — `ACTIVE`, `NOT_SUPPORTED`, `UNAVAILABLE` or `DISABLED` — and act according to `merge_protection_statuses` in `stage-map.yaml` (RR-13, FR-02). Do not change protection settings. Mark the limitation for `NOT_SUPPORTED` in the HUMAN_MERGE package.

**Expected Outputs.** The check result and the Merge Protection Status in History.

**Completion Conditions.** `PASS` only when the first three conditions are met, and the Merge Protection Status is `ACTIVE`, `NOT_SUPPORTED` (under the conditions of FR-02) or `UNAVAILABLE` with an integration order approved by a human. `UNAVAILABLE` without such a decision → `BLOCKED` (Merge Protection Unavailable). `DISABLED` under a mandatory Merge Protection Policy → `BLOCKED` (Merge Protection Disabled), until protection is enabled or as long as there is no valid `foundation-exception-record`.

**Optionality.** Mandatory.

**Failure Handling.** All returns cross HUMAN_PR_APPROVAL and require a human's decision:
- the target branch has changed, the content of the changes is the same → a Loopback to VERIFICATION with the condition `CHANGESET_CONTENT_UNCHANGED`. A new `verification-report` invalidates HUMAN_PR_APPROVAL, so a full APPROVE is needed;
- the PR does not correspond to the approved identity → a Loopback to PR_PREPARATION;
- integration requires changing the content of the changes → a Loopback to IMPLEMENTATION.

### HUMAN_MERGE

**Purpose.** A human integrates the approved changes into the target branch (Q5, Q8). This is a delivery action, not an approval. Only a human performs the merge (OD-12).

**Preconditions.** MERGE_READINESS_CHECK has `PASS`, and HUMAN_PR_APPROVAL is in the `VALID` state.

**Responsibilities.** The human performs the merge using the chosen strategy and confirms it, or returns the Story if the merge is impossible. If a controlled manual procedure applies (`NOT_SUPPORTED` or an approved order for `UNAVAILABLE`), the human performs the merge immediately after MERGE_READINESS_CHECK and, together with CONFIRM_MERGED, confirms that the target branch has not changed after the check. If it has changed — REQUEST_CHANGES to MERGE_READINESS_CHECK. A manual procedure does not bypass mandatory project rules.

**Expected Outputs.** The human's confirmation.

**Completion Conditions.** The human has made a decision from the list in `stage-map.yaml`.

**Failure Handling.** The merge is impossible (a conflict, changes in the target branch) → REQUEST_CHANGES to MERGE_READINESS_CHECK, PR_PREPARATION or IMPLEMENTATION. Then the Rework Protocol applies.

### INTEGRATION_CONFIRMATION

**Purpose.** Confirm by fact that exactly the approved changes have been integrated into the target branch (OD-12, steps 7–8). If the integration was performed on an unchecked base, perform Post-Merge Validation (OD-16). Create a `backlog-entry` for approved FOLLOW_UP (OD-11).

Post-Merge Validation is part of this stage, not a separate stage: it does not require new routes, gates or Loopbacks ([workflow-policies.md](workflow-policies.md#9-git-integration)).

**Preconditions.** The human has confirmed the merge. HUMAN_PR_APPROVAL is valid for the Changeset Content Identity. Changes to the target branch after the merge do not affect it ([workflow-state-model.md](workflow-state-model.md#7-approval-validity)).

**Inputs.** The HUMAN_PR_APPROVAL approval; `pull-request`; `verification-report` (Target Base Identity); the merge result; the current state of the target branch; the Testing Strategy, Story Risk Assessment and Testing Approach (for Post-Merge Validation).

**Responsibilities.** The checks are performed in a mandatory order (`post_merge_validation.check_order` in `stage-map.yaml`):
1. **Integration Match.** The Story's merge result corresponds to the approved Changeset Content Identity regardless of the merge strategy, without relying only on Git SHA. There is no unapproved content. The merge result is evaluated, not a later state of the target branch.
2. **Base Comparison.** The Target Base Identity at the moment of the merge is compared with the checked one. If it matches, Post-Merge Validation is not needed.
3. **Post-Merge Validation** — only if the base has changed. Check the actual integrated result on the current state of the target branch, perform the relevant integration checks and tests, record the result and the checked state in `post-merge-validation-record`.
4. **FOLLOW_UP Backlog.** Create a `backlog-entry` for each approved FOLLOW_UP with a reference to the Story and the Finding Identity.

The agent does not change the target branch: it does not perform merge, revert or push and does not fix integrated code.

**Expected Outputs.** `integration-record`; `post-merge-validation-record` (if it was performed); `backlog-entry` for approved FOLLOW_UP.

**Completion Conditions.** `PASS` only when Integration Match has passed, Post-Merge Validation has passed or is not needed, and all `backlog-entry` items are created. Only then does the Story move to DONE.

**Optionality.** Mandatory. Post-Merge Validation is a conditional part of the stage.

**Failure Handling.**
- The merge has not been performed yet → `CHANGES_REQUIRED` with a Loopback to HUMAN_MERGE.
- The merge introduced the wrong changes or unapproved content → `BLOCKED` (Integration Mismatch). Post-Merge Validation is not performed.
- Post-Merge Validation did not pass → `BLOCKED` (Post-Merge Validation Failed) with diagnostics in `post-merge-validation-record`.
- A technical failure to run the checks → Execution State `FAILED` within Technical Attempts.
- There is no automatic fix of an already performed merge. The decision is made by a human through the Recovery Protocol ([workflow-state-model.md](workflow-state-model.md#9-blocked-and-recovery-protocol)).
- After an external fix and `RESUME` (OD-17): for Post-Merge Validation Failed — `PASS` only with the original Integration Match `PASS` and a repeated Post-Merge Validation `PASS`; for Integration Mismatch — `PASS` only with proof of the proper integration of the approved Changeset Content Identity of this Story. Such proof is a new merge after a correction (`ROUTE_BACK` → HUMAN_MERGE → a new CONFIRM_MERGED). Integration Match always evaluates the merge recorded by the last CONFIRM_MERGED. The success of another Story by itself does not give `PASS`.
