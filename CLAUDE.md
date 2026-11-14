# CLAUDE.md

Guidance for Claude Code in this repository.

## Project

Asset Data Service: a Node.js + TypeScript service that fetches and manages data about
high-voltage power transformers and their related parts.

## Methodology: Spec-Driven Development (GitHub Spec Kit)

The project constitution in `.specify/memory/constitution.md` is binding — read it before
planning or writing code.

Workflow for every feature (skills in `.claude/skills/`):

1. `/speckit-specify <feature description>` → `specs/NNN-feature/spec.md` + feature branch
2. `/speckit-clarify` (optional) → resolve ambiguities in the spec
3. `/speckit-plan <tech choices>` → `plan.md`, `research.md`, `data-model.md`, `contracts/`
4. `/speckit-tasks` → `tasks.md`
5. `/speckit-analyze` (recommended) → consistency check across spec, plan and tasks
6. `/speckit-implement` → execute tasks test-first

Rules:

- Never write production code without an approved spec, plan and tasks.
- Spec Kit scripts are PowerShell: `.specify/scripts/powershell/`.
- Do not edit files under `.specify/templates/` unless intentionally customizing the workflow.
