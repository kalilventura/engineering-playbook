---
name: safe-refactoring
description: Use when restructuring existing code without changing its behavior, or deciding how to sequence a refactor safely
---

# Safe Refactoring

## Overview

Refactoring is incremental and protected by tests. Separate structural changes from behavior changes so failures are easy to diagnose.

## When to Use

- Restructuring code (extracting, renaming, moving, simplifying) without intending to change behavior.
- A task mixes "fix/add behavior" with "the surrounding code needs cleanup" — decide whether to split into two changes.

## Workflow

```
Understand current behavior
        ↓
Establish confidence with tests (add characterization tests if missing)
        ↓
Make one coherent structural change
        ↓
Run focused tests, then broader validation
        ↓
Inspect the diff
        ↓
Repeat
```

Prefer separating feature changes from large refactors whenever practical — a PR that both renames half the package and fixes a bug is hard to review and hard to bisect if something breaks.

## Quick Reference

| Situation | Do |
|---|---|
| Code has no tests before refactoring | Add characterization tests first, don't refactor untested behavior blind (see [[testing-practices]]) |
| Refactor + behavior change both needed | Split into two PRs/commits — structural, then behavioral |
| Large structural change | Break into smaller coherent steps, validate after each |
| Diff mixes rename + logic change | Separate them — a reviewer can't tell which caused a break |

## Common Mistakes

- Refactoring and changing behavior in the same step, so a test failure can't tell you which one broke.
- Making several unrelated structural changes at once instead of one coherent change validated in isolation.
- Skipping tests before refactoring code that has none — add characterization tests first.
