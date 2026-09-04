---
name: clean-architecture
description: Use when defining or reviewing where code should live, drawing boundaries between domain/application/infrastructure, deciding whether to split a service, or introducing a port/interface/adapter
---

# Clean Architecture

## Overview

Business rules stay independent of frameworks, transport, and infrastructure. Dependencies point inward: adapters → application → domain. Composition roots wire concrete implementations at the edge.

## When to Use

- Deciding which layer new code belongs in.
- Reviewing whether a change leaks infrastructure concerns (HTTP, DB, Kafka, SDKs) into domain/application code.
- Deciding whether to introduce an interface/port, or whether to split a service.
- Doing an architecture review of a proposed or existing design.

## Dependency Direction

- Domain must remain framework-independent — no HTTP types, ORM structs, or SDK clients in domain code.
- Application depends on **contracts** (interfaces) for external capabilities, not concrete implementations.
- Infrastructure implements those contracts.
- Composition roots (e.g. `main.go`, DI setup) wire concrete implementations into the application at startup — this is the only place allowed to know about both sides.

## Ports and Adapters

- **Ports** define capabilities the application needs or exposes; **adapters** implement or consume them.
- Inbound ports: HTTP handlers, Kafka consumers, CLI commands — things that drive the application.
- Outbound ports: repositories, object storage, external APIs, event publishers — things the application drives.
- Do not create an interface only so a test can mock it. An interface should represent a real boundary, capability, strategy, or variation point.

## Service Boundaries

Split services around meaningful business or ownership boundaries, considering:

- data ownership
- change frequency
- deployment independence
- failure isolation
- communication contracts between services

Do not split a service merely because a package or file grew large — that's a refactor inside the boundary, not a new boundary.

## Quick Reference

| Question | Answer |
|---|---|
| Where does this business rule go? | Domain, if it holds regardless of transport/storage. |
| Where does "call Postgres" go? | Infrastructure, behind a port the application depends on. |
| Do I need an interface here? | Only if there's a real boundary/variation — not for mockability alone. |
| Should this be a new service? | Only if ownership, deployment, or failure isolation genuinely differ — not because a package is big. |
| How does this map to a Go package layout? | See [[go-conventions]] for concrete `internal/domain|application|infrastructure` structure. |

## Architecture Review Checklist

When reviewing an architecture or a proposed change, check: problem definition, boundaries, dependency direction, contracts, complexity, failure modes, observability, evolution path, and trade-offs. Prefer documented trade-offs over universal pattern claims ("always use CQRS") — justify the choice for this system.

## Common Mistakes

- Interfaces created purely for test mocking, with only one real implementation and no variation point.
- Domain code importing a web framework or ORM type.
- Splitting a monolith into services along file-size lines instead of ownership/deployment lines.
- Business orchestration logic living inside a controller or Kafka handler instead of the application layer.
