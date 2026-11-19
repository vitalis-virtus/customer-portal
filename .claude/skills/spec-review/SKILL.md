---
name: spec-review
description: Review Skill of the SPEC_REVIEW stage. Independently, in an isolated context, checks story-spec together with clarification-record, writes a review-report with findings and a verdict, and returns a result envelope. Does not conduct an interview and does not change the checked artifacts. Started only manually with ready input and output paths, record_id and attempt number.
argument-hint: "story=<ID> spec=<path> clarification=<path> output=<path> record_id=review-report/<NNNN> attempt=<N> [source=<path>] [mode=pilot] [previous_reports=<path,...>] [focus=<hypotheses>]"
disable-model-invocation: true
user-invocable: true
context: fork
agent: general-purpose
background: false
allowed-tools: Read, Grep, Glob, Bash(git log:*), Bash(date:*)
disallowed-tools: Edit, NotebookEdit, AskUserQuestion, Agent, Skill
---

# SPEC_REVIEW — Review Skill

## 1. Purpose

Independently check `clarification-record` and `story-spec` before HUMAN_SPEC_APPROVAL. The Skill executes the SPEC_REVIEW stage according to `docs/workflow/stages.md` (section 7) and `docs/workflow/stage-map.yaml` (`reviews: [CLARIFICATION, SPECIFICATION]`). These files and `docs/contracts/` are authoritative; in case of a discrepancy, they apply, not this Skill.

The Skill is executed in a separate context (`context: fork`). Inputs are only the files at the given paths. The working context, chat, reasoning and self-assessment of the producer stage are not an input and not evidence. A `PASS` of the producer stage does not mean that the review has passed. The reviewer does not fix artifacts: it records findings and a verdict (`workflow-policies.md` §4).

**Language** (`AGENTS.md`): `review-report` is written in English. Quotations of the human's decisions from `clarification-record` stay verbatim in their original language.

## 2. Inputs

Arguments (`$ARGUMENTS`):

| Argument | Required | Meaning |
|---|---|---|
| `story` | yes | Story ID, for example `US-001` |
| `spec` | yes | A ready path of the `story-spec` version being checked |
| `clarification` | yes | A ready path of the current version of `clarification-record` |
| `output` | yes | A ready path of the `review-report` file. The Skill does not compute the path itself |
| `record_id` | yes | `review-report/<NNNN>`. The number is not computed; it is taken from the argument |
| `attempt` | yes | The stage attempt number |
| `source` | no | A ready path of the Story source. Without it — `story-source` from the registry |
| `mode=pilot` | no | A pilot outside the registered Workflow |
| `previous_reports` | no | Previous SPEC_REVIEW `review-report` files of this Story |
| `focus` | no | Hypotheses to check. These are not ready findings |

There is no human: the Skill does not ask. If a mandatory argument is missing or an input cannot be read — `BLOCKED` (section 6).

Logical inputs — `docs/contracts/stage-io-map.yaml` → `SPEC_REVIEW`, paths — `docs/contracts/artifact-registry.yaml`:
- `story-spec` and `clarification-record` — both are the subject of the check;
- `story-source` — the Story description;
- `product-foundation` — at minimum: the Business Glossary, Business Rules and NFRs related to the Story; as needed — other Product Foundation documents.

`architecture-foundation` is not an input. Assess the Technical Design boundary by the content of `story-spec` and `clarification-record`. Read only the documents you need (Progressive Context Loading).

## 3. Procedure

1. **Check the arguments.** In `mode=pilot`, `output` cannot lie in `docs/stories/`. If the `output` file already exists — `BLOCKED`, do not overwrite it.
2. **Read the artifacts directly** from `spec` and `clarification`, then the Story and the Foundation. Do not rely on summaries, the Handoff or traceability tables as evidence: check them against the primary text.
3. **Check the previous reports**, if they are passed. Each `OPEN` finding receives a new state. Mark the same problem with the same `F-NNN`. New IDs continue the highest existing number; without previous reports — from `F-001`.
4. **Check `clarification-record`:**
   - a `CQ` decision has an explicit answer of the human, `decided_by` and `decided_at`;
   - an agent's recommendation is not presented as a decision;
   - a deferred question has a described impact;
   - a resolved question is not at the same time listed as open;
   - the Handoff does not silently resolve an open decision;
   - the Story Type is proposed according to `workflow-policies.md` §16.
