---
name: impact-analysis
description: Stage Skill of the IMPACT_ANALYSIS stage. Assesses the applicability of the stage by triggers IA-01…IA-05 for the approved story-spec, writes an applicability-decision and, if the stage is applicable, an impact-analysis (affected areas, contracts, data, shared components, other Stories, regression risks), and returns a result envelope. Does not conduct an interview and does not design the solution. Started only manually with ready paths, version, record_id and attempt number.
argument-hint: "story=<ID> spec=<path> output=<path to impact-analysis v<N>.md> version=<N> record_output=<path> record_id=applicability-decision/<NNNN> attempt=<N> [mode=pilot] [previous=<path to v<N-1>>] [reason=\"<entry reasons>\"]"
disable-model-invocation: true
user-invocable: true
---

# IMPACT_ANALYSIS — Stage Skill

## 1. Purpose

Determine the Story's impact on existing behavior, contracts, data, modules and other Stories in progress. The Skill executes the IMPACT_ANALYSIS stage according to `docs/workflow/stages.md` (section 8) and `docs/workflow/stage-map.yaml` (`APPLICABILITY_IMPACT_ANALYSIS`). These files and `docs/contracts/` are authoritative. If they diverge from this Skill, they apply.

The stage is optional (`optional: true`), so the Skill first assesses its applicability and always records this assessment in an `applicability-decision` (`stage-io-map.yaml`, R4). `impact-analysis` is created only when the stage is applicable.

**Stage boundaries:**
- it does not design the solution (SOLUTION_DESIGN), the API (API_DESIGN) or the data (DB_DESIGN) and does not assess the applicability of these stages;
- it does not change requirements: product decisions live only in the approved `story-spec` and `clarification-record`;
- it does not change the code, `story-spec` or other artifacts;
- it is not a Change Impact Assessment, that is, an assessment of the impact of a change of an upstream artifact (`workflow-state-model.md` §8).

The result is finally checked by DESIGN_REVIEW: both `impact-analysis` and the `NOT_APPLICABLE` justification (`reviewed_by`, `na_checked_by`).

**Language** (`AGENTS.md`): both files are written in English. The human's reasons passed in `reason` are quoted verbatim in their original language.

## 2. Inputs

Arguments (`$ARGUMENTS`):

| Argument | Required | Meaning |
|---|---|---|
| `story` | yes | Story ID, for example `US-003` |
| `spec` | yes | A ready path of the current approved version of `story-spec`. Take its version number from the front matter |
| `output` | yes | A ready path of the `impact-analysis` file. The Skill does not compute the path itself |
| `version` | yes | The version of `impact-analysis`. Matches `v<N>` in `output`. Computed by the Orchestrator |
| `record_output` | yes | A ready path of the `applicability-decision` file |
| `record_id` | yes | `applicability-decision/<NNNN>`. The number is taken from the argument |
| `attempt` | yes | The stage attempt number |
| `mode=pilot` | no | A pilot outside the registered Workflow |
| `previous` | for `version > 1` | A ready path of the previous version of `impact-analysis` (`v<N-1>`). It is not changed |
| `reason` | no | The reasons for this entry, one per line: the findings of SOLUTION_DESIGN that returned the Story to this stage, and/or the verbatim reason of the human's `RESUME` decision |

There is no human in the context: the Skill asks nothing. If a mandatory argument is missing or an input cannot be read, the verdict is `BLOCKED` (section 6).

`reason` explains why the stage is executed again: what SOLUTION_DESIGN considers incomplete or wrong in the impact assessment, or why the human rejected the return (for example, that the detected contradiction is not real). This is diagnostics and a decision regarding the Loopback, not a source of requirements: it does not add new requirements.

**Re-entry (D-4).** Each entry is a new applicability assessment based on the current facts and a new `applicability-decision`. Previous conclusions are not carried over automatically (AP-04, AP-08). `previous` and `reason` show what to check particularly carefully, but the conclusion is drawn from `story-spec`, the code and the Foundation. The Skill does not create a `revalidation-record`: for `PASS`, a new version is always created.

Logical inputs — `docs/contracts/stage-io-map.yaml` → `IMPACT_ANALYSIS`, paths — `docs/contracts/artifact-registry.yaml`:
- `story-spec` — requirements, AC, scope, Story Type and the section "Open for Technical Design";
- `codebase` — only the paths the Story may touch (section 3.2);
- `architecture-foundation` — at minimum `docs/architecture/package-map.md` and `architecture.md`, to map the Story to modules and shared infrastructure; other conventions — only as needed;
- the Workflow State of other non-terminal Stories — read-only and only within section 3.3 (`stages.md`, IMPACT_ANALYSIS → "As needed").

`clarification-record`, `story-source` and the Product Foundation are not inputs. Neither is chat history. Read only the documents you need (Progressive Context Loading).

