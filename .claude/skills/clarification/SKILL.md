---
name: clarification
description: Stage Skill of the CLARIFICATION stage. Interactively clarifies a Candidate Story with the human, records their decisions in clarification-record and returns a result envelope. Started only manually with a ready output path, version and attempt number.
argument-hint: "story=<ID> output=<path to v<N>.md> version=<N> attempt=<N> [source=<path>] [mode=pilot] [previous=<path>] [reason=<return reason>]"
disable-model-invocation: true
user-invocable: true
---

# CLARIFICATION — Stage Skill

## 1. Purpose

Eliminate the ambiguities of the Story before SPECIFICATION: intent, boundaries, rules and constraints. The Skill executes the CLARIFICATION stage according to `docs/workflow/stages.md` (section 6) and `docs/workflow/stage-map.yaml`. These files and `docs/contracts/` are authoritative; in case of a discrepancy, they apply, not this Skill.

The Skill works in the main session, because it conducts an interview with the human.

**Language** (`AGENTS.md`):
- the interview with the human — questions, context, options, recommendations, the «Вирішено людиною» / «Відкрито» blocks — is in Ukrainian;
- `clarification-record` is written in English;
- the human's answers and decisions are recorded as verbatim quotations in their original language. Do not translate or paraphrase them.

## 2. Inputs

Arguments (`$ARGUMENTS`):

| Argument | Required | Meaning |
|---|---|---|
| `story` | yes | Story ID, for example `US-001` |
| `output` | yes | A ready path of the `clarification-record` file. The Skill does not compute the path and version itself |
| `version`, `attempt` | yes | The version number (matches `v<N>` in `output`) and the stage attempt number |
| `source` | no | A ready path of the Story source. Without it — `story-source` from the registry |
| `mode=pilot` | no | A pilot outside the registered Workflow |
| `previous`, `reason` | for `version > 1` | The previous version of `clarification-record` and the reason for the return |

If a mandatory argument is missing, ask the human. Do not guess.

Logical inputs — `docs/contracts/stage-io-map.yaml` → `CLARIFICATION`, paths — `docs/contracts/artifact-registry.yaml`:
- `story-source` — the Story description from its source (for FEATURE — `docs/product/epic-map.md`);
- `product-foundation` — at minimum: the Business Glossary, Business Rules and NFRs related to the Story; as needed — Product Vision, Personas;
- `architecture-foundation` — mandatory for TECHNICAL and INFRASTRUCTURE; for other types — only to identify technical constraints that affect the requirements.

Chat history is not an input. Read only the documents you need (Progressive Context Loading).

## 3. Procedure

1. **Check the arguments.** In `mode=pilot`, `output` cannot lie in `docs/stories/`. If the `output` file already exists, do not overwrite it without the human's permission.
2. **Read the Story** from the source. Check the preconditions of `stages.md` §6: if the Product Foundation marks a dependency that must be resolved before Clarification of this Story (for example, Open Product Dependencies in the Epic Map), and it is not resolved — `BLOCKED` without an interview.
3. **Propose a Story Type** from `story_types` in `stage-map.yaml` according to the table in `docs/workflow/workflow-policies.md` §16 (Story source and content). The type is only proposed: SPECIFICATION records it, the human approves it at HUMAN_SPEC_APPROVAL. For technical types, do not invent a User Persona.
4. **Read the Foundation** and sort what you find into:
   - **established facts** — approved rules and decisions with IDs (BR-, NFR-, etc.);
   - **unresolved questions** — open dependencies, "recommendations", undefined items. These are not facts.
5. **Find gaps**: ambiguities, contradictions, missing requirements needed for the mandatory specification content of this type (`workflow-policies.md` §16).
6. **Sort the questions.** For the interview — only those without which specification is impossible. Non-critical ones go to the risks section or the handoff. Do not ask about what is already decided in the Foundation.
   **The Clarification boundary:** a question that affects the observable behavior of the product or the Acceptance Criteria is resolved with the human before `PASS`. Only purely technical implementation details are passed on.
7. **Conduct the interview** (section 3.1). Repeat rounds until each critical question has an answer or an explicit decision of the human to defer it.
8. **A contradiction with the Foundation** → `BLOCKED` (section 6). Do not fix the Foundation and do not bypass it.
9. **Final consistency check** before `PASS`:
   - the human's decisions and the derived requirements do not contradict each other;
   - a question marked as resolved is not at the same time listed as open or as a gap;
   - the Handoff does not silently resolve an open decision of the human;
   - recommendations are not presented as the human's decisions.
   A detected inconsistency → return to the interview.
10. **Write `clarification-record`** to `output` (section 5) and return the envelope.

