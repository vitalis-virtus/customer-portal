---
name: specification
description: Stage Skill of the SPECIFICATION stage. Turns clarification-record into a verifiable story-spec (Story Type, requirements, Acceptance Criteria with a Verification Method, scope) and returns a result envelope. Does not conduct an interview. Started only manually with ready input and output paths, version and attempt number.
argument-hint: "story=<ID> clarification=<path> output=<path to v<N>.md> version=<N> attempt=<N> [source=<path>] [mode=pilot] [previous=<path>] [reason=<return reason>]"
disable-model-invocation: true
user-invocable: true
---

# SPECIFICATION — Stage Skill

## 1. Purpose

Produce a verifiable Story specification from the clarified requirements. The Skill executes the SPECIFICATION stage according to `docs/workflow/stages.md` (section 7) and `docs/workflow/stage-map.yaml`. These files and `docs/contracts/` are authoritative; in case of a discrepancy, they apply, not this Skill.

There is no interview. Product decisions live only in `clarification-record`. If a decision is missing, this is a return to CLARIFICATION (section 6), not a question to the human.

**Language** (`AGENTS.md`): `story-spec` is written in English. The human's decisions quoted from `clarification-record` stay verbatim in their original language. Anything addressed to the human is in Ukrainian.

## 2. Inputs

Arguments (`$ARGUMENTS`):

| Argument | Required | Meaning |
|---|---|---|
| `story` | yes | Story ID, for example `US-001` |
| `clarification` | yes | A ready path of the current version of `clarification-record`. Take its version number from the front matter of this file |
| `output` | yes | A ready path of the `story-spec` file. The Skill does not compute the path and version itself |
| `version`, `attempt` | yes | The version of `story-spec` (matches `v<N>` in `output`, does not depend on the version of `clarification-record`) and the stage attempt number |
| `source` | no | A ready path of the Story source. Without it — `story-source` from the registry |
| `mode=pilot` | no | A pilot outside the registered Workflow |
| `previous`, `reason` | for `version > 1` | The previous version of `story-spec` and the reason for the return |

Ask the human only about a missing mandatory argument. Do not guess.

Logical inputs — `docs/contracts/stage-io-map.yaml` → `SPECIFICATION`, paths — `docs/contracts/artifact-registry.yaml`:
- `clarification-record` — the human's decisions (`CQ`), established facts, risks, Handoff;
- `story-source` — the Story description;
- `product-foundation` — at minimum: the Business Glossary, Business Rules and NFRs related to the Story; as needed — other Product Foundation documents.

`architecture-foundation` is not an input of this stage. Technical facts of the Foundation get into the specification only through `clarification-record`, with a reference to it. Chat history is not an input. Read only the documents you need (Progressive Context Loading).

## 3. Procedure

1. **Check the arguments.** In `mode=pilot`, `output` cannot lie in `docs/stories/`. If the `output` file already exists, do not overwrite it without the human's permission.
2. **Read `clarification-record`, the Story and the Foundation.** Identify: `CQ` decisions with `decided_by`; deferred questions with their impact; established facts with IDs (BR-, NFR-); risks; the Handoff. An agent's recommendation without the human's decision is not a requirement.
3. **Record the Story Type** from `story_types` in `stage-map.yaml` according to the table in `docs/workflow/workflow-policies.md` §16. The starting point is the CLARIFICATION proposal. Justify a divergence from it. For technical types, do not invent a User Persona.
4. **Formulate the requirements** `REQ-NN`: an observable behavior or a technical result, each with a source (a `CQ-NN` or a Foundation rule ID). Reference Foundation rules rather than rewrite them. Cover the mandatory content of the type (§16).
5. **Define the scope and out of scope.** Deferred `CQ` and what is explicitly excluded in `clarification-record` are out of scope, with the inherited impact.
6. **Formulate the Acceptance Criteria** `AC-NN` (section 3.1).
7. **Check the Clarification boundary.** If a requirement cannot be formulated without a new product decision, or the decisions contradict each other or the Foundation — stop and return a verdict according to section 6. Do not choose an option for the human and do not record "assumptions".
8. **Check the Technical Design boundary.** Technical matters already established by the Foundation and recorded in `clarification-record` — only a reference. New ones (routes, error codes beyond the Foundation, schemas, algorithms, code structure, libraries) are not decided — move them to the section "Open for Technical Design".
9. **Check the quality** (section 4). Fix a nonconformity; if it comes from `clarification-record` — step 7.
10. **Write `story-spec`** to `output` (section 5) and return the envelope.

### 3.1. Acceptance Criteria

- Each AC: an ID `AC-NN`, a reference to a `REQ` and a source, condition → action → expected observable result, the kind (positive / negative / boundary), a Verification Method.
- Cover positive scenarios, failures (invalid input, conflict, a forbidden action) and boundaries (a value at the boundary and beyond it). The list of topics for AC in the Handoff of `clarification-record` is a minimum, not a ceiling.
- One AC checks one behavior. No words like "correctly", "fast", "conveniently" without a measurable value.
- The Verification Method describes how to observe the result: through which interface or action and what exactly is checked (the response, the stored state, the absence of data in responses and logs). Do not choose tools and test levels — they are defined by the Testing Strategy (FG-01) and the Testing Approach of `implementation-plan`.
- The IDs `AC-NN` and `REQ-NN` are stable between versions. Do not reuse a removed ID.