## 3. Procedure

1. **Check the arguments.**
   - In `mode=pilot`, `output` and `record_output` cannot lie in `docs/stories/`.
   - `version` is an integer from `1`. For `version > 1`, `previous` is mandatory, the file exists, and its front matter has `artifact_type: impact-analysis` and `version: <version - 1>`. For `version: 1`, `previous` is not passed. Otherwise — section 6.
   - If the `output` or `record_output` file already exists, the verdict is `BLOCKED`. Do not overwrite the files.
2. **Read `story-spec`**, and also `previous` and `reason`, if they are passed. Identify `story_type`, `REQ`, `AC`, scope and out of scope, and also "Open for Technical Design". Identify also what the Story adds, changes or removes.
3. **Determine the affected areas** (section 3.1) and record the code state (section 3.2).
4. **Check other Stories** (section 3.3).
5. **Evaluate the triggers IA-01…IA-05** (section 3.4) and determine `result`.
6. **If `result: APPLICABLE`**, perform the impact analysis according to section 5.2. Until the verdict is determined, keep it in memory, not in a file.
7. **Determine the verdict** (section 6).
8. **Write the files** in this order:
   - `applicability-decision` to `record_output` (section 5.1), if the applicability assessment is completed;
   - `impact-analysis` to `output` (section 5.2) — **only** for `PASS`.

   For other verdicts, do not create the `impact-analysis` file, not even partially. An unregistered Working Draft will stop the next attempt.
9. **Return the envelope** (section 6).

### 3.1. Affected Areas

- Map `REQ` and `AC` to modules, Platform, the composition root, migrations and public contracts according to `package-map.md` and `architecture.md`. Also to existing external interfaces and data structures.
- An area is a specific path (a file or a directory) or a logical contract: an API operation, a data structure, a shared component.
- For each area, record:
  - whether it exists in the code;
  - whether the Story adds, changes or removes it;
  - a reference to `REQ`/`AC`.
- A non-existent area that the Story only adds is not a change of the existing one. List it anyway.

### 3.2. Code State (D-3)

- Reference only specific paths that were read, never the repository root `.`.
- The source version: `git log -1 --format=%h -- <path>` (`artifact-rules.md` §3, item 4).
- Check uncommitted changes with the command `git status --porcelain -- <path>` for each path read. If there are changes:
  - analyze the actual content of the working tree;
  - mark the path `uncommitted: true` in `basis`;
  - describe the limitation in `impact-analysis`.

  The recorded `commit` does not identify such content.
- A path without a history in Git (new or untracked) cannot be an `inputs` element, because there is no `commit`. Record it only in `basis` with `commit: null`.

### 3.3. Other Stories (D-2)

- Check only **registered** Stories **visible** in the current working tree: `state.yaml` under the registry's `storage_root` (`docs/stories/*/state.yaml`). Skip the current Story.
- Pilot snapshots under `docs/pilots/` are not registered Stories, so do not check them. The exception is the files of the current Story passed in the arguments.
- **Relevant** — only Stories whose `story_status` is not `DONE` and not `CANCELLED`. For each such Story, compare the affected areas (section 3.1) with what it touches:
  - according to its current `impact-analysis`, `solution-design`, `implementation-plan`, `files` from `implementation-changeset`, if they exist;
  - otherwise — according to the scope of its `story-spec`.
- Do not build a registry of all Stories or a search across branches. Record the actual scope in `basis`: the checked path, the IDs and state of the Stories found, the limitation "only Stories of the current branch are visible". If there are no Stories, say so.

### 3.4. Applicability Triggers

The triggers — `stage-map.yaml` → `APPLICABILITY_IMPACT_ANALYSIS`, verbatim:

| ID | Trigger | What to check |
|---|---|---|
| IA-01 | The Story changes or removes existing behavior, an existing external contract or an existing data structure, rather than only adding new ones. | Areas from section 3.1 that exist and are changed or removed |
| IA-02 | The Story changes components, configuration or code used by several modules or shared technical infrastructure. | Changes in Platform, the composition root, `go.mod`, shared configuration |
| IA-03 | The Story requires migration or transformation of data that already exists. | Existing stored data and migrations that the Story affects |
| IA-04 | The Story changes artifacts, contracts or files touched by another non-terminal Story. | Section 3.3 |
| IA-05 | The Story Type is BUG, INFRASTRUCTURE or SECURITY. | `story_type` in `story-spec` |

Rules (`stage-map.yaml`, AP-01…AP-03):

- Each trigger receives `met` or `not_met` with a reference to the source: a file and section or an ID.
- The stage is applicable (`APPLICABLE`) if at least one trigger is `met` (AP-01).
- If the applicability of a trigger is ambiguous (data is missing, an area could not be mapped, an overlap with another Story is unclear), the trigger is considered `met` (AP-02). Record the reason for the ambiguity.
- `NOT_APPLICABLE` — only when all triggers are `not_met`, each with a source (AP-03).

