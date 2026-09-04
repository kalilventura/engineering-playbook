# Go: structured logging + OpenTelemetry

## 1. `slog` JSON handler with automatic trace injection

Always wrap the base `slog.JSONHandler` with a handler that pulls `trace_id`/`span_id` out of the `context.Context` if a span is active. This is the single most important piece — without it, nothing correlates.

```go
type otelHandler struct {
	slog.Handler
}

func (h otelHandler) Handle(ctx context.Context, r slog.Record) error {
	if span := trace.SpanContextFromContext(ctx); span.IsValid() {
		r.AddAttrs(
			slog.String("trace_id", span.TraceID().String()),
			slog.String("span_id", span.SpanID().String()),
		)
	}
	return h.Handler.Handle(ctx, r)
}

func newLogger() *slog.Logger {
	base := slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{Level: slog.LevelInfo})
	return slog.New(otelHandler{base}).With(
		slog.String("service.name", serviceName),
		slog.String("service.version", "1.0.0"),
		slog.String("deployment.environment", envOr("ENVIRONMENT", "dev")),
	)
}
```

**Always use `logger.InfoContext(ctx, ...)` / `ErrorContext(ctx, ...)` / `WarnContext(ctx, ...)`, never the context-less variants** (`logger.Info(...)`) inside request handling code — the context-less versions skip trace injection entirely since there's no `ctx` to pull the span from.

## 2. Gin access logs

`gin.Default()` includes `gin.Logger()`, which prints **plain text, not JSON, with no trace_id**:
```
[GIN] 2026/09/04 - 14:32:01 | 200 |      2.1ms |  127.0.0.1 | POST     "/orders"
```
Don't use it in production. Use `gin.New()` and add a custom structured middleware instead:

```go
func slogMiddleware(logger *slog.Logger) gin.HandlerFunc {
	return func(c *gin.Context) {
		start := time.Now()
		c.Next()
		ctx := c.Request.Context()
		logger.InfoContext(ctx, "http request",
			slog.String("method", c.Request.Method),
			slog.String("path", c.Request.URL.Path),
			slog.Int("status", c.Writer.Status()),
			slog.Duration("duration", time.Since(start)),
		)
	}
}
```

**Ordering matters.** `otelgin.Middleware` must run *before* the logging middleware, or the span won't be in the context yet when the log is emitted:

```go
router.Use(otelgin.Middleware(serviceName)) // 1. creates the span
router.Use(slogMiddleware(logger))          // 2. logs — trace_id now available
router.Use(gin.Recovery())
```

## 3. OTel SDK setup (tracer provider + propagator)

```go
func initTracer(ctx context.Context) (func(context.Context) error, error) {
	exporter, err := otlptracegrpc.New(ctx,
		otlptracegrpc.WithEndpoint(envOr("OTEL_EXPORTER_OTLP_ENDPOINT", "localhost:4317")),
		otlptracegrpc.WithInsecure(),
	)
	if err != nil {
		return nil, err
	}

	res, err := resource.New(ctx,
		resource.WithAttributes(
			semconv.ServiceName(serviceName),
			semconv.ServiceVersion("1.0.0"),
			attribute.String("deployment.environment", envOr("ENVIRONMENT", "dev")),
		),
	)
	if err != nil {
		return nil, err
	}

	tp := sdktrace.NewTracerProvider(
		sdktrace.WithBatcher(exporter),
		sdktrace.WithResource(res),
		sdktrace.WithSampler(sdktrace.ParentBased(sdktrace.TraceIDRatioBased(1.0))), // lower in prod
	)
	otel.SetTracerProvider(tp)
	otel.SetTextMapPropagator(propagation.NewCompositeTextMapPropagator(
		propagation.TraceContext{}, // W3C "traceparent" header
		propagation.Baggage{},
	))
	return tp.Shutdown, nil
}
```

## 4. Outbound HTTP calls (propagating context to another service)

Use `otelhttp.NewTransport` — it injects `traceparent` automatically on every outgoing request:

```go
var httpClient = &http.Client{
	Transport: otelhttp.NewTransport(http.DefaultTransport),
	Timeout:   5 * time.Second,
}

req, _ := http.NewRequestWithContext(ctx, http.MethodPost, url, body) // ctx must carry the span
resp, err := httpClient.Do(req) // traceparent injected here
```

If you build a request without `NewRequestWithContext(ctx, ...)` using the *live* context (e.g. you pass `context.Background()` instead), the header won't be injected and the downstream service starts a brand-new, disconnected trace.

## 5. Database driver / ORM logging

This is **not automatic** in Go — tracing and logging are two separate things you wire up separately:

- **Tracing (spans per query)**: use `otelsql` (wraps `database/sql`) or `otelgorm` for GORM. Gives you a child span per query, linked to the current trace, for free:
  ```go
  db, err := otelsql.Open("pgx", dsn, otelsql.WithAttributes(semconv.DBSystemPostgreSQL))
  ```
- **Structured logging of queries**: the driver won't emit JSON or trace_id on its own. You must implement the driver's logger interface (`pgxpool.Config.ConnConfig.Logger` for pgx, `logger.Interface` for GORM) and route it through your `slog.Logger`, passing the query's `context.Context` through so trace injection still works.

**Rule of thumb for Go**: every third-party library that logs needs to be explicitly told to use your `slog.Logger`, or it'll log its own way (usually plain text to stdout, with no trace_id) regardless of how well the rest of your app is configured.