### 3.1. Interview

- A round — 1–4 related questions. For each:
  - an ID `CQ-NN`, stable between rounds and versions;
  - context: what is ambiguous and where this is visible (a reference to a document or rule);
  - options, if any;
  - a recommendation with a justification, separately marked as a recommendation.
- In each round, show two blocks: «Вирішено людиною» and «Відкрито».
- A decision is only an explicit answer of the human to a specific question. Silence or a general "ok" does not approve a recommendation; clarify which `CQ` it refers to.
- Do not choose the recommended option for the human.
- The human may explicitly defer a question. Then ask about, or propose for confirmation, the impact: what drops out of scope, what SPECIFICATION and the subsequent stages inherit.
- Ask once for the identifier of the decision maker (`decided_by`) if it is not provided.
- Take the decision time from `date -u +%Y-%m-%dT%H:%M:%SZ`, not from examples.

## 4. Quality Criteria

- Every statement about a requirement has a source: a Foundation document or a `CQ` with the human's decision.
- No invented requirement, business rule or architectural decision.
- An agent's recommendation is nowhere presented as the human's decision.
- Every critical question has an answer or a decision to defer it with a described impact.
- The mandatory content for the proposed type is covered or explicitly marked as a gap.
- The human's answers and decisions are quoted verbatim in their original language.

## 5. Output

One `clarification-record` file at the `output` path, in English.

Front matter — **only** the fields of `docs/contracts/artifact-rules.md` §1: `artifact_type: clarification-record`, `story_id`, `version`, `produced_by: { stage: CLARIFICATION, attempt }`, `created_at` (the actual UTC), `inputs`. For a source — `{ source, path, commit }`, commit — `git log -1 --format=%h -- <path>`. For `version > 1` — also `revision_reason` and the previous version in `inputs`. There is no status, Story Type or pilot marker in the front matter.

Body sections:

1. **Candidate Story** — ID, source, description.
2. **Proposed Story Type** — the type and the justification.
3. **Scope Understanding** — in scope / out of scope.
4. **Established Facts** — with references to the Foundation.
5. **Questions and Decisions** — for each `CQ`: context, options, the agent's recommendation, the human's decision (a verbatim quotation in its original language), `decided_by`, `decided_at`.
6. **Deferred Questions** — the `CQ`, who deferred it and when, the impact on scope and subsequent stages.
7. **Contradictions and Risks**.
8. **Handoff for SPECIFICATION** — requirements and decisions with references to a `CQ` or the Foundation.

In `mode=pilot`, the first line of the body: "Pilot artifact outside the official registered Workflow. Not officially approved and not an input of stages of registered Stories; may be an input of subsequent stages of the same pilot."

## 6. Verdict and Return

CLARIFICATION has `optional: false` and `loopbacks: []`, so only **`PASS`** and **`BLOCKED`** are allowed. `CHANGES_REQUIRED` and `NOT_APPLICABLE` are not allowed for this stage.

| Situation | Action |
|---|---|
| Waiting for the human's answer | Not a verdict and not `BLOCKED`. Continue the interview, do not return an envelope |
| All critical questions are resolved or explicitly deferred, the file is written | `PASS` |
| An unresolved Product Foundation precondition | `BLOCKED` |
| A contradiction with the Foundation | `BLOCKED`: Foundation Change Required or clarification of the Foundation by a human |
| A decision from the list of escalations is needed (`workflow-policies.md` §2) | `BLOCKED` (Escalation Required) |

Envelope — the format of `docs/contracts/workflow-state.md` §6, unchanged:

```yaml
result:
  story_id: US-001
  stage: CLARIFICATION
  attempt: 1
  verdict: PASS                      # PASS | BLOCKED
  outputs:
    - { type: clarification-record, version: 1 }
  loopback_target: null
  blocker: null                      # for BLOCKED: { reason, details, needed }
  summary: "4 questions resolved, 1 deferred (CQ-03)"
```

For `BLOCKED`: `outputs: []`, and the human's decisions already obtained go into `blocker.details`. The Skill does not create a `foundation-change-request`: describe the needed Foundation change in `blocker.needed`. In `mode=pilot`, add the marker "pilot" to `summary`.

## 7. Constraints

- Do not change `state.yaml`, `history.jsonl`, statuses, Currency or approvals.
- Write only the `output` file. Do not change the Foundation, the Workflow Definition, `docs/contracts/` or other artifacts.
- Do not compute the version number, record number or path — take them from the arguments.
- Do not pass a Human Gate and do not simulate the decisions of the human or the Orchestrator.
- Do not start other stages and Skills.
- Do not run `git commit`, `git push` or destructive operations.
