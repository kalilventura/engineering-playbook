---
name: clean-code
description: Use when writing or reviewing functions, deciding whether to extract or split code, or judging whether an abstraction is justified
---

# Clean Code

## Overview

Reduce accidental complexity, not enforce arbitrary formatting. The goal is code that's easier to understand, change, test, and review.

## When to Use

- Writing a new function and deciding its shape.
- Reviewing whether a function or abstraction is doing too much.
- Deciding whether to add a wrapper, interface, or configuration option.

## Small, Readable Functions

Prefer functions with:

- one clear purpose
- meaningful names
- limited branching, low nesting
- explicit inputs/outputs (no hidden globals/state)

Concrete signal, not a hard rule: nesting past 2-3 levels deep, or a function you can't summarize in one sentence without "and", is a strong candidate to split or flatten (guard clauses, early returns, extracted predicate).

Extract code into its own function only when the extracted concept has a meaningful name or boundary — extraction for its own sake adds indirection without clarity.

## Single Responsibility

A function should have one focused reason to change. Separate these when they differ:

- validation
- business decisions
- persistence
- serialization
- external communication
- formatting

If a function changes for two unrelated reasons, split it along that seam.

## Removing Accidental Complexity

Before adding an abstraction, ask whether the complexity is actually required by the problem. Be cautious with:

- premature interfaces (see [[clean-architecture]] for when an interface is justified)
- wrappers around a dependency "just in case"
- generic abstractions built for one caller
- deep indirection (a call chain you must trace through 4 files to understand)
- unnecessary configuration for values that never change

## Quick Reference

| Smell | Fix |
|---|---|
| Function does validation + persistence + formatting | Split by responsibility |
| Deeply nested conditionals | Extract guard clauses, invert conditions |
| Interface with one implementation, never varies | Delete the interface |
| Config flag that's never toggled | Hardcode the value |
| Extracted helper used exactly once, no clearer name than inline | Inline it back |

## Common Mistakes

- Extracting a function that just renames a one-line operation without adding clarity.
- Adding a generic/config-driven abstraction for a single current use case.
- Treating "small function" as a line-count target instead of a responsibility boundary — a 3-line function that does two unrelated things is still worth splitting.
