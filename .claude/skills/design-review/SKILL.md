---
name: design-review
description: Review Skill of the DESIGN_REVIEW stage. Independently, in an isolated context, checks the whole technical design of a Story (solution-design and, if applicable, impact-analysis, api-design, db-design) against story-spec, the Architecture Foundation and each other, checks the applicability-decisions of IMPACT_ANALYSIS, API_DESIGN and DB_DESIGN, writes a review-report with findings and a verdict, and returns a result envelope. Does not change the checked artifacts. Started only manually with ready paths, record_id and attempt number.
argument-hint: "story=<ID> spec=<path> design=<path> [impact=<path>] [api=<path>] [db=<path>] ia_decision=<path> api_decision=<path> db_decision=<path> output=<path> record_id=review-report/<NNNN> attempt=<N> [mode=pilot] [previous_reports=<path,...>]"
disable-model-invocation: true
user-invocable: true
context: fork
agent: general-purpose
background: false
allowed-tools: Read, Grep, Glob, Write, Bash(git log:*), Bash(git status:*), Bash(date:*)
disallowed-tools: Edit, NotebookEdit, AskUserQuestion, Agent, Skill
---

# DESIGN_REVIEW — Review Skill

## 1. Purpose

Independently check the entire technical design of the Story as a whole. The Skill executes the DESIGN_REVIEW stage according to `docs/workflow/stages.md` (section 8) and `docs/workflow/stage-map.yaml` (`reviews: [IMPACT_ANALYSIS, SOLUTION_DESIGN, API_DESIGN, DB_DESIGN]`, `context_isolation: required`). These files and `docs/contracts/` are authoritative. If they diverge from this Skill, they apply.

The Skill is executed in a separate context (`context: fork`, OD-10). Inputs are only the files at the given paths and the Foundation sources. The working context, reasoning and self-assessment of the producer stages are not an input and not evidence. A `PASS` of a producer stage does not mean that the review has passed. The reviewer does not fix artifacts: it records findings and a verdict (`workflow-policies.md` §4).

**Boundaries:** the Skill does not change the checked artifacts, `story-spec`, the Foundation or the code; does not write code, SQL or migrations; does not make product or architectural decisions; does not approve dependencies, risks or Human Gates; does not perform Loopbacks; does not change State or History. `PASS` is not a Human Approval: the technical design is approved only by a human at HUMAN_TECHNICAL_APPROVAL, after IMPLEMENTATION_PLANNING and PLAN_REVIEW (`artifact-rules.md` §5).

**Language** (`AGENTS.md`): `review-report` is written in English. Quotations of the human's decisions stay verbatim in their original language.

## 2. Inputs

Arguments (`$ARGUMENTS`):

| Argument | Required | Meaning |
|---|---|---|
| `story` | yes | Story ID, for example `US-001` |
| `spec` | yes | A ready path of the current approved version of `story-spec` |
| `design` | yes | A ready path of the current version of `solution-design` |
| `impact`, `api`, `db` | no | Ready paths of the current versions of `impact-analysis`, `api-design`, `db-design`. Each is passed only when its stage is applicable. If one is absent, its stage is not applicable: do not search for or use the artifact, even if its files exist |
| `ia_decision`, `api_decision`, `db_decision` | yes | Ready paths of the valid `applicability-decision` records of IMPACT_ANALYSIS, API_DESIGN and DB_DESIGN (`kind: ENTRY`) |
| `output` | yes | A ready path of the `review-report` file. The Skill does not compute the path itself |
| `record_id` | yes | `review-report/<NNNN>`. The number is taken from the argument |
| `attempt` | yes | The stage attempt number |
| `mode=pilot` | no | A pilot outside the registered Workflow |
| `previous_reports` | no | Previous DESIGN_REVIEW `review-report` files of this Story |

There is no human: the Skill does not ask. If a mandatory argument is missing or an input cannot be read — `BLOCKED` (section 6).

Logical inputs — `docs/contracts/stage-io-map.yaml` → `DESIGN_REVIEW`:
- `story-spec` — `REQ`, `AC`, scope, "Open for Technical Design";
- the design artifacts: `solution-design`; `impact-analysis`, `api-design`, `db-design` when applicable;
- `applicability-decision` — the decisions of IMPACT_ANALYSIS, API_DESIGN and DB_DESIGN (the `na_checked_by` check);
- `architecture-foundation` — at minimum the Architecture Rules (`docs/architecture/architecture.md` §8); as needed — the conventions the design concerns (`package-map.md`, `api-conventions.md`, `persistence-conventions.md`, `security-conventions.md`), only the needed sections;
- as needed — the existing code that the design concerns, specific paths.

`clarification-record`, the Product Foundation (except the rules `story-spec` references) and chat history are not inputs. A `revalidation-record` is not checked in this increment: the Orchestrator does not create it. Read only the documents you need (Progressive Context Loading).

## 3. Procedure

