---
name: go-conventions
description: Use when writing Go services - structuring packages, wiring dependencies, defining interfaces, handling errors, or propagating context/goroutines
---

# Go Conventions

## Overview

Practical conventions for Go services: explicit construction over global bootstrap, small consumer-owned interfaces, and errors that carry context without panicking.

## When to Use

- Laying out a new Go service or package.
- Deciding whether something needs an interface, and where it should live.
- Writing or reviewing error handling in Go code.
- Propagating `context.Context` or starting/coordinating goroutines.

## Project Structure

```
cmd/<service>/main.go
internal/
  domain/
  application/
  infrastructure/
  interfaces/
```

Each executable's `main.go` is its composition root — the one place that constructs concrete dependencies and wires them together. Prefer explicit construction (pass dependencies in) over global bootstrap packages or `init()` magic.

## Interfaces and Dependency Injection

- Prefer small, consumer-owned interfaces defined where they're used, not where they're implemented.
- Use interfaces for: external dependencies, meaningful variation, strategies, dependency inversion at a real boundary (see [[clean-architecture]]).
- Prefer constructor injection over service locators or global singletons.
- Prefer simple hand-written fakes over mocking frameworks for tests (see [[testing-practices]]).

## Context and Concurrency

- Propagate `context.Context` explicitly as the first parameter through any call chain that does I/O (HTTP, DB, Kafka, other services) — don't store it on a struct.
- Respect cancellation: check `ctx.Err()` / select on `ctx.Done()` in loops or long-running work instead of ignoring it.
- Set timeouts/deadlines at the boundary that starts the call chain (e.g., an inbound HTTP handler), not buried in a leaf function.
- Every goroutine you start must have a clear owner and a clear stop condition — a goroutine with no way to be cancelled or joined is a leak.
- Prefer channels/`sync.WaitGroup`/`errgroup` for coordinating goroutines over ad-hoc `sync.Mutex`-guarded flags, unless you're protecting simple shared state.
- Don't launch a goroutine per request/item without bounding concurrency (worker pool, semaphore) when the input size is unbounded.

## Error Handling

- Return errors explicitly; don't use panic for ordinary application failures (reserve panic for programmer errors that should crash).
- Add useful context when wrapping (`fmt.Errorf("...: %w", err)`), and preserve the underlying cause so callers can inspect it.
- Don't swallow errors (`_ = err` without a documented reason).
- Never include secrets, tokens, or credentials in error messages or logs.

## Quick Reference

| Situation | Do |
|---|---|
| Need to call Postgres from application layer | Define a small interface in application, implement it in infrastructure |
| Error from a lower layer | Wrap with context, propagate, don't panic |
| Wiring dependencies | Construct explicitly in `main.go`, inject via constructor |
| Testing a consumer of an interface | Hand-written fake, not a mocking framework, unless verifying an interaction matters |
| Function does I/O | Take `ctx context.Context` as first param, propagate it down |
| Starting a goroutine | Give it an owner and a stop condition (cancel via ctx, join via WaitGroup/errgroup) |
| Unbounded input triggering goroutines | Bound concurrency with a worker pool or semaphore |

## Common Mistakes

- Defining interfaces next to their implementation instead of next to the consumer that needs them.
- Panicking on expected failures (bad input, network errors) instead of returning an error.
- Wrapping errors without context (`return err` losing the call site).
- Global `var db *sql.DB` instead of explicit construction and injection.
- Storing `context.Context` on a struct instead of passing it explicitly.
- Firing goroutines with no way to cancel or wait for them (leaks on shutdown/error paths).
