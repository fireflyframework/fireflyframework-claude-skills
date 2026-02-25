---
name: implement-observability
description: Use when adding metrics, tracing, or health checks to a Firefly Framework service — covers Micrometer metrics, OpenTelemetry tracing, custom health indicators, structured logging with Logstash, and reactive context propagation
---

# Implement Observability

## When to Use

Use this skill when adding any observability capability to a Firefly Framework service: Micrometer metrics, distributed tracing with OpenTelemetry, custom health indicators, structured logging, or reactive context propagation. The `fireflyframework-observability` module auto-configures all of these with sensible defaults. You only write code when you need custom metrics, custom health checks, or custom trace spans.

## Architecture Overview

The observability stack is built on three auto-configuration entry points:

- `FireflyObservabilityAutoConfiguration` -- master auto-configuration that activates metrics, tracing, and health based on classpath detection and `firefly.observability.*` properties. All features are enabled by default.
- `FireflyObservabilityEnvironmentPostProcessor` -- translates `firefly.observability.tracing.bridge` (OTEL or BRAVE) and `firefly.observability.metrics.exporter` (PROMETHEUS, OTLP, or BOTH) into Spring Boot auto-configuration exclusions at startup. No POM changes are needed to switch backends.
- `ReactiveContextPropagationAutoConfiguration` -- calls `Hooks.enableAutomaticContextPropagation()` to bridge ThreadLocal values (MDC, OpenTelemetry context) into Reactor Context across all operators and thread boundaries.

The default configuration profile (`application-firefly-observability.yml`) ships with the module and is included automatically.

## Metrics

### How Auto-Configuration Works

When `io.micrometer.core.instrument.MeterRegistry` is on the classpath and `firefly.observability.metrics.enabled` is `true` (the default), `FireflyObservabilityAutoConfiguration.MetricsConfiguration` registers a `FireflyMeterRegistryCustomizer` bean. This customizer applies common tags to every metric in the registry:

```java
// FireflyMeterRegistryCustomizer applies these tags to ALL metrics automatically
registry.config().commonTags(
    "application", applicationName,   // from spring.application.name
    "environment", environment         // from spring.profiles.active
);
```

The `ExemplarsAutoConfiguration` enables Prometheus exemplar linking when both `micrometer-registry-prometheus` and a tracing bridge are on the classpath. This lets you click a Prometheus metric in Grafana and jump directly to the trace that caused it.

### Creating Custom Metrics with FireflyMetricsSupport

All Firefly module metrics extend `FireflyMetricsSupport`, which provides null-safe metric creation, consistent naming via `MetricNaming`, thread-safe caching with `ConcurrentHashMap`, and reactive timed operations. When `MeterRegistry` is null (actuator not on classpath), all methods become no-ops.

To create a custom metrics class for your module, extend `FireflyMetricsSupport` and pass your module name to the constructor. All metrics are automatically prefixed with `firefly.{module}.`.

```java
package com.example.payments.metrics;

import io.micrometer.core.instrument.MeterRegistry;
import org.fireflyframework.observability.metrics.FireflyMetricsSupport;
import org.fireflyframework.observability.metrics.MetricTags;
import org.springframework.stereotype.Component;

import java.time.Duration;
import java.util.concurrent.atomic.AtomicLong;

@Component
public class PaymentMetrics extends FireflyMetricsSupport {

    private final AtomicLong activePayments = new AtomicLong(0);

    public PaymentMetrics(MeterRegistry meterRegistry) {
        super(meterRegistry, "payments");

        // Register gauges in the constructor -- bound to a live object
        gauge("active.count", activePayments, AtomicLong::get);
    }

    public void recordPaymentSuccess(String paymentType, Duration duration) {
        // Counter: firefly.payments.payment.processed (status=success)
        recordSuccess("payment.processed", MetricTags.OPERATION, paymentType);

        // Timer: firefly.payments.payment.duration (with p50, p95, p99 percentiles)
        timer("payment.duration",
                MetricTags.OPERATION, paymentType,
                MetricTags.STATUS, MetricTags.SUCCESS)
                .record(duration);
    }

    public void recordPaymentFailure(String paymentType, Throwable error) {
        // Counter: firefly.payments.payment.processed (status=failure, error.type=...)
        recordFailure("payment.processed", error, MetricTags.OPERATION, paymentType);
    }

    public void recordPaymentAmount(String paymentType, double amount) {
        // Distribution summary: firefly.payments.payment.amount (with p50, p95, p99)
        distributionSummary("payment.amount",
                MetricTags.OPERATION, paymentType)
                .record(amount);
    }

    public void paymentStarted() {
        activePayments.incrementAndGet();
    }

    public void paymentCompleted() {
        activePayments.decrementAndGet();
    }
}
```

