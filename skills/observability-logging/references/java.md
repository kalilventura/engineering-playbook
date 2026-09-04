# Java (Spring Boot): structured logging + OpenTelemetry

## 1. Dependencies

No Java Agent needed for this pattern — use the Micrometer Tracing bridge, which instruments Spring's own HTTP/JDBC layers and wires trace context into logging automatically:

```xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-tracing-bridge-otel</artifactId>
</dependency>
<dependency>
    <groupId>io.opentelemetry</groupId>
    <artifactId>opentelemetry-exporter-otlp</artifactId>
</dependency>
<dependency>
    <groupId>net.logstash.logback</groupId>
    <artifactId>logstash-logback-encoder</artifactId>
    <version>7.4</version>
</dependency>
```

(If the org standardizes on the OTel Java Agent instead, the agent handles instrumentation at the bytecode level and this bridge dependency isn't needed — check which approach the team already uses before adding both.)

## 2. Config (`application.yml`)

```yaml
management:
  tracing:
    sampling:
      probability: 1.0   # lower in prod (e.g. 0.1), pair with tail-sampling in the Collector
  otlp:
    tracing:
      endpoint: http://localhost:4318/v1/traces

otel:
  resource:
    attributes:
      service.name: payment-service
      service.version: 1.0.0
      deployment.environment: dev
```

`service.name`/`deployment.environment` here must match, field for field, whatever the Go side (or any other service) uses — this is what makes Grafana's Service Graph and cross-service dashboards work.

## 3. Automatic MDC population — no manual code required

Once the tracing bridge is on the classpath, Micrometer Tracing populates SLF4J's MDC with `traceId` and `spanId` automatically for **any log statement emitted inside a traced request** — including logs from your own code, from Spring's internals, and from any library that logs via SLF4J (Hibernate included). You never call anything like `MDC.put("traceId", ...)` yourself.

```java
private static final Logger log = LoggerFactory.getLogger(PaymentController.class);

log.info("payment processed order_id={} transaction_id={}", orderId, transactionId);
// -> trace_id/spanId are already in the JSON output, injected via MDC
```

## 4. Logback config for JSON output (`logback-spring.xml`)

```xml
<configuration>
    <springProperty scope="context" name="serviceName" source="spring.application.name"/>

    <appender name="JSON_STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <encoder class="net.logstash.logback.encoder.LogstashEncoder">
            <customFields>{"service.name":"${serviceName}"}</customFields>
            <fieldNames>
                <timestamp>timestamp</timestamp>
                <message>message</message>
                <logger>logger</logger>
                <level>level</level>
            </fieldNames>
            <includeMdcKeyName>traceId</includeMdcKeyName>
            <includeMdcKeyName>spanId</includeMdcKeyName>
        </encoder>
    </appender>

    <root level="INFO">
        <appender-ref ref="JSON_STDOUT"/>
    </root>
</configuration>
```

Rename `traceId`/`spanId` to `trace_id`/`span_id` in the encoder output if you want field names to match the Go side exactly (Loki correlation doesn't require identical names, but it makes dashboards and derived-field configs simpler to share across teams).

## 5. Why Hibernate/JPA logs "just work"

Hibernate logs through SLF4J like everything else in the Spring ecosystem. Since SLF4J routes to the same Logback pipeline you configured above, **Hibernate's SQL logs automatically come out as JSON with the correct trace_id** — no extra wiring, unlike the Go side where every driver needs explicit configuration.

To see SQL statements at all, enable it explicitly (off by default, verbose):
```yaml
logging:
  level:
    org.hibernate.SQL: DEBUG
```
Keep this at `DEBUG` and disabled in production by default — see the "log levels" principle in the main SKILL.md.

## 6. Receiving a propagated trace (inbound requests from other services)

No manual code is needed here either. As long as `micrometer-tracing-bridge-otel` is on the classpath and Spring MVC handles the request, the incoming `traceparent` header is extracted automatically, the trace continues (rather than starting a new one), and a child span is created for the request — MDC gets populated in time for the very first log line in your controller.
