# Artifact Contracts — Simplified MVP

**Status:** `APPROVED (MVP)` — Phase 4, Simplified Artifact Contracts MVP. The human architect approved the design on 2026-11-17.

The package defines where Story artifacts live, what metadata they have, how the state and human decisions are stored, and what each of the 22 stages reads and creates. The goal is a practical foundation for the first real Story. The package does not implement Stage Skills, the Orchestrator or validation.

**Boundaries of the approval.** Only the design of the MVP contracts is approved, that is, the five files of this package. The approval does not mean that the extended guarantees of the Workflow Definition are implemented. What is deferred and conservatively simplified is listed in the section "Deferred Advanced Features / Workflow Compatibility". The Workflow Definition remains in force in full: no Story can reach `DONE` by bypassing its mandatory requirements.

The package replaced the previous, overly complex edition of Phase 4 (JSON Schema, catalog, fixtures, Durable Transition Protocol, Git identity profiles). That edition was never approved as Phase 4 and is not canonical. Its full copy is kept outside the repository together with a SHA-256 manifest.

## Files

| File | What it contains |
|---|---|
| [artifact-registry.yaml](artifact-registry.yaml) | The single source of paths: artifact types, categories, owner stages, the gate where the type is approved |
| [artifact-rules.md](artifact-rules.md) | Metadata of documents and records, versions, Staleness, findings, who writes what |
| [workflow-state.md](workflow-state.md) | `state.yaml`, `history.jsonl`, approvals, the stage result envelope |
| [stage-io-map.yaml](stage-io-map.yaml) | Inputs and outputs of each of the 22 stages |

Routes, verdicts, loopbacks, gates and the Approval Scope are defined only by the approved Workflow Definition ([stage-map.yaml](../workflow/stage-map.yaml)). The package does not repeat or change them.

## How It Works

1. A stage takes the input paths from the registry, reads the current versions, writes a new file (`v<N>.md` or a record `<NNNN>.md`) and returns a result envelope.
2. The Orchestrator checks the result, registers the files in `state.yaml`, chooses the route from `stage-map.yaml` and appends an event to `history.jsonl`.
3. At a Human Gate, the Orchestrator stops, shows the human the decision package and records their decision: who, what, when, which versions.
4. Statuses (Version Status, Currency, Approval State) live only in `state.yaml`. Artifact files do not change after registration.

## Storage

MVP placement:

```
docs/stories/<story_id>/
├── state.yaml
├── history.jsonl
├── artifacts/<artifact_type>/v<N>.md
└── records/<record_type>/<NNNN>.md
docs/backlog/<entry_id>.md          # project level
```

Story data is committed to this Story's feature branch together with the code. Git provides the change history and an immutable trail.

This is a **proposed MVP simplification**, not an approved replacement of the isolated storage root policy (decision D-01 of the old package). Limitations:

- **Branch switching.** The Story state is visible only in this Story's feature branch. On another branch, the Story files are absent or different. The Orchestrator works with a Story only on its branch.
- **Several Stories at once.** Each Story has its own branch and directory. Parallel work — through separate branches or `git worktree`. A shared overview of all Stories from one branch is not available.
- **Story data gets into the PR.** Code review does not take it into account (`workflow-state.md`, section 8).
- **The state after merge is a known unresolved limitation.** After HUMAN_MERGE, the Story state updates (`state.yaml`, `history.jsonl`, `integration-record`, `post-merge-validation-record`, the transition to `DONE` or `BLOCKED`) appear after the feature branch has already been integrated. The current MVP does not define how these updates get into and are stored in the target branch. This is a task for the future Orchestrator implementation. The Story structure does not change because of this.
- **Commits.** [AGENTS.md](../../AGENTS.md) prohibits commits without explicit permission. Who commits the Story data (a human or an agent with permission) must be decided before the first Story.

## Deferred Advanced Features / Workflow Compatibility

The MVP does not implement all rules of the approved Workflow Definition. No Human Gate is removed or weakened: every gate requires a human's decision.

### Fully Supported

- All 22 stages, routes and gates — according to the unchanged `stage-map.yaml`.
- Artifact owners, immutable versions, a new version for a correction, dependencies on specific versions.
- Staleness of controlled artifacts, Revalidation through `revalidation-record`.
- Review PASS ≠ Human Approval. A human's decision records who, what, when and which versions.
- APPROVE, REQUEST_CHANGES, CANCEL, CONFIRM_MERGED. Confirmation of a Loopback through a gate, Recovery actions.
- Optional Stages with `applicability-decision`, `NOT_APPLICABLE` only for `optional: true`.
- Retry Counters (Technical Attempts, Quality Loops, Repeated Failure), Cancellation Protocol, Step Mode and Auto Mode.

