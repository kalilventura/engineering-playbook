---
name: testing-practices
description: Use when writing or reviewing unit tests, integration tests, or test doubles - deciding what to test, how to isolate it, and which double to use
---

# Testing Practices

## Overview

Tests are production code and part of the design. Different test types provide different confidence — unit tests for fast feedback on behavior, integration tests for real boundaries, E2E for critical user-visible flows.

## When to Use

- Writing a new test and deciding its shape and scope.
- Reviewing a test for flakiness, unclear intent, or over-mocking.
- Deciding between a unit test and an integration test for a given piece of behavior.
- Choosing a test double.

## Fast, Independent, Self-Validating

Tests should be fast, independent, deterministic, and self-validating. Avoid:

- shared mutable state between tests
- arbitrary sleeps to synchronize async behavior — use polling/condition-based waits instead
- hidden dependencies (test only passes given prior test's side effect)
- manual inspection to determine pass/fail

## One Behavior Per Test

Each test should primarily communicate one behavior or rule. Multiple assertions are fine when they describe the same behavior (e.g., checking several fields of one resulting object). Name tests to communicate scenario, action, and expected outcome.

## Readable, Maintainable Tests

- Clear setup, meaningful test data (not `"foo"`/`"bar"` when a realistic value would clarify intent).
- Focused assertions — assert the behavior under test, not incidental details.
- Simple helpers, not a test framework of their own.
- Avoid coupling tests to implementation trivia (internal field order, private method calls) — couple to observable behavior.

## Test Doubles

Use the simplest useful double:

| Double | Use when |
|---|---|
| Fake | A working, simplified implementation exists or is cheap to write — prefer this |
| Stub | You just need a canned return value |
| Spy | You need to know it was called, but don't need to verify all interaction details |
| Mock | The interaction itself (call sequence, arguments) is part of the contract being tested |

Avoid verifying implementation details unless the interaction is genuinely part of the contract — over-verifying makes tests brittle to refactors that don't change behavior.

## Integration Tests

Integration tests validate real boundaries: PostgreSQL, Kafka, object storage, OpenSearch, external services, where those interactions are important enough that mocking them would hide real risk.

- Isolate data per test (unique keys/schemas, not shared fixtures mutated across tests).
- Keep tests deterministic; clean up state after each run.
- Avoid arbitrary sleeps — wait on the actual condition (message consumed, row exists).

## Common Mistakes

- Using `time.Sleep`/arbitrary delays to wait for async work instead of polling a condition.
- Mocking every collaborator in a unit test, hiding whether the code actually integrates correctly (that's what integration tests are for).
- One test asserting three unrelated behaviors — split it.
- Integration tests sharing a database/fixture across tests, causing order-dependent failures.