5. **Check `story-spec`** according to the criteria of section 4.
6. **Check the `focus` hypotheses.** Confirm or reject each one against the documents and explain the conclusion in the report. A hypothesis without confirmation does not become a finding.
7. **Classify the findings** (section 3.1), determine the verdict and the proposed loopback target (section 6).
8. **Write `review-report`** to `output` (section 5) and return the envelope.

### 3.1. Findings

- A finding is only a material problem: it affects the correctness, completeness, verifiability or traceability of the requirements. There are no stylistic findings.
- `severity` — only `BLOCKING` or `NON_BLOCKING` (`artifact-rules.md` §6).
  - `BLOCKING`: a requirement defect that does not allow approving the specification. This includes an invented product decision, an open product question handed over to Technical Design, an unverifiable AC or an AC without a Verification Method, a contradiction, missing mandatory coverage, a violation of a Foundation rule or of the contract metadata.
  - `NON_BLOCKING`: an optional improvement that does not violate requirements and rules.
  - Do not downgrade a violation of an AC or a mandatory Foundation rule. When in doubt whether a Foundation rule is violated — `BLOCKING` (`workflow-policies.md` §7).
- Each finding contains:
  - `F-NNN`;
  - severity;
  - location: file, section, ID (`REQ`/`AC`/`CQ`);
  - explanation;
  - evidence: a quotation or a reference to the source;
  - a specific recommendation of what to change. The reviewer recommends, but does not fix it itself.
- If a finding requires a new or different decision of the human, mark this in the recommendation (the target is CLARIFICATION). Do not choose an option for the human.

## 4. Quality Criteria

Check:

1. **Completeness.** Each `CQ` decision is reflected in a `REQ`/`AC` or marked as not affecting it, with a reason. Nothing from the Handoff is lost.
2. **Consistency with the Foundation.** The specification does not contradict the BR-, NFR- and Glossary rules. It references the rules rather than rewriting them with a change of content.
3. **Traceability.** Each `REQ` and each `AC` has a source: a `CQ` with the human's decision or a Foundation rule ID. The references exist and mean what is claimed.
4. **Verifiability of AC.** Each AC has a condition, an action, an observable result and a Verification Method. The Verification Method states through what to observe and what to check, and the check can actually be performed this way. There are no vague words without a measurable value.
5. **Negative and boundary scenarios.** Invalid input, conflict, a forbidden action, a value at the boundary and beyond it — where the decisions or rules require them.
6. **Internal consistency.** `REQ` and `AC` do not contradict each other. There is no material duplication that gives different expectations for one behavior.
7. **No invented decisions.** There are no requirements without a source. An open question is not silently interpreted. What is deferred has not got into scope. What is marked "open" is really not resolved in `clarification-record` or the Foundation.
8. **The Technical Design boundary.** There are no new technical decisions (routes, error codes beyond the Foundation, schemas, algorithms, libraries). The section "Open for Technical Design" does not contain product decisions on which the observable behavior or an AC depends.
9. **Story Type.** The type is from `story_types`, justified according to `workflow-policies.md` §16. The mandatory content of the type is covered (for FEATURE — the role and the need; for technical types — Technical Goal, Scope, Constraints, without an invented Persona).
10. **Metadata and references** (`artifact-rules.md` §1):
    - the front matter fields correspond to the contract;
    - the version of `clarification-record` in `inputs` matches the passed one;
    - the `commit` of each source matches `git log -1 --format=%h -- <path>`; otherwise the input is `STALE`, and this is `BLOCKING`;
    - `acceptance_criteria` matches the AC IDs in the body;
    - `story_type` matches the body;
    - `revision_reason` is present for `version > 1`;
    - in `mode=pilot`, the pilot line is present.

## 5. Output

One `review-report` file at the `output` path, in English.