### Supported Conservatively or Manually

| Requirement | MVP | Consequence |
|---|---|---|
| Approved Content Identity | The document version, the record's `record_id`, `changeset_commit` for the changeset | No hash. Integrity relies on the immutability of files and on Git |
| Review Binding Guard (RR-10) | Review is bound to the changeset version. After `changeset_commit`, only Story files may be changed | Stricter: any rebase or squash requires a new IMPLEMENTATION and review |
| `CHANGESET_CONTENT_UNCHANGED` | `changeset_commit` has not changed | Stricter |
| Foundation Staleness | Dependency on the git commit of a file or directory | Coarse: any change of a file makes the dependent versions `STALE` |
| Applicability Basis (AP-04, AP-08) | `basis` is recorded in the decision, re-evaluation — on every re-entry | Stricter, but more work |
| Finding Identity, Repeated Failure | `F-NNN` is assigned by the reviewer, a match is determined by the LLM | Less deterministic |
| Out-of-Stage Revalidation | Not supported separately. The human chooses `ROUTE_BACK` | Stricter and more expensive |
| Session Recovery, integrity | A discrepancy between `state.yaml`, History and files → `BLOCKED` (Integrity Error), resolved by a human | No automatic recovery |
| MERGE_READINESS_CHECK | Comparison of commits: between `changeset_commit` and the PR head only Story files have changed, target branch = `verified_target_base` (`workflow-state.md`, section 8) | Sufficient for the typical case |
| SUSPENDED → VALID | `revalidation-record` returns Currency `CURRENT`, but the approval remains `SUSPENDED` until the human makes APPROVE | Stricter: a full APPROVE instead of RECONFIRM |

### Deferred. An Approved Simplification of the Workflow Is Needed

| Capability | What the Workflow requires | What the MVP does |
|---|---|---|
| **RECONFIRM** | Gate decision RECONFIRM (OD-06) | Not offered. The human chooses APPROVE. A restriction of the human's choice, so it requires approval |
| **Cryptographic Changeset Content Identity** | Rebuilding commits without changing the content does not change the identity (`workflow-policies.md`, section 9) | Identity = commit. A discrepancy with the wording of section 9 |
| **Full Git Integration Verification** | Proof that the merge introduced exactly the approved changes and nothing else (Integration Match), and Post-Merge Validation (OD-12 step 7, OD-16, OD-17) | The Artifact Contracts **do not implement** this check. They set only the minimal Git metadata: `feature_branch`, `base_commit`, `changeset_commit`, `verified_target_base`, `merge_commit`, `target_before_merge`. Without an approved verification method, INTEGRATION_CONFIRMATION does not return `PASS`, and the Story remains in `BLOCKED` until a human's decision |
| **Durable Transition / Crash Recovery** | Deterministic recovery of the consistency of State and History | None. A discrepancy is resolved by a human |
| **Advanced contract validation** | Deterministic contract validation, versioned contracts, version pinning | No validator and no version pinning. The rules are performed by the Orchestrator and review according to instructions |

**DONE.** The MVP does not mark a Story as `DONE` until the integration requirements of the approved Workflow are met: Integration Match `PASS`, Post-Merge Validation `PASS` (if the target branch has changed) and all DoD conditions (`stage-map.yaml` RR-09, `terminal_states.DONE`). The presence of an `integration-record` or a human's attestation does not prove this by itself.

### Prerequisites for the First Story (Independent of This Package)

- **FG-01** — Testing Strategy. Without it, the first real Story is not registered (`workflow-policies.md`, section 15).
- **PRQ-02** — Merge Protection Configuration before the first HUMAN_MERGE.
- **PRQ-01** — the canonical Backlog. The registry defines the path `docs/backlog/`, but approval is needed that this is the canonical Backlog.

## What Was Borrowed from the Mentor's Implementation

- The registry as the single source of paths and owners.
- Stage I/O through logical keys rather than paths.
- Result envelope: the Skill does not change the state.
- A simple YAML state and a JSONL log.

Differences from the mentor: a separate file for each version (the mentor has one path per type), statuses only in the state (the mentor has them in front matter), data separately for each Story, preservation of who made a decision.
