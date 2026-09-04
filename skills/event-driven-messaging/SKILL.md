---
name: event-driven-messaging
description: Use when designing events, writing Kafka consumers/producers, or defining event contracts between services
---

# Event-Driven Messaging

## Overview

Events are facts that occurred, and event contracts are APIs between independently evolving systems. Consumers are adapters — business logic stays in application/domain layers, not in the handler.

## When to Use

- Designing a new event/message contract.
- Writing or reviewing a Kafka consumer or producer.
- Deciding how a service should react to duplicate, out-of-order, or poison messages.

## Event-Driven Architecture

Events represent facts that already happened (`OrderPlaced`, not `PlaceOrder`). Consumers must account for:

- **duplicates** — processing must be idempotent (test this explicitly, see [[testing-practices]])
- **retries** — a redelivered message shouldn't cause a different outcome
- **ordering** — don't assume global order unless the transport guarantees it for your partitioning
- **poison messages** — a message that can never be processed successfully needs a dead-letter path, not an infinite retry loop
- **partial failures** — a handler that does two side effects needs to define what happens if the second fails after the first succeeds

Keep business orchestration in application services, not inside the Kafka handler itself.

## Kafka Consumers and Producers

Consumer flow (an adapter, see [[clean-architecture]]):

```
Kafka message
  → Deserialize
  → Validate contract
  → Map to application command
  → Execute use case
  → Handle acknowledgment/retry
```

Producers map application events to versioned contracts before publishing. Prefer IDs/references over embedding large documents in the payload — the payload should be enough to act on, not a full denormalized snapshot. Mapping and publishing belong in an outbound adapter, not inline in domain/application code (see [[clean-architecture]]).

## Event Contracts

Events are APIs. Define, at minimum:

- type
- version
- event ID
- entity ID
- timestamp
- metadata
- payload

Evolution rules: prefer additive, backward-compatible changes (new optional fields) over breaking changes (renaming/removing fields, changing types). Never put secrets or unnecessarily large payloads into an event.

## Quick Reference

| Concern | Rule |
|---|---|
| Same message delivered twice | Handler must be idempotent |
| Need to change an event's shape | Add a field, don't rename/remove — or version the event |
| Handler can't process a message ever | Dead-letter it, don't retry forever |
| Payload size | IDs/references over embedded documents |
| Business logic location | Application/domain, not the consumer handler |

## Common Mistakes

- Non-idempotent consumers that double-charge/double-send on redelivery.
- Renaming or removing a field in an existing event contract without versioning.
- Business rules (discount calculation, eligibility checks) living inside the Kafka handler.
- No dead-letter path — a malformed message blocks the whole partition on infinite retry.