Front matter — the fields of `docs/contracts/artifact-rules.md` §2 and only them:
- `record_type: review-report`;
- `record_id` from the argument;
- `story_id`;
- `produced_by: { stage: SPEC_REVIEW, attempt }`;
- `created_at` — the actual UTC from `date -u +%Y-%m-%dT%H:%M:%SZ`;
- `subjects` — `{ type: story-spec, version }` and `{ type: clarification-record, version }` from the front matter of the checked files;
- `verdict`;
- `findings` — `{ id, severity, summary, status }` for each new finding and each previous `OPEN` one.

Body sections:

1. **Subject of the Check** — paths and versions, the Foundation sources read.
2. **Summary** — the verdict, the number of findings by severity, the proposed loopback target.
3. **Findings** — a subsection for each `F-NNN` according to section 3.1.
4. **Previous Findings** — the new state of each `OPEN` finding from previous reports, if any.
5. **`focus` Hypotheses** — confirmed or rejected, with a reference to an `F-NNN` or to the evidence.
6. **Checked Criteria** — briefly, by items 1–10 of section 4: met or `F-NNN`.

In `mode=pilot`, the first line of the body: "Pilot artifact outside the registered Workflow. Not registered, not approved, not an input of any stage." The checked pilot artifacts are not approved either; do not present them and the review as an approval.

## 6. Verdict and Return

SPEC_REVIEW is a mandatory review stage with loopbacks `SPECIFICATION` and `CLARIFICATION`, without `crosses_gates`. Therefore **`PASS`**, **`CHANGES_REQUIRED`** and **`BLOCKED`** are allowed. `NOT_APPLICABLE` is not allowed (`workflow-policies.md` §4).

| Situation | Verdict |
|---|---|
| There are no BLOCKING findings | `PASS` (NON_BLOCKING findings are allowed) |
| There is a BLOCKING finding that requires a new or different decision of the human | `CHANGES_REQUIRED`, `loopback_target: CLARIFICATION` |
| There are BLOCKING findings that can be eliminated only in `story-spec` | `CHANGES_REQUIRED`, `loopback_target: SPECIFICATION` |
| A contradiction with the Foundation that cannot be eliminated by changing the Story artifacts | `BLOCKED` (Foundation Change Required) |
| A decision from the list of escalations is needed (`workflow-policies.md` §2) | `BLOCKED` (Escalation Required) |
| An argument is missing, an input cannot be read or `output` already exists | `BLOCKED` before the check, the file is not written |

`loopback_target` is only a proposal. The route is determined by the Orchestrator from `stage-map.yaml`. For `PASS`, `CHANGES_REQUIRED` and a `BLOCKED` found during the check, the report is written. `outputs: []` — only when there was no check.

Envelope — the format of `docs/contracts/workflow-state.md` §6, unchanged and without additional fields:

```yaml
result:
  story_id: US-001
  stage: SPEC_REVIEW
  attempt: 1
  verdict: CHANGES_REQUIRED          # PASS | CHANGES_REQUIRED | BLOCKED
  outputs:
    - { record: review-report/0001 }
  loopback_target: SPECIFICATION     # only for CHANGES_REQUIRED: SPECIFICATION | CLARIFICATION
  blocker: null                      # for BLOCKED: { reason, details, needed }
  summary: "2 BLOCKING (F-001, F-002), 1 NON_BLOCKING"
```

For `BLOCKED`, describe what is needed in `blocker.needed`; the Skill does not create a `foundation-change-request`. In `mode=pilot`, add the marker "pilot" to `summary`.

## 7. Constraints

- Write only the `output` file. Do not change `story-spec`, `clarification-record`, the Foundation, the Workflow Definition, `docs/contracts/` or other artifacts.
- Do not change `state.yaml`, `history.jsonl`, statuses, Currency or approvals.
- Do not compute the path or record number — take them from the arguments.
- Do not conduct an interview, do not ask the human and do not make product decisions.
- Do not rely on the working context of the producer stage or its self-assessment.
- Do not pass a Human Gate, do not simulate the Orchestrator or the human's decisions; do not start other stages and Skills.
- Do not run `git commit`, `git push` or destructive operations.
