# Artifact Rules

**Status:** `APPROVED (MVP)`, Simplified Artifact Contracts MVP. Approved on 2026-11-17 as the design of the MVP contracts; boundaries of the approval — [README.md](README.md).

This document defines the common metadata of artifacts and the rules for working with versions. Paths and owners are defined by [artifact-registry.yaml](artifact-registry.yaml), and the storage of statuses by [workflow-state.md](workflow-state.md).

## 1. Controlled Documents

Each version is a separate Markdown file `artifacts/<artifact_type>/v<N>.md` with YAML front matter.

```yaml
---
artifact_type: story-spec          # key from artifact-registry.yaml
story_id: US-001
version: 2                         # an integer from 1
produced_by:
  stage: SPECIFICATION             # the stage from produced_by in the registry
  attempt: 1                       # the stage attempt number
created_at: 2026-11-18T10:15:00Z   # the actual time, not an example
inputs:                            # everything this version is based on
  - { type: clarification-record, version: 1 }
  - { source: product-foundation, path: docs/product/business-rules.md, commit: c1565b1 }
revision_reason: "F-002: clarified AC-03"   # only for version > 1
---
```

| Field | Rule |
|---|---|
| `artifact_type` | A key from `artifacts` of the registry |
| `story_id` | Matches `story_id` in `state.yaml` |
| `version` | `1` for the first version, then +1. Matches `v<N>` in the file name |
| `produced_by` | The stage specified as the owner of the type in the registry |
| `created_at` | ISO-8601 UTC, the time the version was created |
| `inputs` | A direct dependency on a **specific** version of an artifact (`type` + `version`), a record (`record: review-report/0002`) or a source (`source` + `path` + `commit`) |
| `revision_reason` | Mandatory for `version > 1`: what changed and why. References to findings, if any |

Type-specific fields are added only where the Workflow depends on them:

- `story-spec`: `story_type` (one of `story_types` in `stage-map.yaml`) and `acceptance_criteria: [AC-01, AC-02]`;
- `implementation-changeset`: `feature_branch`, `base_commit`, `changeset_commit` and `files` (a list of changed paths).

The front matter has **no** status, Currency, approval or review conclusions. All of this belongs to `state.yaml`.

## 2. Supporting Records

Each record is a separate Markdown file `records/<record_type>/<NNNN>.md` with a common header.

```yaml
---
record_type: review-report
record_id: review-report/0002      # <record_type>/<NNNN>, the number is the next one for this type in the Story
story_id: US-001
produced_by: { stage: SPEC_REVIEW, attempt: 1 }
created_at: 2026-11-18T11:00:00Z
subjects:                          # what exactly the record checks, describes or confirms
  - { type: story-spec, version: 2 }
  - { type: clarification-record, version: 1 }
verdict: CHANGES_REQUIRED          # for records that have a verdict
findings:
  - { id: F-002, severity: BLOCKING, summary: "AC-03 has no Verification Method", status: OPEN }
---
```

| Field | Rule |
|---|---|
| `record_id` | A stable reference to the record from other files and `state.yaml` |
| `subjects` | The exact versions to which the record belongs. A record regarding one version does not count for another |
| `verdict` | For review and check records: one of `stage_verdicts` in `stage-map.yaml` |
| `findings` | Only in review and verification reports (see section 6) |

Additional fields by type:

| Type | Fields |
|---|---|
| `revalidation-record` | `subjects`: the version that remains current. `upstream`: the new upstream versions. `rationale` |
| `applicability-decision` | `target_stage`, `kind: PLANNED \| ENTRY`, `result: APPLICABLE \| NOT_APPLICABLE`, `triggers` (ID → `met`/`not_met` with a reference to the source), `basis` (the versions and facts on which the decision is based) |
| `verification-report` | `verified_changeset: { version, changeset_commit }`, `verified_target_base` (commit), `dod` (the state of the DoD conditions), `follow_up` |
| `pull-request` | `repository`, `number`, `source_branch`, `target_branch`, `changeset: { version, changeset_commit }` |
| `integration-record` | `merge_commit`, `target_before_merge`, `checks` and results. The record records facts, but does not by itself prove integration (README.md, "Full Git Integration Verification") |
| `cancellation-record` | The fields from the Cancellation Protocol list (`workflow-policies.md`, section 17) |
| `risk-acceptance-record`, `foundation-exception-record` | `decided_by`, `decided_at`, the `finding` or rule, the boundaries of the acceptance |