## 4. Quality Criteria

- Each affected area has an impact assessment or an explicit justification of the absence of impact (`stages.md`, Completion Conditions).
- Each trigger has a conclusion and a source. The conclusions do not contradict the list of areas.
- Each `met` trigger is elaborated in `impact-analysis`.
- Each source in `inputs` and `basis` is a specific path with a current `commit`. Uncommitted changes are marked.
- The scope of the check of other Stories and its limitations are recorded.
- There are no technical decisions: structure, routes, schemas, algorithms, libraries. Open technical questions are moved to "Open for Solution Design".
- There are no requirements without a source in `story-spec`. An open question is not silently interpreted.

## 5. Output

Both files are written in English. In `mode=pilot`, the first line of the body of both files: "Pilot artifact outside the official registered Workflow. Not officially approved and not an input of stages of registered Stories; may be an input of subsequent stages of the same pilot." There are no statuses, Currency, approval or pilot marker in the front matter.

### 5.1. `applicability-decision`

Front matter — the common record header and the fields of the type from `docs/contracts/artifact-rules.md` §2, and only them:

```yaml
---
record_type: applicability-decision
record_id: applicability-decision/0001       # from the argument
story_id: US-003
produced_by: { stage: IMPACT_ANALYSIS, attempt: 1 }
created_at: 2026-11-19T18:00:00Z             # date -u +%Y-%m-%dT%H:%M:%SZ
subjects:
  - { type: story-spec, version: 2 }         # the approved version against which applicability was assessed
target_stage: IMPACT_ANALYSIS
kind: ENTRY
result: NOT_APPLICABLE                       # APPLICABLE | NOT_APPLICABLE
triggers:
  IA-01: { status: not_met, ref: "story-spec v2 §Scope; internal/ — the module does not exist yet" }
  IA-02: { status: not_met, ref: "..." }
  IA-03: { status: not_met, ref: "..." }
  IA-04: { status: not_met, ref: "docs/stories/ — there are no non-terminal Stories" }
  IA-05: { status: not_met, ref: "story-spec v2 front matter: story_type FEATURE" }
basis:                                       # basis_facts of the stage
  - { type: story-spec, version: 2 }
  - { fact: story_type, value: FEATURE }
  - { source: codebase, path: main.go, commit: <commit>, uncommitted: false }
  - { source: architecture-foundation, path: docs/architecture/package-map.md, commit: <commit>, uncommitted: false }
  - { fact: other_stories, scope: "docs/stories/*/state.yaml in the current working tree", visible: [], limitation: "only Stories of the current branch are visible; pilot snapshots docs/pilots/ were not checked" }
---
```

The record does not contain `verdict` and `findings`: it is not a review report (`artifact-rules.md` §2).

Body sections:
1. **Subject** — the version of `story-spec`, the code and Foundation paths read.
2. **Affected Areas** — a short list from section 3.1.
3. **Triggers** — for each IA-xx, the conclusion, the justification and the reason for the ambiguity if AP-02 was applied.
4. **Other Stories** — the scope of the check and the limitations (section 3.3).
5. **Entry Reasons** — only if `reason` or `previous` is passed:
   - a verbatim quotation of each line of `reason` and how it was taken into account;
   - if `previous` is passed and `result` is now `NOT_APPLICABLE`: which fact or conclusion changed regarding the `met` triggers of the previous version. Also record that `impact-analysis` `v<N-1>` no longer describes the valid assessment.

### 5.2. `impact-analysis` (only `PASS`)

Front matter — the fields of `docs/contracts/artifact-rules.md` §1, and only them:
- `artifact_type: impact-analysis`, `story_id`, `version`;
- `produced_by: { stage: IMPACT_ANALYSIS, attempt }`;
- `created_at`;
- `inputs`: `{ type: story-spec, version }` and each path read that has a history in Git as `{ source: codebase | architecture-foundation, path, commit }`. For `version > 1` — also `{ type: impact-analysis, version: <N-1> }` (lineage, `artifact-rules.md` §1, §3);
- `revision_reason` — only for `version > 1`: what changed relative to `v<N-1>` and why, with a reference to the lines of `reason`.