### Metric Naming Conventions

The `MetricNaming` class enforces consistent naming. All metrics follow the pattern `firefly.{module}.{metric}`:

- `MetricNaming.prefix("payments")` returns `"firefly.payments"`
- `MetricNaming.name("firefly.payments", "payment.processed")` returns `"firefly.payments.payment.processed"`
- Module names must be lowercase alphanumeric starting with a letter

### Standard Metric Tags

Use constants from `MetricTags` for consistent tag keys and values across all modules:

| Constant | Value | Usage |
|---|---|---|
| `MetricTags.STATUS` | `"status"` | Operation outcome |
| `MetricTags.SUCCESS` | `"success"` | Successful operation |
| `MetricTags.FAILURE` | `"failure"` | Failed operation |
| `MetricTags.ERROR_TYPE` | `"error.type"` | Exception class name |
| `MetricTags.OPERATION` | `"operation"` | Operation name |
| `MetricTags.COMMAND_TYPE` | `"command.type"` | CQRS command class |
| `MetricTags.QUERY_TYPE` | `"query.type"` | CQRS query class |
| `MetricTags.EVENT_TYPE` | `"event.type"` | Domain event class |
| `MetricTags.PUBLISHER_TYPE` | `"publisher.type"` | EDA publisher |
| `MetricTags.CONSUMER_TYPE` | `"consumer.type"` | EDA consumer |
| `MetricTags.DESTINATION` | `"destination"` | Topic/queue name |
| `MetricTags.AGGREGATE_TYPE` | `"aggregate.type"` | Event sourcing aggregate |

### Reactive Timer Support

`FireflyMetricsSupport` provides `timed()` methods that wrap `Mono` and `Flux` with timer recording:

```java
public Mono<PaymentResult> processPayment(PaymentRequest request) {
    Mono<PaymentResult> operation = paymentGateway.charge(request);

    // Wraps the Mono -- timer starts on subscription, stops on any terminal signal
    return timed("payment.gateway.duration", operation,
            MetricTags.PROVIDER, "stripe");
}
```

### Built-In Module Metrics

The CQRS module provides `CommandMetricsService` (extends `FireflyMetricsSupport` with module `"cqrs"`) which automatically records:

| Metric | Type | Description |
|---|---|---|
| `firefly.cqrs.command.processed` | Counter | Total commands processed successfully |
| `firefly.cqrs.command.failed` | Counter | Total commands that failed |
| `firefly.cqrs.command.validation.failed` | Counter | Total validation failures |
| `firefly.cqrs.command.processing.time` | Timer | Command processing duration |
| `firefly.cqrs.command.type.processed` | Counter | Per-command-type success (tag: `command.type`) |
| `firefly.cqrs.command.type.failed` | Counter | Per-command-type failure (tag: `command.type`) |
| `firefly.cqrs.command.type.processing.time` | Timer | Per-command-type duration (tag: `command.type`) |

The EDA module provides `EdaMetrics` (extends `FireflyMetricsSupport` with module `"eda"`) which records:

