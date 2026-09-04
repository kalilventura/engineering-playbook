# Engineering Playbook

## Core Principles

1. Understand the existing system before changing it.
2. Prefer the smallest coherent change that solves the problem.
3. Keep business logic independent from transport and infrastructure.
4. Preserve dependency direction.
5. Introduce abstractions only when they represent a real boundary, capability, or meaningful variation.
6. Keep functions small and readable.
7. Write tests that are fast, independent, deterministic, and self-validating.
8. Use integration tests at real infrastructure boundaries.
9. Treat event contracts as APIs.
10. Refactor safely and avoid unrelated changes.
11. Automate quality gates, but keep humans responsible for architectural decisions.
12. AI-generated changes must be validated by tests, static analysis, and review.

## Skill Selection

- Architecture → `skills/clean-architecture/`
- Clean code → `skills/clean-code/`
- Go → `skills/go-conventions/`
- Testing → `skills/testing-practices/`
- Messaging/Kafka → `skills/event-driven-messaging/`
- Observability/logging → `skills/observability-logging/`
- Refactoring → `skills/safe-refactoring/`
- AI agent boundaries and quality gates → `skills/engineering-governance/`
- Library/framework documentation lookup → `skills/context7-mcp/`
- Docker compose setup → `skills/docker-compose-layered/`
- API endpoint documentation → `skills/documenting-api-endpoints/`

## Command Selection

- Code review (human or AI-authored) → `commands/code-review.md`
- CI pipeline design/review → `commands/ci-pipeline.md`
- Fixing a bounded CI failure → `commands/autofix.md`
- Adding test coverage → `commands/test-automation.md`
- Deployment readiness check → `commands/deployment-gates.md`
