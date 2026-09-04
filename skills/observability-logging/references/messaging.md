# Distributed tracing with messaging (Kafka)

Messaging breaks two assumptions synchronous (HTTP) tracing makes: there's no persistent connection carrying the context, and the producer→consumer relationship isn't always 1:1 (batches can mix messages from several origin traces). This document covers how to propagate context correctly and how to handle both cases.

## 1. Propagation: context travels in message headers

Same mechanism as HTTP — W3C Trace Context (`traceparent`) — just injected into the message headers/metadata instead of an HTTP header. Most modern brokers support this natively (Kafka, RabbitMQ, SQS, Pub/Sub).

### Go — Kafka (`segmentio/kafka-go`)

**Producer:**
```go
carrier := propagation.MapCarrier{}
otel.GetTextMapPropagator().Inject(ctx, carrier)

headers := []kafka.Header{}
for k, v := range carrier {
    headers = append(headers, kafka.Header{Key: k, Value: []byte(v)})
}

msg := kafka.Message{
    Topic:   "orders",
    Value:   payload,
    Headers: headers, // traceparent goes in here
}
```

**Consumer:**
```go
carrier := propagation.MapCarrier{}
for _, h := range msg.Headers {
    carrier[h.Key] = string(h.Value)
}
ctx := otel.GetTextMapPropagator().Extract(context.Background(), carrier)

ctx, span := tracer.Start(ctx, "process-order-message",
    trace.WithSpanKind(trace.SpanKindConsumer),
)
defer span.End()
```

### Java — Spring Kafka

With `micrometer-tracing-bridge-otel` on the classpath, `KafkaTemplate` (producer) and `@KafkaListener` (consumer) inject/extract the `traceparent` automatically — no manual code, same experience as HTTP in Spring:

```java
@KafkaListener(topics = "orders")
public void consume(String payload) {
    // trace_id/spanId are already in the MDC when this method runs,
    // extracted automatically from the message header
    log.info("order message consumed payload={}", payload);
}
```

If manual control is needed (e.g. a broker without native Spring support, or processing outside the standard listener), use OTel's `Propagator` directly on the `Headers` of the `ConsumerRecord`/`ProducerRecord`, following the same principle as the Go example.

## 2. `SpanKind`: PRODUCER and CONSUMER, not CLIENT/SERVER

HTTP uses `CLIENT`/`SERVER`. Messaging has its own kinds in the OTel semantic conventions:

- `PRODUCER` — span for whoever publishes the message
- `CONSUMER` — span for whoever processes the message

This matters because tools like Tempo use `span.kind` to draw the Service Graph and aggregate RED metrics correctly. Using `CLIENT`/`SERVER` in an async flow makes the graph assume a request/response relationship that doesn't exist, distorting latencies and error rates.

Recommended attributes (messaging semantic conventions):
```
messaging.system = "kafka"
messaging.destination.name = "orders"
messaging.operation = "publish" | "process"
messaging.message.id = "..."
```

## 3. Parent-child vs. Span Links: the batch problem

**Message-by-message consumption**: parent-child works normally. The consumer span is a child of the context extracted from that specific message, forming a continuous producer → queue → consumer trace.

**Batch consumption** (common in Kafka with batched `poll()`, processing several messages from different producers/traces in a single invocation): parent-child doesn't work, because the batch can't "belong" to multiple origin traces at once. OTel's recommended solution is **Span Links**:

```go
ctx, batchSpan := tracer.Start(context.Background(), "process-batch",
    trace.WithSpanKind(trace.SpanKindConsumer),
)

for _, msg := range batch {
    carrier := extractCarrier(msg.Headers)
    msgCtx := otel.GetTextMapPropagator().Extract(context.Background(), carrier)
    linkedSpanContext := trace.SpanContextFromContext(msgCtx)

    // Link, not parent: references the original trace without "belonging" to it
    batchSpan.AddLink(trace.Link{SpanContext: linkedSpanContext})
}
```

Result: each original trace (producer → queue) stays complete and navigable on its own, and the batch span appears connected to each one by a link — letting you answer "which batch was this order processed in?" without forcing an artificial tree that doesn't reflect the real topology.

**Rule of thumb**: message-by-message → parent-child. Batched → batch span with links to each origin trace.

## 4. Logs in the async context

Same principle as always (`trace_id`/`span_id` in every log), with one particularity: explicitly record the **publish moment** and the **consume moment**, even if they're far apart in time — this is what lets you spot messages published but never processed, by looking at an incomplete trace in Tempo.

```json
{"level":"INFO","message":"order message published","trace_id":"4bf9...","messaging.destination":"orders"}
```
```json
{"level":"INFO","message":"order message consumed","trace_id":"4bf9...","messaging.operation":"process","lag_ms":1200}
```

## 5. Retry and Dead Letter Queue

When moving a message to the DLQ, re-propagate the original `traceparent` onto the error message. Without this, investigating "why did this message land in the DLQ" only sees the isolated consumption, with no visibility into where the message originated — losing exactly the context that's most useful for diagnosing the root cause.

## Summary

| Step | What to do |
|---|---|
| Publish | Inject `traceparent` into message headers, `PRODUCER` span |
| Consume (1 msg) | Extract `traceparent`, create a child `CONSUMER` span |
| Consume (batch) | Extract `traceparent` from each message, use **span links** on the batch span |
| Retry/DLQ | Re-propagate the original `traceparent` onto the error message |
| Logs | `trace_id` on publish and consume, even if far apart in time |
