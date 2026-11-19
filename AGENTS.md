# AGENTS.md

Minimal bootstrap instructions for the AI agent (Claude Code) in this repository.

## About the Project

- **Customer Portal** — an educational backend project in Go.
- **Primary educational goal** — build an Artifact Driven Agentic SDLC.
- **Current phase** — setting up the Agentic Infrastructure, not implementing business functionality.

## Agent Behavior

- Do not invent business requirements or important architectural decisions. If a requirement or an important architectural decision is not defined, first check the existing documentation. If there is no answer, explain the uncertainty and ask the user for a decision.
- Do not create components or documents that the current task does not require.
- Do not change files unrelated to the current task.
- Do not run `git commit`, `git push` or destructive operations without explicit permission.
- Explain non-trivial decisions and report uncertainties.

## Infrastructure Development Mode

- The workflow has not been implemented yet.
- Do not try to run or simulate the Story Orchestrator, skills or Human Gates.
  - Exception (Phase 6): it is allowed to develop the Skill `.claude/skills/story-orchestrator/` and to run it only on an explicit user command with `mode=pilot`, outside `docs/stories/`. This does not cover Human Gates.
- Create infrastructure components incrementally and only on a separate task.
- Do not add references to documents or components that do not exist yet.

## Language

- Communication with the user and explanations — in Ukrainian.
- Documentation and instruction files — in English.
- Generated Story artifacts — in English. Content provided by a human (answers, decisions, comments, reasons) is quoted verbatim in its original language.
- Until a Stage Skill is migrated to English, the output language defined in that Stage Skill applies.