Records are immutable. To correct a record, a new one is created with `corrects: <record_id>`.

## 3. Versions and Immutability

1. **A draft before registration.** The stage writes the version file, but it becomes canonical only when the Orchestrator registers it in `state.yaml`. An unregistered file is a Working Draft: it is not an input of another stage, and a repeated attempt of the same stage may overwrite it.
2. **A published version is immutable.** After registration, the file is not edited, including when its status changes.
3. **A correction is a new version.** Any change of content gives `v<N+1>` with `revision_reason`. The previous version becomes `SUPERSEDED`.
4. **A source version** is the git commit of the file's last change at the moment the version is created (`git log -1 --format=%h -- <path>`). A reference to a directory (`docs/architecture/`) is allowed, but then any change in the directory makes the dependent version `STALE`.
5. **Identity of approved content** (Approved Content Identity) in the MVP:
   - for a controlled document — its `version`;
   - for a record — `record_id`;
   - for `implementation-changeset` — `version` together with `changeset_commit`;
   - for `pull-request` — `record_id`, that is, the repository, number, branches and changeset version (PR metadata is not included).

## 4. Staleness and Repeated Checks

Version Status and Currency are defined in `workflow-state-model.md`, section 5. Their values are stored only in `state.yaml`.

**Staleness rule.** The current version X has Currency `STALE` if, for at least one of its `inputs`, one of two conditions holds:

- the current version of the upstream artifact is greater than the recorded one, and there is no `revalidation-record` confirming X relative to this new version;
- for a source, there are new commits after the recorded `commit` that changed `path`.

Otherwise, Currency is `CURRENT`.

Rules:

- `STALE` and `SUPERSEDED` versions are not valid inputs of a stage. They may be read only for Revalidation.
- When the upstream has changed, the owner stage of the dependent artifact does one of two things on re-entry: creates a `revalidation-record` (the change does not affect it) or a new version (the change affects it).
- An indirect impact is handled by a chain: a new version of B makes C `STALE`, even if B depended on A.
- A review report created against versions other than the current ones does not count as `PASS`.

## 5. Review and Human Approval

- **Review PASS is not Human Approval.** A review report opens the way to a Human Gate, but the gate is passed only by a human's decision recorded in `state.yaml` → `approvals` ([workflow-state.md](workflow-state.md), section 4).
- A producer does not check itself as the final check. Its artifact is checked by the stage from `reviewed_by` in `stage-map.yaml`.
- A Human Approval is bound to **all** corresponding elements of the Approval Scope (`approval_scope` of the gate in `stage-map.yaml`): the exact versions of documents and the `record_id` of supporting records. A new version of a document or the replacement of an approved record with a new record of the same purpose makes the approval `INVALIDATED` ([workflow-state.md](workflow-state.md), section 4).
- An approval does not become valid for changed content without a new human decision.

## 6. Findings

- A finding has an ID `F-NNN`, stable within the Story. The same problem found again (in particular by another stage) is marked with the same ID. The match is determined by the reviewer.
- `severity`: `BLOCKING` or `NON_BLOCKING`.
- `status`: `OPEN`, `RESOLVED` (confirmed by an independent review or verification) or `DISPOSITIONED` (a human's decision or an approved Disposition, in particular FOLLOW_UP).
- The next report of the same review stage regarding the same artifact lists all previous `OPEN` findings and their new state. An open finding does not disappear silently.
- A violation of an Acceptance Criterion, a mandatory Foundation rule or a DoD condition remains `BLOCKING` (RR-14). A decision at a gate does not accept it.

## 7. Who Writes What

| What | Who writes | What is not allowed |
|---|---|---|
| Version and record files of its own type | The owner stage from the registry | Writing files of another type, changing registered files |
| `state.yaml`, `history.jsonl` | Only the Orchestrator | A Stage Skill does not change the state directly |
| `risk-acceptance-record`, `foundation-exception-record`, `cancellation-record` | The Orchestrator, only by a human's decision | The agent does not make these decisions itself |
| `docs/contracts/` | A human | The agent does not change or weaken the contracts to pass a stage; such an obstacle gives `BLOCKED` |