| Metric | Type | Description |
|---|---|---|
| `firefly.eda.publish.duration` | Timer | Event publish latency (tags: `publisher.type`, `destination`, `event.type`, `status`) |
| `firefly.eda.publish.count` | Counter | Event publish count (tags: `publisher.type`, `destination`, `status`) |
| `firefly.eda.publish.message.size` | Summary | Published message size in bytes |
| `firefly.eda.consume.duration` | Timer | Event consume latency (tags: `consumer.type`, `source`, `event.type`, `status`) |
| `firefly.eda.consume.count` | Counter | Event consume count (tags: `consumer.type`, `source`, `status`) |
| `firefly.eda.publisher.health` | Gauge | Publisher health (1=healthy, 0=unhealthy) |
| `firefly.eda.consumer.health` | Gauge | Consumer health (1=healthy, 0=unhealthy) |
| `firefly.eda.circuit.breaker.state.change` | Counter | Circuit breaker transitions |
| `firefly.eda.retry.attempt` | Counter | Retry attempts (tags: `attempt`, `status`) |
| `firefly.eda.listener.errors` | Counter | Listener error count (tags: `listener.method`, `error.type`, `event.type`) |

### Custom Actuator Endpoints

The CQRS module includes a custom actuator endpoint `CqrsMetricsEndpoint` registered at `/actuator/cqrs`:

```
GET /actuator/cqrs           -- Complete CQRS metrics overview
GET /actuator/cqrs/commands  -- Command processing metrics
GET /actuator/cqrs/queries   -- Query processing metrics
GET /actuator/cqrs/handlers  -- Handler registry information
GET /actuator/cqrs/health    -- CQRS framework health status
```

### Metrics Exporter Configuration

The exporter is controlled by a single property. The `FireflyObservabilityEnvironmentPostProcessor` sets the Spring Boot enable flags automatically:

```yaml
firefly:
  observability:
    metrics:
      exporter: PROMETHEUS   # PROMETHEUS (default), OTLP, or BOTH
```

| Value | Prometheus Scrape | OTLP Push | Use Case |
|---|---|---|---|
| `PROMETHEUS` | Enabled | Disabled | Standard Prometheus/Grafana/Thanos/Mimir |
| `OTLP` | Disabled | Enabled | Push to otel-collector-contrib |
| `BOTH` | Enabled | Enabled | Migration or multiple backends |

## Tracing

### How Auto-Configuration Works

When `io.micrometer.observation.ObservationRegistry` is on the classpath and `firefly.observability.tracing.enabled` is `true` (the default), `FireflyObservabilityAutoConfiguration.TracingConfiguration` registers:

1. `FireflyTracingSupport` -- reactive-safe span creation using the Micrometer Observation API
2. `FireflyBaggageConfiguration` -- holds the list of custom baggage field names to propagate (default: `X-Transaction-Id`)

### Tracing Bridge Selection

Both OpenTelemetry and Brave bridges ship on the classpath. The `FireflyObservabilityEnvironmentPostProcessor` reads `firefly.observability.tracing.bridge` and excludes the unused bridge's auto-configuration:

```yaml
firefly:
  observability:
    tracing:
      bridge: OTEL           # OTEL (default) or BRAVE
      sampling-probability: 1.0   # Override in production (e.g. 0.1 for 10%)
      propagation-type: W3C  # W3C (default) or B3; composite propagator includes both
      baggage-fields:
        - X-Transaction-Id
```

| Bridge | Propagation | Export | Use Case |
|---|---|---|---|
| `OTEL` | W3C TraceContext | OTLP to otel-collector | Default, modern stacks |
| `BRAVE` | B3 | Zipkin-compatible | Backward compat with Zipkin ecosystems |

### Creating Custom Trace Spans with FireflyTracingSupport

`FireflyTracingSupport` wraps `Mono` and `Flux` operations with named Observation spans. The Observation is propagated via Reactor Context (not ThreadLocal), making it safe for reactive chains that switch schedulers.

