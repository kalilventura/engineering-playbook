# Skills/Commands Restructuring — Design

## Problem

The playbook currently stores every rule as a one-line markdown file under
topic directories (`architecture/`, `clean-code/`, `go/`, `testing/`,
`messaging/`, `refactoring/`, `governance/`, `harness/`, `workflow/`). These
files are called "skills" but aren't structured as Claude Code skills (no
frontmatter, no discovery description, no usable content beyond a single
sentence), and there's no distinction between reference knowledge an agent
should consult automatically and processes a user/agent explicitly triggers.

## Decision

Adopt the Claude Code skill/command format explicitly (accepting reduced
portability to other agent tools, per user's choice) and split the repo into
two top-level directories:

### `skills/` — reference knowledge, loaded by description match

One skill per current topic directory, each expanded from its 1-line files
into a full `SKILL.md` (frontmatter `name`/`description` starting with "Use
when...", Overview, When to Use, Quick Reference table, Common Mistakes):

- `clean-architecture` ← `architecture/*`
- `clean-code` ← `clean-code/*`
- `go-conventions` ← `go/*`
- `testing-practices` ← `testing/*`
- `event-driven-messaging` ← `messaging/*`
- `safe-refactoring` ← `refactoring/*`
- `engineering-governance` ← `governance/*` (boundaries/quality-gates are
  standing decision rules, not user-triggered actions — skill, not command)

### `commands/` — explicit action, user/agent runs it on demand

One command per file, frontmatter `description` + execution steps:

- `code-review.md` ← `workflow/code-review.md`
- `ci-pipeline.md`, `ai-code-review.md`, `autofix.md`, `test-automation.md`,
  `deployment-gates.md` ← `harness/*`

### Merged, not moved

`workflow/development-workflow.md` overlaps with `AGENTS.md`'s existing
before/during/after checklist — fold its content into `AGENTS.md` and delete
the file rather than making it a skill or command.

## Out of scope

- No change to the philosophy/principles content itself, only its structure
  and format.
- No CI/tooling changes (e.g., no skill-linter, no auto-validation of
  frontmatter) — not requested.

## Migration

Old directories (`architecture/`, `clean-code/`, `go/`, `testing/`,
`messaging/`, `refactoring/`, `governance/`, `harness/`, `workflow/`) are
deleted once content is migrated. `README.md` and `PLAYBOOK.md` are updated
to reference `skills/` and `commands/` instead of the old tree.
