---
description: Diagnose and fix a bounded CI failure (compilation, test, lint) with the minimal change
---

Use for bounded failures only: compilation errors, failing tests, lint violations, other mechanical corrections.

1. Diagnose the root cause — read the actual failure, don't guess.
2. Apply the minimal fix that addresses the cause.
3. Validate: re-run the failing check, then the broader suite.
4. Expose the change for review (diff, PR) — don't merge autofix changes silently.

Never disable a test, weaken a lint rule, or lower a quality gate to make the failure go away. If the fix isn't bounded/mechanical (it requires a design decision), stop and hand it back for human/architectural input instead of forcing a fix. See `engineering-governance` skill.
