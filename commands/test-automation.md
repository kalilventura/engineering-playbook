---
description: Add or extend automated test coverage, choosing the right level
---

Default to the lowest level that gives real confidence:

- **Unit tests** — live close to the code, cover business logic and edge cases. See `testing-practices` skill.
- **Integration tests** — at real infrastructure boundaries (DB, Kafka, storage, external services) where mocking would hide real risk.
- **E2E/browser tests** — reserved for critical, user-visible workflows. Don't use E2E as a substitute for unit/integration coverage.

Validate asynchronous outcomes by waiting on the actual condition (message consumed, row written), never with an arbitrary sleep.