## 4. Quality Criteria

- Each requirement and AC has a source: the Foundation or a `CQ` with the human's decision. Nothing invented.
- Each `CQ` decision is reflected in a `REQ`/`AC` or explicitly marked as not affecting the specification, with a reason.
- Each AC is specific, verifiable and has a Verification Method; important negative and boundary scenarios are covered.
- The requirements do not contradict each other, the Foundation and `clarification-record`.
- The scope boundaries are unambiguous; what is deferred has not got into scope.
- No open product question is silently interpreted; a recommendation is not presented as a decision.
- There are no technical decisions that belong to Technical Design.

## 5. Output

One `story-spec` file at the `output` path, in English.

Front matter — the fields of `docs/contracts/artifact-rules.md` §1 and only them: `artifact_type: story-spec`, `story_id`, `version`, `produced_by: { stage: SPECIFICATION, attempt }`, `created_at` (the actual UTC from `date -u +%Y-%m-%dT%H:%M:%SZ`), `inputs`, `story_type`, `acceptance_criteria: [AC-01, …]`. In `inputs` — `{ type: clarification-record, version }` and each source read as `{ source, path, commit }`, commit — `git log -1 --format=%h -- <path>`. For `version > 1` — `revision_reason` and the previous version of `story-spec` in `inputs`. There are no statuses, approval or pilot marker in the front matter.

Body sections:

1. **Story** — ID, source, the Story Type with a justification; the mandatory content of the type (for FEATURE — the role and the need).
2. **Scope** — in scope / out of scope, with sources.
3. **Requirements** — `REQ-NN` with sources.
4. **Acceptance Criteria** — `AC-NN` according to section 3.1.
5. **Decision Traceability** — a table `CQ` → `REQ`/`AC` or "does not affect: reason".
6. **Risks and Constraints** — references to the risks of `clarification-record` and the Foundation gaps that apply to the Story.
7. **Open for Technical Design** — technical questions that the specification deliberately does not resolve.

In `mode=pilot`, the first line of the body: "Pilot artifact outside the official registered Workflow. Not officially approved and not an input of stages of registered Stories; may be an input of subsequent stages of the same pilot." The pilot `clarification-record` is not registered either; do not present either of them as approved.

## 6. Verdict and Return

SPECIFICATION has `optional: false` and a single loopback target — `CLARIFICATION` (without `crosses_gates`), so **`PASS`**, **`CHANGES_REQUIRED`** and **`BLOCKED`** are allowed. `NOT_APPLICABLE` is not allowed.

| Situation | Verdict |
|---|---|
| The specification is written, the criteria of section 4 are met | `PASS`, `outputs: [{ type: story-spec, version }]` |
| A product decision is missing, the requirements are incomplete or contradict each other | `CHANGES_REQUIRED`, `loopback_target: CLARIFICATION`, `outputs: []` |
| A contradiction with the Foundation | `BLOCKED` (Foundation Change Required) |
| A decision from the list of escalations is needed (`workflow-policies.md` §2) | `BLOCKED` (Escalation Required) |

Envelope — the format of `docs/contracts/workflow-state.md` §6, unchanged and without additional fields:

```yaml
result:
  story_id: US-001
  stage: SPECIFICATION
  attempt: 1
  verdict: PASS                      # PASS | CHANGES_REQUIRED | BLOCKED
  outputs:
    - { type: story-spec, version: 1 }
  loopback_target: null              # CLARIFICATION only for CHANGES_REQUIRED
  blocker: null                      # for BLOCKED: { reason, details, needed }
  summary: "Type FEATURE, 9 REQ, 18 AC; all CQ reflected"
```

For `CHANGES_REQUIRED`, do not write the file. In `summary`, briefly list the gaps with references to `CQ` or rules, and show the details to the human in the chat (in Ukrainian). Do not assign `F-NNN` IDs — that is done by review. For `BLOCKED`: `outputs: []`; the Skill does not create a `foundation-change-request` — describe the needed change in `blocker.needed`. In `mode=pilot`, add the marker "pilot" to `summary`. On re-entry, the Skill always creates a new version with `revision_reason`; this Draft does not create a `revalidation-record`.

## 7. Constraints

- Do not change `state.yaml`, `history.jsonl`, statuses, Currency or approvals.
- Write only the `output` file. Do not change `clarification-record`, the Foundation, the Workflow Definition, `docs/contracts/` or other artifacts.
- Do not compute the version number or path — take them from the arguments.
- Do not conduct an interview and do not make product decisions instead of the human.
- Do not pass a Human Gate, do not simulate SPEC_REVIEW, the Orchestrator or the human's decisions; do not start other stages and Skills.
- Do not run `git commit`, `git push` or destructive operations.