1. **Check the arguments.** In `mode=pilot`, `output` cannot lie in `docs/stories/`. If the `output` file already exists — `BLOCKED`, do not overwrite it.
2. **Read the artifacts directly** from the given paths, then the Foundation. Do not rely on summaries, handoff or traceability tables as evidence: check them against the primary text.
3. **Check the previous reports**, if they are passed. Each `OPEN` finding receives a new state. Mark the same problem with the same `F-NNN`. New IDs continue the highest existing number; without previous reports — from `F-001`.
4. **Check the design** according to the criteria of section 4.
5. **Classify the findings** (section 3.1), determine the verdict and the proposed loopback target (section 6).
6. **Write `review-report`** to `output` (section 5) and return the envelope.

### 3.1. Findings

- A finding is only a material problem: it affects the correctness, completeness, consistency, security or traceability of the design. There are no stylistic findings.
- `severity` — only `BLOCKING` or `NON_BLOCKING` (`artifact-rules.md` §6).
  - `BLOCKING`: a defect that does not allow approving the design. This includes a violation of an AC, of an Architecture Rule (AR-) or of another mandatory Foundation rule, in particular the Security Rules (RR-14); a contradiction between design artifacts or with `story-spec`; an AC without a place in the design; an invented requirement; an `OPEN` or `DEFERRED` decision resolved beyond the authority of the Story; a wrong or unjustified `NOT_APPLICABLE`; a wrong dependency classification (OD-09); a violation of the contract metadata.
  - `NON_BLOCKING`: an improvement that does not violate requirements and rules.
  - Do not downgrade a violation of an AC or a mandatory Foundation rule. When in doubt whether a Foundation rule is violated — `BLOCKING` (`workflow-policies.md` §7).
- Each finding contains: `F-NNN`; severity; location (file, section, ID); explanation; evidence (a quotation or a reference to the source); a specific recommendation and the stage that must fix it. The reviewer recommends, but does not fix it itself.
- If a finding requires a new or different decision of the human, mark this in the recommendation (the target is CLARIFICATION). Do not choose an option for the human.

## 4. Quality Criteria

Check:

1. **Consistency with `story-spec`.** Each `AC` has a place in `solution-design`; each interface change of `api-design` and each schema change of `db-design` is justified by a `REQ`/`AC` or a design decision; the referenced `REQ`/`AC` exist. There are no requirements without a source, and nothing out of scope got into the design.
2. **Consistency between the design artifacts.** `api-design` implements the use cases, error categories and security decisions of `solution-design`; `db-design` implements its data placement and required guarantees by persistence-level mechanisms (DB-D03); `impact-analysis` areas and risks are handled in `solution-design`; the handoff sections ("Open for Solution Design", "Open for API and DB Design", "Open for DB Design and Implementation") are taken into account. The artifacts do not contradict each other.
3. **Architecture Foundation.** The design does not violate the Architecture Rules (AR-), module boundaries and import directions (Package Map), and the conventions it concerns. It references rules rather than rewriting them with a change of content. No new technology outside Architecture §6 (AR-012).
4. **Security at the architecture level** (OD-05) — for every Story: authentication, authorization and "who is acting", trust boundaries, sensitive data in responses, logs and storage, sessions and credentials. Mandatory Security Rules are not weakened (RR-14).
5. **Dependencies** (OD-09). Each new or changed dependency has a correct classification. A significant or ambiguous dependency must not be presented as approved: a human's approval is required (`workflow-policies.md` §2).
6. **Open and deferred Foundation decisions.** A decision is resolved in the design only where the Foundation delegates it to the Technical Design of this Story (for example, the transactionality of the first migration, `persistence-conventions.md` §7, §13); otherwise it is escalated. An `OPEN` product question is applied only from a human decision in `story-spec`.
7. **Applicability decisions** (`na_checked_by`, AP-05). For each of the three records: `target_stage`, `kind: ENTRY`; the triggers of `stage-map.yaml` → `applicability_conditions`, each with a status and a source; `result` ⇔ the triggers (AP-01, AP-03); `subjects` and `basis` are the current versions required by `basis_facts`. For `NOT_APPLICABLE`: each `not_met` is supported by the source it references, and the `not_applicable_evidence` is present. `APPLICABLE` ⇔ the corresponding artifact is passed.
8. **Metadata and references** (`artifact-rules.md` §1, §2): the front matter fields of each artifact correspond to the contract; the `inputs` versions match the passed artifacts; a conditional artifact is in `inputs` only when its stage is applicable; the `commit` of each source matches `git log -1 --format=%h -- <path>` (otherwise the input is `STALE`, and this is `BLOCKING`); no source references the repository root `.`; in `mode=pilot`, the pilot line is present.

## 5. Output

One `review-report` file at the `output` path, in English.

