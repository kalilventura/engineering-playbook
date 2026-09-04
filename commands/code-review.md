---
description: Review a change (human or AI-authored) for correctness, architecture, tests, security, and scope
---

Review the diff/PR in front of you, against repository instructions (AGENTS.md/CLAUDE.md) and the playbook. Cover, in order of priority:

1. **Correctness** — does it do what it claims, including edge cases?
2. **Requirements** — does it match the actual ask, not more/less?
3. **Architecture** — see `clean-architecture` skill: dependency direction, boundaries, unjustified abstractions.
4. **Error handling** — are failures handled explicitly, not swallowed?
5. **Tests** — see `testing-practices` skill: are the right things tested, at the right level?
6. **Security** — secrets, injection, auth/authz, unsafe deserialization.
7. **Maintainability** — see `clean-code` skill.
8. **Performance** — only flag if there's a concrete, realistic cost.
9. **Observability** — see `observability-logging` skill: logs/metrics/traces for anything operationally significant.
10. **Scope** — unrelated changes bundled in, or missing changes the stated scope implies.

Classify every finding as one of:

- **Blocking** — must be fixed before merge (correctness, security, broken contract).
- **Important** — should be fixed, but doesn't have to block if tracked.
- **Suggestion** — optional improvement.

Prioritize actionable findings over cosmetic preferences. Each finding needs a concrete failure scenario, not a taste opinion.

Do not replace human approval for high-impact changes (see `engineering-governance` skill for what counts as high-impact). Do not silently modify production code as part of the review — propose the change, don't apply it.