Body sections:
1. **Subject** — the version of `story-spec`, the `record_id` of the applicability decision, the `met` triggers.
2. **Affected Areas** — a table: area, whether it exists, whether the Story adds / changes / removes it, `REQ`/`AC`, the impact assessment or the justification that there is no impact.
3. **Behavior and External Contracts** — what of the existing is changed or removed and for whom.
4. **Data and Migrations** — the existing data affected, and whether its migration or transformation is needed. Do not design the schema.
5. **Shared Components** — Platform, the composition root, configuration, module dependencies.
6. **Other Stories** — overlaps or their absence, the scope of the check.
7. **Regression Risks** — what of the existing may break, with a reference to the area.
8. **Analysis Limitations** — uncommitted or untracked paths, ambiguities (AP-02), visibility boundaries.
9. **Open for Solution Design** — questions that the stage deliberately does not resolve.
10. **Entry Reasons** — only if `reason` is passed: a verbatim quotation of each line and how it affected the analysis. A human's decision is recorded in the canonical artifact, not only in History (`workflow-policies.md` §2).

## 6. Verdict and Return

IMPACT_ANALYSIS has `optional: true` and loopbacks `SPECIFICATION` and `CLARIFICATION`, both with `crosses_gates: [HUMAN_SPEC_APPROVAL]`. Therefore **`PASS`**, **`NOT_APPLICABLE`**, **`CHANGES_REQUIRED`** and **`BLOCKED`** are allowed.

| Situation | Verdict | `outputs` |
|---|---|---|
| An argument is missing, `version` and `previous` are not consistent (section 3, step 1), an input cannot be read, `output` or `record_output` already exists | `BLOCKED` before the assessment. No files | `[]` |
| All triggers are `not_met` | `NOT_APPLICABLE` | `[{ record: <record_id> }]` |
| `APPLICABLE`, the analysis is completed, the criteria of section 4 are met | `PASS` | `[{ type: impact-analysis, version }, { record: <record_id> }]` |
| `APPLICABLE`, `story-spec` is infeasible or contradicts the existing system, and this can be eliminated in `story-spec` without a new decision of the human | `CHANGES_REQUIRED`, `loopback_target: SPECIFICATION` | `[{ record: <record_id> }]` |
| `APPLICABLE`, and the elimination requires a new or different product decision of the human | `CHANGES_REQUIRED`, `loopback_target: CLARIFICATION` | `[{ record: <record_id> }]` |
| A contradiction with the Foundation that cannot be eliminated by changing the Story artifacts | `BLOCKED` (Foundation Change Required) | `[{ record }]`, if the assessment is completed, otherwise `[]` |
| A decision from the list of escalations is needed (`workflow-policies.md` §2) | `BLOCKED` (Escalation Required) | the same |
| Information that only a human can provide is missing | `BLOCKED` (Missing Information) | the same |

Rules:

- `CHANGES_REQUIRED` is possible only with `result: APPLICABLE`. If the stage is not applicable, the verdict is `NOT_APPLICABLE`. Design problems are then detected by the subsequent stages SOLUTION_DESIGN, API_DESIGN, DB_DESIGN and DESIGN_REVIEW.
- `loopback_target` is only a proposal. Both targets cross HUMAN_SPEC_APPROVAL. Confirmation, the route and the human's decision belong to the Orchestrator (RR-04). The Skill does not perform or simulate them.
- For `CHANGES_REQUIRED`, findings are passed in `summary`: briefly each problem with a reference to `REQ`/`AC` and the code area. Do not assign `F-NNN` IDs; that is done by review.
- For `BLOCKED`, describe what is needed in `blocker.needed`. The Skill does not create a `foundation-change-request`.
- In `mode=pilot`, add the marker "pilot" to `summary`.

Envelope — the format of `docs/contracts/workflow-state.md` §6, unchanged and without additional fields:

```yaml
result:
  story_id: US-003
  stage: IMPACT_ANALYSIS
  attempt: 1
  verdict: PASS                      # PASS | NOT_APPLICABLE | CHANGES_REQUIRED | BLOCKED
  outputs:
    - { type: impact-analysis, version: 1 }
    - { record: applicability-decision/0001 }
  loopback_target: null              # SPECIFICATION | CLARIFICATION only for CHANGES_REQUIRED
  blocker: null                      # for BLOCKED: { reason, details, needed }
  summary: "APPLICABLE (IA-01, IA-02); 4 areas, 2 regression risks; pilot"
```

## 7. Constraints

- Write only the `record_output` file and, for `PASS`, the `output` file.
- Do not change `story-spec`, `clarification-record`, the code, the Foundation, the Workflow Definition, `docs/contracts/` or other artifacts.
- Do not change the `state.yaml` and `history.jsonl` of any Story, as well as statuses, Currency or approvals.
- Do not compute the path, version or record number: take them from the arguments.
- Bash — only `git log`, `git status`, `git ls-files` and `date`.
- Do not conduct an interview, do not ask the human and do not make product or technical decisions instead of the corresponding stages.
- Do not perform a Loopback, do not pass a Human Gate, do not simulate the Orchestrator, DESIGN_REVIEW or the human's decisions. Do not start other stages, Skills or subagents.
- Do not run `git commit`, `git push` or destructive operations.