```java
package com.example.payments.service;

import io.micrometer.common.KeyValues;
import org.fireflyframework.observability.tracing.FireflyTracingSupport;
import org.springframework.stereotype.Service;
import reactor.core.publisher.Mono;

@Service
public class PaymentProcessingService {

    private final FireflyTracingSupport tracingSupport;
    private final PaymentGateway gateway;

    public PaymentProcessingService(FireflyTracingSupport tracingSupport,
                                     PaymentGateway gateway) {
        this.tracingSupport = tracingSupport;
        this.gateway = gateway;
    }

    public Mono<PaymentResult> processPayment(PaymentRequest request) {
        // Simple span -- just a name
        return tracingSupport.trace("payment.process",
                gateway.charge(request));
    }

    public Mono<PaymentResult> processPaymentDetailed(PaymentRequest request) {
        // Span with key-values for dashboards and trace detail
        return tracingSupport.trace("payment.process",
                gateway.charge(request),
                // Low cardinality: indexed, used in dashboards (keep bounded)
                KeyValues.of("payment.method", request.getMethod(),
                             "currency", request.getCurrency()),
                // High cardinality: NOT indexed, for trace detail only
                KeyValues.of("payment.id", request.getPaymentId()));
    }
}
```

### Accessing the Current Observation

To read the current Observation from within a reactive chain (without relying on ThreadLocal):

```java
FireflyTracingSupport.currentObservation()
    .flatMap(observation -> {
        // Add contextual data to the current span
        observation.highCardinalityKeyValue("order.id", orderId);
        return processOrder(orderId);
    });
```

### WebClient Trace Propagation

`TracingWebClientCustomizer` automatically customizes all `WebClient` instances to propagate the `X-Transaction-Id` header on outgoing HTTP requests. Standard trace context propagation (W3C TraceContext, B3) is already handled by Spring Boot's `ObservationWebClientCustomizer`. The transaction ID is read from the Reactor Context first (reactive-safe), falling back to MDC for non-reactive callers.

### OTLP Export Configuration

The default OTLP configuration targets `otel/opentelemetry-collector-contrib`. Override via environment variables:

| Variable | Default | Purpose |
|---|---|---|
| `OTEL_EXPORTER_OTLP_ENDPOINT` | `http://localhost:4317` | Collector endpoint (gRPC) |
| `OTEL_EXPORTER_OTLP_PROTOCOL` | `grpc` | Transport (`grpc` or `http/protobuf`) |
| `OTEL_EXPORTER_OTLP_TRACES_ENDPOINT` | Falls back to above | Trace-specific endpoint |
| `OTEL_EXPORTER_OTLP_METRICS_ENDPOINT` | `http://localhost:4318/v1/metrics` | Metrics-specific endpoint |

## Health Checks

### FireflyHealthIndicator Base Class

