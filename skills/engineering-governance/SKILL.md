---
name: engineering-governance
description: Use when deciding what an AI agent may change autonomously versus what needs human approval, or defining quality gates for a pipeline
---

# Engineering Governance

## Overview

Boundaries for AI-assisted development: automation should reinforce engineering standards, not create a path around them. Agents get real autonomy for bounded, reversible work; humans stay accountable for high-impact decisions.

## When to Use

- Deciding whether a change is safe to make autonomously or needs human sign-off.
- Defining or reviewing CI quality gates.
- Reviewing AI-generated code review findings for severity and actionability.

## Agent Boundaries

Agents may: explain code, review changes, generate tests, propose refactors, apply bounded fixes (compilation errors, lint, mechanical corrections).

Human judgment remains required for: major architecture changes, sensitive security decisions, risky data migrations, breaking API/event contracts, production deployment approval.

Agents must never weaken a quality gate to get past it (disabling a test, lowering coverage thresholds, removing a security check) — see Quality Gates below.

### Risk Tiers

Classify a change before deciding autonomy, don't default to "small = safe":

| Tier | Examples | Autonomy |
|---|---|---|
| Low | Lint/format fixes, typo fixes, adding tests for existing behavior, mechanical renames with no behavior change ([[safe-refactoring]]) | Agent may apply directly |
| Medium | New endpoint/handler in an established pattern, non-breaking refactor inside a module, adding a new test suite ([[testing-practices]]) | Agent proposes, fast human review |
| High | Schema/data migrations, auth/payment paths, breaking API or event contract changes, cross-service architecture changes | Human approval required before merge, regardless of diff size |

A large diff isn't automatically high risk, and a one-line diff isn't automatically low risk (e.g. a one-line change to a migration or an auth check is high risk).

## Quality Gates

Recommended gates: build, unit tests, integration tests, static analysis, security scanning, meaningful coverage, architecture checks, AI review. Gates must be explicit, reproducible, and visible — not a manual step someone might skip.

## Automated Review

AI review should assess: requirements, correctness, architecture, tests, security, maintainability, complexity, scope. Findings need a severity and a rationale — "this could be better" without a reason isn't actionable. AI review must not silently modify production code without an explicit remediation workflow (e.g., a proposed diff a human approves). See [[testing-practices]] and [[safe-refactoring]] for what "well-tested" and "safely refactored" concretely mean when scoring findings.

## Quick Reference

| Situation | Rule |
|---|---|
| CI test fails | Diagnose and fix the cause — never disable the test to unblock the pipeline |
| AI review flags an issue | Attach severity + rationale, don't auto-apply to production code silently |
| Change touches auth/payments/migrations | Requires human approval regardless of how "small" it looks |
| Agent wants to lower a coverage threshold | Not allowed — that's weakening the gate, not passing it |

## Common Mistakes

- Treating a flaky test as "just disable it" instead of fixing the race condition.
- Letting an agent auto-merge a fix to a security-sensitive path without human review.
- AI review comments with no severity, making it impossible to triage blocking vs. cosmetic.
