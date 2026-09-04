---
description: Verify a change is ready to deploy
---

Before approving deployment, verify:

- CI passed (build, tests, static analysis).
- Security scan has no unresolved critical findings.
- Test results are traceable to the artifact being deployed.
- Required approvals are present.
- Environment-specific checks (config, migrations, feature flags) are done.

AI may assist by gathering this evidence and diagnosing failures, but the production approval decision follows the team's risk policy — see `engineering-governance` skill for what requires human sign-off regardless of how clean the checks look.
