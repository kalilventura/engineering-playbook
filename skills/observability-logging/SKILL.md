---
name: observability-logging
description: Use when adding or reviewing logging/observability in a Go or Java service - structured logs, trace/log correlation, OpenTelemetry instrumentation, or a Loki/Tempo/Prometheus/Grafana pipeline
---

# Observability & Structured Logging (Go + Java)

## Overview

Structured JSON logs correlated with OpenTelemetry traces, consistent across Go (Gin/slog) and Java (Spring Boot/SLF4J) services that call each other. Logs and traces should complement each other, not exist as two disconnected data sources.

## When to Use

- Adding structured logging to a new Go or Java service.
- Reviewing existing logging/tracing setup for correctness or consistency.
- Wiring trace_id/span_id correlation between logs and traces.
- Setting up or reviewing the collection pipeline (OTel Collector, Tempo, Loki, Prometheus, Grafana).
- Debugging why logs and traces aren't correlating, or why a Service Graph edge is missing.

## Core Principles

- **JSON always.** Never plain-text/line-based logs in anything that ships to production.
- **trace_id + span_id in every log line** emitted inside a request context — this is what makes logs and traces useful together.
- **Log levels with discipline**: `ERROR` (something broke, needs action), `WARN` (degraded but functioning), `INFO` (business-relevant events, not "entered function X"), `DEBUG` (off by default in production).
- **Actionable messages, not descriptive ones.** Bad: `"error occurred"`. Good: `"failed to charge card: gateway timeout after 3 retries"`.
- **Never log secrets or PII**: passwords, tokens/API keys, full card numbers, SSNs/CPFs. Redact or omit — don't rely on downstream masking.
- **Watch cardinality.** Don't let high-cardinality fields (user IDs, request IDs, raw timestamps) become Loki index labels — keep them as log body fields.
- **One rich event per unit of work**, not scattered log lines for a single operation — mirrors how a span already models the work.
- **Same vocabulary everywhere.** `service.name`, `service.version`, `deployment.environment` must be spelled identically across every service (see `references/conventions.md`).

## Workflow

1. Identify the language(s) involved — Go, Java, or both talking to each other.
2. Read the matching reference file(s) fully before writing code — they contain ready-to-adapt patterns, not just theory:
   - **`references/go.md`** — `slog` JSON handler with OTel trace injection, Gin middleware, instrumented HTTP client, database driver/ORM logging (pgx, GORM, `otelsql`).
   - **`references/java.md`** — Spring Boot + SLF4J/Logback with `logstash-logback-encoder`, Micrometer Tracing bridge, Hibernate/JPA log correlation.
   - **`references/conventions.md`** — semantic conventions, W3C Trace Context propagation across services, OTel Collector pipeline design, sampling strategy, Service Graph.
   - **`references/messaging.md`** — distributed tracing over Kafka/message brokers: header propagation, `PRODUCER`/`CONSUMER` span kinds, span links for batch consumption, DLQ re-propagation.
3. If multiple services are involved, confirm trace context actually propagates end-to-end (see `references/conventions.md` → "Propagação entre serviços") — a missing `traceparent` header anywhere in the chain breaks correlation silently.
4. If database drivers/ORMs are involved, check the language reference — tracing spans are usually automatic, but structured *logging* of queries usually needs explicit logger wiring (not automatic in Go; mostly is in Java via SLF4J).
5. If setting up the collection pipeline itself (Collector, Tempo, Loki, Prometheus, Grafana datasources), go to `references/conventions.md` → "Pipeline de coleta".
6. If a message broker (Kafka, RabbitMQ, SQS) is involved anywhere in the flow, go to `references/messaging.md` — trace context does not propagate through brokers automatically and needs manual header injection/extraction.

## Quick Reference

| Situation | Do |
|---|---|
| Adding logging to a new service | Structured JSON via the idiomatic logger (`log/slog`, SLF4J+Logback) |
| Correlating logs to traces | Inject trace_id/span_id into every log line inside a request context |
| Calling another service | Use the instrumented HTTP client so `traceparent` propagates automatically |
| Publishing/consuming via Kafka/RabbitMQ/SQS | Manually inject/extract `traceparent` in message headers — brokers don't propagate it (see `references/messaging.md`) |
| Naming a resource attribute | Use OpenTelemetry Semantic Conventions, not an invented name |
| Sending telemetry to a backend | Always through an OTel Collector, never directly from the app |
| Deciding sampling | Head sampling 10–20% + Collector-side `tail_sampling` that keeps all errors |

## Common Mistakes

- Plain-text logs in production instead of structured JSON.
- Logging without trace_id/span_id, leaving logs and traces as disconnected data sources.
- Using high-cardinality fields (user ID, request ID) as Loki labels instead of body fields.
- A manually constructed HTTP call bypassing the instrumented client, silently breaking trace propagation.
- Assuming trace context propagates through Kafka/message brokers without manual header injection/extraction.
- Inconsistent `service.name`/`deployment.environment` values across services, breaking Grafana's Service Graph and cross-service correlation.
- Logging secrets, tokens, or PII instead of redacting them at the source.
