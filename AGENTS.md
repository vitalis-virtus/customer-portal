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

- The Harness is developed incrementally. The full Workflow is not implemented yet.
- Pilot execution is allowed through the Story Orchestrator (`.claude/skills/story-orchestrator/`):
  - start it only on an explicit command of the user, with `mode=pilot`, outside `docs/stories/`;
  - once started, it invokes Stage Skills and subagents according to its own rules; no separate user command is needed for each invocation;
  - it supports only its implemented stages and commands; anything else remains unsupported. Registration of real Stories is not supported.
- Pilot Human Gate decisions are simulations, not official Workflow approvals. Only the human makes them, explicitly, through the Orchestrator command `decide`. Never invent, assume or apply a Human Gate decision on your own.
- Create infrastructure components incrementally and only on a separate task.
- Do not add references to documents or components that do not exist yet.

## Language

- Communication with the user and explanations — in Ukrainian.
- Documentation and instruction files — in English.
- Generated Story artifacts — in English. Content provided by a human (answers, decisions, comments, reasons) is quoted verbatim in its original language.