Front matter — the fields of `docs/contracts/artifact-rules.md` §2 and only them:
- `record_type: review-report`; `record_id` from the argument; `story_id`;
- `produced_by: { stage: DESIGN_REVIEW, attempt }`;
- `created_at` — the actual UTC from `date -u +%Y-%m-%dT%H:%M:%SZ`;
- `subjects` — `{ type, version }` of each checked design artifact (from their front matter) and `{ record: <record_id> }` of each checked `applicability-decision`;
- `verdict`;
- `findings` — `{ id, severity, summary, status }` for each new finding and each previous `OPEN` one.

`story-spec` is the reference of the check, not its subject: it is not in `subjects`.

Body sections:

1. **Subject of the Check** — paths and versions, the applicability decisions, the Foundation sources and code paths read.
2. **Summary** — the verdict, the number of findings by severity, the proposed loopback target.
3. **Findings** — a subsection for each `F-NNN` according to section 3.1.
4. **Previous Findings** — the new state of each `OPEN` finding from previous reports, if any.
5. **Checked Criteria** — briefly, by items 1–8 of section 4: met or `F-NNN`.

In `mode=pilot`, the first line of the body: "Pilot artifact outside the official registered Workflow. Not officially approved and not an input of stages of registered Stories; may be an input of subsequent stages of the same pilot." The checked pilot artifacts are not approved either; do not present them and the review as an approval.

## 6. Verdict and Return

DESIGN_REVIEW is a mandatory review stage (`optional: false`) with loopbacks `DB_DESIGN`, `API_DESIGN`, `SOLUTION_DESIGN`, `IMPACT_ANALYSIS` (without `crosses_gates`) and `SPECIFICATION`, `CLARIFICATION` (with `crosses_gates: [HUMAN_SPEC_APPROVAL]`). Therefore **`PASS`**, **`CHANGES_REQUIRED`** and **`BLOCKED`** are allowed. `NOT_APPLICABLE` is not allowed (OD-01, `workflow-policies.md` §4).

| Situation | Verdict |
|---|---|
| There are no `OPEN` `BLOCKING` findings | `PASS` (`NON_BLOCKING` findings are allowed) |
| `BLOCKING` findings that can be eliminated in the design artifacts or upstream | `CHANGES_REQUIRED`, `loopback_target` — the stage that must fix them (section 3.1), or higher |
| A `BLOCKING` finding requires a new or different decision of the human | `CHANGES_REQUIRED`, `loopback_target: CLARIFICATION` |
| The design requires a change to the Foundation that cannot be eliminated by changing the Story artifacts | `BLOCKED` (Foundation Change Required) |
| A decision from the list of escalations is needed (`workflow-policies.md` §2), in particular the approval of a significant dependency (OD-09) | `BLOCKED` (Escalation Required) |
| An argument is missing, an input cannot be read or `output` already exists | `BLOCKED` before the check, the file is not written |

Rules:

- If findings point to several targets, choose the highest upstream one: CLARIFICATION, SPECIFICATION, IMPACT_ANALYSIS, SOLUTION_DESIGN, API_DESIGN, DB_DESIGN. After the return, the Story goes through the subsequent stages again (RR-06).
- `loopback_target` is only a proposal. The route and the human's confirmation for targets with `crosses_gates` (RR-04) belong to the Orchestrator. The Skill does not perform or simulate them.
- For `PASS`, `CHANGES_REQUIRED` and a `BLOCKED` found during the check, the report is written: `outputs: [{ record: <record_id> }]`. `outputs: []` — only when there was no check.
- For `BLOCKED`, describe what is needed in `blocker.needed`. The Skill does not create a `foundation-change-request`.
- In `mode=pilot`, add the marker "pilot" to `summary`.

Envelope — the format of `docs/contracts/workflow-state.md` §6, unchanged and without additional fields:

```yaml
result:
  story_id: US-001
  stage: DESIGN_REVIEW
  attempt: 1
  verdict: CHANGES_REQUIRED          # PASS | CHANGES_REQUIRED | BLOCKED
  outputs:
    - { record: review-report/0003 }
  loopback_target: DB_DESIGN         # only for CHANGES_REQUIRED: DB_DESIGN | API_DESIGN | SOLUTION_DESIGN | IMPACT_ANALYSIS | SPECIFICATION | CLARIFICATION
  blocker: null                      # for BLOCKED: { reason, details, needed }
  summary: "1 BLOCKING (F-001), 2 NON_BLOCKING; pilot"
```

## 7. Constraints

- Write only the `output` file. Do not change the checked artifacts, `story-spec`, the applicability records, the Foundation, the Workflow Definition, `docs/contracts/`, the code or other files.
- Do not change `state.yaml`, `history.jsonl`, statuses, Currency or approvals.
- Do not compute the path or record number — take them from the arguments.
- Do not conduct an interview, do not ask the human and do not make product or architectural decisions.
- Do not rely on the working context of the producer stages or their self-assessment.
- Do not pass a Human Gate, do not approve dependencies or risks, do not simulate the Orchestrator or the human's decisions; do not start other stages, Skills or subagents.
- Bash — only `git log`, `git status` and `date`. Do not run `git commit`, `git push` or destructive operations.