All Firefly health indicators extend `FireflyHealthIndicator` (which extends Spring Boot's `AbstractHealthIndicator`). The base class provides standard detail helpers for consistent health response formatting:

```java
package com.example.payments.health;

import org.fireflyframework.observability.health.FireflyHealthIndicator;
import org.springframework.boot.actuate.health.Health;
import org.springframework.stereotype.Component;

import java.time.Duration;

@Component
public class PaymentGatewayHealthIndicator extends FireflyHealthIndicator {

    private final PaymentGateway gateway;

    public PaymentGatewayHealthIndicator(PaymentGateway gateway) {
        super("paymentGateway");   // component name in health response
        this.gateway = gateway;
    }

    @Override
    protected void doHealthCheck(Health.Builder builder) throws Exception {
        builder.up();

        // Standard numeric detail
        addMetricDetail(builder, "transactions.today", gateway.getTransactionCount());

        // Error rate with threshold -- marks DOWN if rate > 0.05
        addErrorRate(builder, gateway.getErrorRate(), 0.05);

        // P99 latency with threshold -- marks DOWN if p99 > 500ms
        addLatency(builder, gateway.getP99Latency(), Duration.ofMillis(500));

        // Connection pool details -- marks DOWN if pool exhausted
        addConnectionPool(builder,
                gateway.getActiveConnections(),
                gateway.getIdleConnections(),
                gateway.getMaxConnections());
    }
}
```

Available helper methods on `FireflyHealthIndicator`:

| Method | Behavior |
|---|---|
| `addMetricDetail(builder, name, value)` | Adds a named numeric detail |
| `addErrorRate(builder, rate, threshold)` | Adds error rate; marks DOWN if rate exceeds threshold |
| `addLatency(builder, p99, threshold)` | Adds p99 latency; marks DOWN if latency exceeds threshold |
| `addConnectionPool(builder, active, idle, max)` | Adds pool details; marks DOWN if pool exhausted (active >= max) |

### HealthMetricsBridge

`HealthMetricsBridge` bridges health indicator status to Micrometer gauge metrics, registering a `firefly.health.status` gauge for each component. This allows health status to be scraped by Prometheus:

| Status | Gauge Value |
|---|---|
| UP | 1.0 |
| DOWN | 0.0 |
| OUT_OF_SERVICE | -1.0 |
| UNKNOWN | -2.0 |

Register your custom health indicator with the bridge:

```java
@Configuration
public class HealthConfig {

    @Bean
    public PaymentGatewayHealthIndicator paymentGatewayHealth(
            PaymentGateway gateway, HealthMetricsBridge bridge) {
        PaymentGatewayHealthIndicator indicator = new PaymentGatewayHealthIndicator(gateway);
        bridge.register("paymentGateway", indicator);
        return indicator;
    }
}
```

### Kubernetes Probes

`KubernetesProbesAutoConfiguration` activates Kubernetes probe groups when `firefly.observability.health.kubernetes-probes` is `true` (the default). The default configuration sets up:

```yaml
management:
  endpoint:
    health:
      show-details: always
      show-components: always
      probes:
        enabled: true
      group:
        liveness:
          include: livenessState              # NEVER check DB in liveness
        readiness:
          include: readinessState,db,diskSpace
```

- `/actuator/health/liveness` -- only livenessState (lightweight, never includes external dependencies)
- `/actuator/health/readiness` -- readinessState + db + diskSpace (reports whether the service can accept traffic)

The default profile exposes `health`, `info`, `metrics`, and `prometheus` endpoints via `management.endpoints.web.exposure.include`.

## Structured Logging

### How Auto-Configuration Works

`StructuredLoggingAutoConfiguration` activates when `firefly.observability.logging.enabled` is `true` (the default). It leverages Spring Boot 3.4+ built-in structured logging:

```yaml
firefly:
  observability:
    logging:
      enabled: true
      structured-format: logstash    # Maps to logging.structured.format.console

logging:
  structured:
    format:
      console: ${firefly.observability.logging.structured-format:logstash}
  level:
    org.fireflyframework: DEBUG
    root: INFO
```

This produces JSON-formatted log output using the Logstash encoder, with trace context fields automatically included when tracing is active.

### MDC Constants

The `MdcConstants` class defines standard MDC field names used across all Firefly modules for consistent log aggregation:

| Constant | MDC Key | Purpose |
|---|---|---|
| `MdcConstants.TRACE_ID` | `traceId` | Distributed trace ID |
| `MdcConstants.SPAN_ID` | `spanId` | Current span ID |
| `MdcConstants.TRANSACTION_ID` | `X-Transaction-Id` | Framework transaction ID |
| `MdcConstants.USER_ID` | `userId` | Audit user identifier |
| `MdcConstants.CORRELATION_ID` | `correlationId` | Cross-service correlation |
| `MdcConstants.REQUEST_ID` | `requestId` | HTTP request tracking |
| `MdcConstants.SERVICE_NAME` | `serviceName` | Multi-service aggregation |
| `MdcConstants.AGGREGATE_TYPE` | `aggregateType` | Event sourcing context |
| `MdcConstants.AGGREGATE_ID` | `aggregateId` | Event sourcing context |

### Using MDC in Reactive Code

With `ReactiveContextPropagationAutoConfiguration` active, you do NOT need manual MDC management. MDC values set before a reactive chain automatically propagate across scheduler switches. No more `doOnEach()`, `doFirst()`, or `doFinally()` MDC hacks.

```java
// MDC values set here will be available in ALL downstream operators,
// even after publishOn(Schedulers.parallel()) or subscribeOn(Schedulers.boundedElastic())
MDC.put("traceId", "abc123");
MDC.put("X-Transaction-Id", "tx-456");

Mono.just("start")
    .publishOn(Schedulers.boundedElastic())
    .map(s -> {
        // MDC.get("traceId") returns "abc123" here -- automatic propagation
        log.info("Processing on different thread");
        return s;
    })
    .publishOn(Schedulers.parallel())
    .map(s -> {
        // MDC.get("traceId") still returns "abc123" after second switch
        log.info("Processing on yet another thread");
        return s;
    });
```

## Reactive Context Propagation

`ReactiveContextPropagationAutoConfiguration` calls `Hooks.enableAutomaticContextPropagation()` at startup, bridging ThreadLocal values (MDC, OpenTelemetry context, Brave context) into Reactor Context across all thread boundaries. This requires `io.micrometer:context-propagation` on the classpath and Reactor 3.5.3+.

Configuration:

```yaml
firefly:
  observability:
    context-propagation:
      reactor-hooks-enabled: true   # default
```

When enabled, all ThreadLocal values registered with Micrometer's `ContextRegistry` are automatically captured at subscription time and restored before each operator executes on a different thread. This covers:

- SLF4J MDC (trace IDs, correlation IDs, transaction IDs)
- OpenTelemetry context (spans, baggage)
- Brave context (spans, B3 propagation)

### Important: No Manual MDC Wiring Needed

With automatic context propagation enabled, you should NOT write manual MDC propagation code. The following patterns are obsolete:

```java
// DO NOT DO THIS -- automatic context propagation handles it
.doOnEach(signal -> {
    if (signal.getContextView().hasKey("traceId")) {
        MDC.put("traceId", signal.getContextView().get("traceId"));
    }
})

// DO NOT DO THIS -- automatic context propagation handles it
.contextWrite(Context.of("traceId", MDC.get("traceId")))
```

## Complete Configuration Reference

```yaml
firefly:
  observability:
    # --- Metrics ---
    metrics:
      enabled: true                    # Enable/disable all metrics
      prefix: firefly                  # Metric name prefix
      exporter: PROMETHEUS             # PROMETHEUS, OTLP, or BOTH

    # --- Tracing ---
    tracing:
      enabled: true                    # Enable/disable all tracing
      bridge: OTEL                     # OTEL or BRAVE (no POM changes needed)
      sampling-probability: 1.0        # 0.0 to 1.0 (use 0.1 in production)
      propagation-type: W3C            # W3C or B3
      baggage-fields:
        - X-Transaction-Id             # Custom fields propagated across services

    # --- Health ---
    health:
      enabled: true                    # Enable/disable health indicators
      kubernetes-probes: true          # Enable liveness/readiness probe groups

    # --- Logging ---
    logging:
      enabled: true                    # Enable/disable structured logging
      structured-format: logstash      # Logstash JSON format

    # --- Context Propagation ---
    context-propagation:
      reactor-hooks-enabled: true      # Enable Reactor automatic context propagation

# Spring Boot Actuator (set by default profile, override as needed)
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  endpoint:
    health:
      show-details: always
      show-components: always
      probes:
        enabled: true
      group:
        liveness:
          include: livenessState
        readiness:
          include: readinessState,db,diskSpace
  tracing:
    enabled: ${firefly.observability.tracing.enabled:true}
    sampling:
      probability: ${firefly.observability.tracing.sampling-probability:1.0}
    propagation:
      type: W3C,B3
  otlp:
    tracing:
      endpoint: ${OTEL_EXPORTER_OTLP_TRACES_ENDPOINT:${OTEL_EXPORTER_OTLP_ENDPOINT:http://localhost:4317}}
      transport: ${OTEL_EXPORTER_OTLP_PROTOCOL:grpc}
    metrics:
      export:
        url: ${OTEL_EXPORTER_OTLP_METRICS_ENDPOINT:http://localhost:4318/v1/metrics}
  metrics:
    tags:
      application: ${spring.application.name:unknown}

server:
  shutdown: graceful

spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s
```

## Common Mistakes

1. **High-cardinality metric tags** -- Never use unbounded values (user IDs, request IDs, UUIDs) as metric tag values. These create a new time series per unique value, causing memory exhaustion in Prometheus. Use low-cardinality values (status codes, operation types, service names) for tags. Put high-cardinality data in trace span attributes via `KeyValues` instead.

2. **Blocking in health checks** -- `FireflyHealthIndicator.doHealthCheck()` runs on the Actuator thread pool. If your health check calls a reactive service, use `.block(Duration.ofSeconds(5))` with a timeout, or better yet, implement a `ReactiveHealthIndicator` instead. Never call `.block()` without a timeout.

3. **Manual MDC propagation in reactive chains** -- With `ReactiveContextPropagationAutoConfiguration` active, manual MDC wiring via `doOnEach()`, `contextWrite()`, or `doFirst()` is unnecessary and can cause double-writes or stale values. Remove all manual MDC code when automatic propagation is enabled.

4. **Forgetting to register health indicators with HealthMetricsBridge** -- If you create a custom `FireflyHealthIndicator` but do not call `healthMetricsBridge.register("name", indicator)`, the health status will not appear as a Prometheus gauge. Always register custom indicators with the bridge.

5. **Using ThreadLocal-based tracing in reactive code** -- Never use `Span.current()` or `Tracer.currentSpan()` directly in reactive operators. These read from ThreadLocal, which may point to the wrong span after a scheduler switch. Use `FireflyTracingSupport.trace()` or `FireflyTracingSupport.currentObservation()` which propagate via Reactor Context.

6. **Disabling context propagation without understanding the impact** -- Setting `firefly.observability.context-propagation.reactor-hooks-enabled=false` breaks trace correlation, MDC logging, and baggage propagation in all reactive chains. Only disable this if you have a specific incompatibility and you are prepared to handle context propagation manually.

7. **Not adjusting sampling probability in production** -- The default `sampling-probability: 1.0` traces every request. In production, set this to `0.1` (10%) or lower to avoid overwhelming the trace collector and storage backend.

8. **Including external dependencies in liveness probes** -- The default liveness group includes only `livenessState`. Never add `db`, `redis`, `diskSpace`, or other external checks to the liveness group. A database outage should not cause Kubernetes to restart your pod (liveness failure). External dependencies belong in the readiness group.

9. **Creating metrics outside FireflyMetricsSupport** -- Directly using `MeterRegistry.counter()` or `MeterRegistry.timer()` bypasses the framework's naming conventions, caching, and null-safety. Always extend `FireflyMetricsSupport` for module metrics, or use the protected helper methods (`counter()`, `timer()`, `gauge()`, `distributionSummary()`) which handle all of this.

10. **Mixing CQRS metrics with EDA metrics in the same class** -- Each `FireflyMetricsSupport` subclass must have exactly one module name passed to `super(registry, "module")`. If you need both CQRS and EDA metrics, inject `CommandMetricsService` and `EdaMetrics` separately. Do not create a single class that tries to serve both namespaces.
