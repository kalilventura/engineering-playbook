---
description: Lay out or review a CI pipeline for a service
---

Structure the pipeline so fast feedback comes first:

```
Checkout
  → Dependencies
  → Formatting / Static Analysis
  → Unit Tests
  → Integration Tests
  → Build
  → Security Scanning
  → AI Code Review
  → Coverage / Reporting
  → Artifact
  → Deployment Gates
  → Deploy
```

When reviewing an existing pipeline, check that: gates run in this rough order (cheap/fast checks before expensive ones), each stage is reproducible outside CI, and no stage can be skipped without an explicit override that's visible in the run. See `engineering-governance` skill for what "explicit and visible" means for quality gates, and `deployment-gates` command for the final gate before deploy.
