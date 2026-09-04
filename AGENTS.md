# AGENTS.md

Engineering standards live in `PLAYBOOK.md`, the skills in `skills/`, and the commands in `commands/` in this repository.

Before coding, understand: existing code and tests, the actual requirement, where the behavior currently lives, what architectural boundary is involved, and which skill(s) apply.

During coding, make the smallest coherent change, follow existing conventions, keep dependencies explicit, keep responsibilities focused, and add or update tests alongside the implementation. Avoid unrelated cleanup.

Before finishing, verify tests, static analysis, integration checks where relevant, build, security checks where relevant, the final diff, and architectural consistency.

Full task sequence:
1. Understand existing code and tests.
2. Classify the task.
3. Identify boundaries and contracts.
4. Select the relevant skill.
5. Make the smallest coherent change.
6. Add or update tests.
7. Run validation.
8. Inspect the final diff.
9. Review against the applicable skills.

General rules:
- No unjustified abstractions.
- Preserve dependency direction.
- Keep domain/application code independent of transport and infrastructure.
- Keep business logic out of adapters.
- Prefer constructor injection and explicit composition.
- Integration tests should exercise real boundaries.
- Do not use arbitrary sleeps to synchronize tests.
- Avoid unrelated refactoring.
- Respect project conventions.

Completion requires the requested behavior, relevant tests, coherent architecture, appropriate validation, no unrelated changes, and understood risks/assumptions.
