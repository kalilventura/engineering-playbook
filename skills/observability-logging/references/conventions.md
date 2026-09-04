# Cross-service conventions: propagation, pipeline, sampling

## Propagação entre serviços (W3C Trace Context)

The standard is the W3C Trace Context spec, propagated via an HTTP header:

```
traceparent: 00-{trace_id}-{span_id}-{flags}
tracestate: vendor-specific (optional)
```

Every OTel SDK (Go, Java, others) understands this header natively when using the standard propagator (`propagation.TraceContext{}` in Go, on by default with the Micrometer bridge in Java). As long as both sides use an instrumented HTTP client/server, propagation is automatic — no header handling code needed.

**Where propagation silently breaks:**
- Manually constructed requests that don't go through the instrumented client (e.g. raw `net.Dial`, a hand-rolled HTTP call bypassing `otelhttp`).
- Async messaging (Kafka, RabbitMQ, SQS): trace context does **not** propagate through message brokers automatically. You must inject `traceparent` into message headers on publish, and extract it on consume, manually.
- Background jobs/schedulers that don't originate from an inbound request — there's no context to propagate, so decide deliberately whether to start a fresh trace or leave tracing out entirely.
- A context passed as `context.Background()` instead of the live request context anywhere in the call chain (Go-specific footgun, but conceptually applies anywhere a "fresh" context object gets used mid-chain).

Symptom of broken propagation: instead of one trace spanning both services, you get two disconnected traces, each looking like it starts and ends in a single service. If that happens, check the outbound call site first — it's almost always a client not going through the instrumented transport.

## Semantic conventions checklist

Keep these identical (same key, same value) across every service in the system:

| Field | Example | Purpose |
|---|---|---|
| `service.name` | `payment-service` | Identifies the service in traces, logs, dashboards, Service Graph |
| `service.version` | `1.0.0` | Correlates issues to a specific deploy |
| `deployment.environment` | `production` / `staging` / `dev` | Separates environments in the same backend |

Field-level conventions for common operations (`http.request.method`, `db.system`, `error.type`, etc.) come from the OpenTelemetry Semantic Conventions spec — using the standard names (rather than inventing your own) is what lets logs, traces, and metrics all use the same vocabulary and get joined automatically in Grafana.

## Pipeline de coleta (OTel Collector)

Never send telemetry directly from an app to the storage backend (Tempo/Loki/Prometheus) in anything beyond a local test. Always go through an OTel Collector:

```
App (OTLP) → Collector → [Tempo, Loki, Prometheus]
```

Collector pipeline has three stages:
- **Receivers**: `otlp` (gRPC on 4317, HTTP on 4318) — both Go and Java apps in this stack export here.
- **Processors**: `batch` (always include — batches exports for efficiency), `attributes`/`resource` processors for enrichment, and (if using tail sampling) the `tail_sampling` processor.
- **Exporters**: one per backend — traces to Tempo, logs to Loki, metrics to Prometheus (or `prometheusremotewrite`).

Decoupling apps from the backend this way means: sampling decisions can be centralized, backend swaps don't touch app code, and you can fan out the same telemetry to multiple backends if needed.

## Sampling strategy

- **Head-based** (decided at the app, e.g. `TraceIDRatioBased(0.1)`): simple, cheap, but can miss rare error traces since the decision happens before you know if the request will fail.
- **Tail-based** (decided at the Collector, after seeing the whole trace): keeps all errors and slow traces regardless of the sampling rate, at the cost of needing to buffer spans until a trace completes.
- **Practical starting point for this stack**: head sampling at 10–20% in each service (`ParentBased(TraceIDRatioBased(0.1–0.2))` in Go, `management.tracing.sampling.probability` in Java) plus a Collector-side `tail_sampling` policy that always keeps traces with errors. `ParentBased` matters: it ensures a downstream service (e.g. Java) respects the sampling decision the upstream service (e.g. Go) already made, rather than each service sampling independently and fragmenting traces.

## Service Graph

Derived automatically by Tempo from spans — no separate instrumentation needed. Tempo looks at client→server span pairs (identified by `span.kind` and the propagated trace context) and computes RED metrics (rate, errors, duration) per service-to-service edge. Prerequisite: services must actually be connected via propagated trace context (see above) — if propagation is broken between two services, the Service Graph won't show an edge between them even if they call each other constantly.

## Correlating the three signal types in Grafana

- **Logs → Traces**: configure a "derived field" on the Loki datasource that extracts `trace_id` from the log line and links to Tempo.
- **Metrics → Traces**: use Tempo's span metrics (auto-generated RED metrics) with exemplars enabled in Prometheus — an exemplar is a single data point in a latency histogram that links back to the exact trace that produced it.
- **Traces → Logs**: in the Tempo datasource config, set up a trace-to-logs link pointing at Loki, filtering by `trace_id`.

All three depend on the same prerequisite: every signal must actually carry `trace_id`/`span_id` (or an equivalent field) using a consistent key name across services.
