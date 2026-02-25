---
name: implement-resilience
description: Use when adding resilience patterns (circuit breaker, retry, rate limiting, bulkhead) to a Firefly Framework service — covers Resilience4j configuration with reactive Reactor decorators
---

# Implementing Resilience Patterns in Firefly Framework

The framework uses two resilience approaches: Resilience4j with reactive Reactor operators for EDA publishers, and a custom `CircuitBreakerManager` with `AdvancedResilienceManager` for service clients.

## Architecture Overview

- **fireflyframework-eda** (`org.fireflyframework.eda.resilience`) -- `ResilientEventPublisher` wraps any `EventPublisher` with Resilience4j circuit breaker, retry, and rate limiter via Reactor `transformDeferred` operators.
- **fireflyframework-client** (`org.fireflyframework.client.resilience` and `org.fireflyframework.resilience`) -- `AdvancedResilienceManager` provides bulkhead isolation, token-bucket rate limiting, adaptive timeout, and load shedding. `CircuitBreakerManager` provides a custom sliding-window circuit breaker with reactive `Mono.defer` integration.

Decorator ordering in `ResilientEventPublisher.applyResilience`:
1. Rate Limiter (outermost -- reject early if throughput exceeded)
2. Circuit Breaker (fail fast if downstream is unhealthy)
3. Retry (innermost -- retry the actual operation)

---

## Circuit Breaker

### Resilience4j Circuit Breaker (EDA Module)

`ResilientEventPublisherFactory` creates Resilience4j `CircuitBreaker` instances from `EdaProperties.Resilience.CircuitBreaker` and registers them through `CircuitBreakerRegistry`.

```java
CircuitBreakerConfig circuitBreakerConfig = CircuitBreakerConfig.custom()
    .failureRateThreshold(50)
    .slowCallRateThreshold(50)
    .slowCallDurationThreshold(Duration.ofSeconds(60))
    .minimumNumberOfCalls(10)
    .slidingWindowSize(10)
    .waitDurationInOpenState(Duration.ofSeconds(60))
    .permittedNumberOfCallsInHalfOpenState(3)
    .build();

CircuitBreaker circuitBreaker = circuitBreakerRegistry
    .circuitBreaker("eda-publisher-" + publisherName, circuitBreakerConfig);
```

Apply to a reactive Mono with `transformDeferred` (checks state at subscription time, not assembly time):

```java
import io.github.resilience4j.reactor.circuitbreaker.operator.CircuitBreakerOperator;

Mono<Void> resilientOperation = operation
    .transformDeferred(CircuitBreakerOperator.of(circuitBreaker))
    .doOnError(ex -> log.debug("Circuit breaker blocked: destination={}", destination, ex));
```

### Custom Circuit Breaker (Client Module)

`CircuitBreakerManager` in `org.fireflyframework.resilience` provides a hand-rolled circuit breaker with `Mono.defer` integration for service client calls.

```java
import org.fireflyframework.resilience.CircuitBreakerConfig;
import org.fireflyframework.resilience.CircuitBreakerManager;

CircuitBreakerConfig config = CircuitBreakerConfig.builder()
    .failureRateThreshold(50.0)
    .minimumNumberOfCalls(5)
    .slidingWindowSize(10)
    .waitDurationInOpenState(Duration.ofSeconds(60))
    .permittedNumberOfCallsInHalfOpenState(3)
    .callTimeout(Duration.ofSeconds(10))
    .build();

CircuitBreakerManager manager = new CircuitBreakerManager(config);

Mono<MyResponse> result = manager.executeWithCircuitBreaker("payment-service", () ->
    webClient.get().uri("/api/payments/{id}", paymentId)
        .retrieve().bodyToMono(MyResponse.class));
```

Preset configurations:
```java
CircuitBreakerConfig.highAvailabilityConfig();  // 30% threshold, 3 min calls, 30s wait, 5s timeout
CircuitBreakerConfig.faultTolerantConfig();     // 70% threshold, 10 min calls, 2min wait, 15s timeout
```

### Circuit Breaker in RestClientBuilder

```java
ServiceClient client = ServiceClient.rest("payment-service")
    .baseUrl("https://payment-service:8443")
    .circuitBreakerManager(circuitBreakerManager)
    .build();
```

Internally, `RestServiceClientImpl` applies retry first (inner), then circuit breaker (outer):
```java
Mono<R> retriedRequest = applyRetry(baseRequest);
return applyCircuitBreakerProtection(retriedRequest);
```

---

## Retry

### Resilience4j Retry (EDA Module)

```java
RetryConfig retryConfig = RetryConfig.custom()
    .maxAttempts(3)
    .waitDuration(Duration.ofMillis(500))
    .retryExceptions(Exception.class)
    .build();

Retry retry = retryRegistry.retry("eda-publisher-" + publisherName, retryConfig);

Mono<Void> resilientOperation = operation
    .transformDeferred(RetryOperator.of(retry));
```

Selective retry -- retry on some exceptions, ignore others:
```java
RetryConfig retryConfig = RetryConfig.custom()
    .maxAttempts(3)
    .retryExceptions(RuntimeException.class)
    .ignoreExceptions(IllegalArgumentException.class)
    .build();
```

### Reactor Retry (Client Module)

`RestServiceClientImpl` uses Reactor's `Retry.backoff` with exponential backoff and jitter:

```java
Retry retrySpec = Retry.backoff(retryMaxAttempts, retryInitialBackoff)
    .maxBackoff(retryMaxBackoff)
    .jitter(retryJitterEnabled ? 0.5 : 0.0)
    .filter(throwable -> throwable instanceof RetryableError
            && ((RetryableError) throwable).isRetryable())
    .doBeforeRetry(signal -> log.warn(
        "Retrying '{}' (attempt {}/{}): {}",
        serviceName, signal.totalRetries() + 1, retryMaxAttempts,
        signal.failure().getMessage()));
```

Builder configuration:
```java
ServiceClient client = ServiceClient.rest("user-service")
    .baseUrl("http://user-service:8080")
    .retry(true, 3, Duration.ofMillis(500), Duration.ofSeconds(10), true)
    .build();

// Or disable:
ServiceClient client = ServiceClient.rest("user-service")
    .baseUrl("http://user-service:8080")
    .noRetry()
    .build();
```

Defaults: `retryEnabled=true`, `maxAttempts=3`, `initialBackoff=500ms`, `maxBackoff=10s`, `jitter=true`.

---

## Rate Limiting

### Resilience4j Rate Limiter (EDA Module)

Rate limiting is **disabled by default**. Enable through configuration.

```java
RateLimiterConfig rateLimiterConfig = RateLimiterConfig.custom()
    .limitForPeriod(100)
    .limitRefreshPeriod(Duration.ofSeconds(1))
    .timeoutDuration(Duration.ofSeconds(5))
    .build();

RateLimiter rateLimiter = rateLimiterRegistry
    .rateLimiter("eda-publisher-" + publisherName, rateLimiterConfig);

Mono<Void> resilientOperation = operation
    .transformDeferred(RateLimiterOperator.of(rateLimiter));
```

### Token Bucket Rate Limiter (Client Module)

`AdvancedResilienceManager` provides a custom token bucket rate limiter, checked synchronously before execution:

```java
AdvancedResilienceManager.ResilienceConfig config =
    new AdvancedResilienceManager.ResilienceConfig(
        10, Duration.ofSeconds(5), 100.0, 150, Duration.ofSeconds(5), Duration.ofSeconds(30));

Mono<MyResponse> result = advancedResilienceManager
    .applyResilience("payment-service", operation, config);
```

---

## Bulkhead

`AdvancedResilienceManager.BulkheadIsolation` provides semaphore-based bulkhead isolation with a bounded elastic scheduler:

```java
BulkheadIsolation bulkhead = new BulkheadIsolation(10, Duration.ofSeconds(5));

Mono<MyResponse> result = bulkhead.execute(() ->
    webClient.get().uri("/api/resource").retrieve().bodyToMono(MyResponse.class));
```

Internally uses `Semaphore.tryAcquire` with `doFinally` to guarantee permit release:
```java
public <T> Mono<T> execute(Supplier<Mono<T>> operation) {
    return Mono.fromCallable(() -> {
        if (!semaphore.tryAcquire(maxWaitTime.toMillis(), TimeUnit.MILLISECONDS)) {
            throw new BulkheadFullException("Bulkhead full, cannot acquire permit");
        }
        return true;
    })
    .subscribeOn(scheduler)
    .flatMap(acquired -> operation.get())
    .doFinally(signal -> semaphore.release());
}
```

---

## Load Shedding and Adaptive Timeout

`SystemLoadSheddingStrategy` monitors CPU, memory, thread pools, GC pressure, and per-service metrics:

```java
SystemLoadSheddingStrategy loadShedding = new SystemLoadSheddingStrategy(
    0.8, 0.9, 0.9, 5000, 1000);  // cpu, memory, threadPool, responseTimeMs, rps

AdvancedResilienceManager manager = new AdvancedResilienceManager(
    loadShedding, circuitBreakerManager);
```

`AdaptiveTimeout` adjusts timeouts based on historical performance: requires 10+ calls, calculates `avgResponseTime * (1 + failureRate * 2) * 2`, clamped between `baseTimeout` and `maxTimeout`.

---

## Combining All Patterns

### EDA Publisher (Resilience4j)

```java
private Mono<Void> applyResilience(Mono<Void> operation, ...) {
    Mono<Void> resilientOperation = operation;

    // 1. Rate limiting (outermost)
    if (resilienceConfig.getRateLimiter().isEnabled() && rateLimiter != null) {
        resilientOperation = resilientOperation
            .transformDeferred(RateLimiterOperator.of(rateLimiter));
    }
    // 2. Circuit breaker
    if (resilienceConfig.getCircuitBreaker().isEnabled() && circuitBreaker != null) {
        resilientOperation = resilientOperation
            .transformDeferred(CircuitBreakerOperator.of(circuitBreaker));
    }
    // 3. Retry (innermost)
    if (resilienceConfig.getRetry().isEnabled() && retry != null) {
        resilientOperation = resilientOperation
            .transformDeferred(RetryOperator.of(retry));
    }
    return resilientOperation;
}
```

The factory wraps any publisher automatically:
```java
EventPublisher resilientPublisher = resilientEventPublisherFactory
    .createResilientPublisher(basePublisher, "my-publisher");
```

Activated by `@ConditionalOnClass({CircuitBreaker.class, Retry.class, RateLimiter.class})` and `@ConditionalOnProperty(prefix = "firefly.eda.resilience", name = "enabled", havingValue = "true", matchIfMissing = true)`.

### Service Client (Custom)

```java
public <T> Mono<T> applyResilience(String serviceName, Mono<T> operation, ResilienceConfig config) {
    return Mono.defer(() -> {
        if (loadSheddingStrategy.shouldShedLoad(serviceName))       // 1. Load shedding
            return Mono.error(new LoadSheddingException(...));
        if (!getRateLimiter(serviceName, config).tryAcquire())      // 2. Rate limiting
            return Mono.error(new RateLimitExceededException(...));
        return getBulkhead(serviceName, config).execute(() -> {     // 3. Bulkhead
            Duration timeout = getAdaptiveTimeout(serviceName, config).calculateTimeout();
            Mono<T> op = operation.timeout(timeout);                // 4. Adaptive timeout
            if (circuitBreakerManager != null)                      // 5. Circuit breaker
                return circuitBreakerManager.executeWithCircuitBreaker(serviceName, () -> op);
            return op;
        });
    });
}
```

---

## Configuration Reference (YAML)

```yaml
firefly:
  eda:
    resilience:
      enabled: true                                     # master toggle (default: true)
      circuit-breaker:
        enabled: true                                   # default: true
        failure-rate-threshold: 50                      # percentage (default: 50)
        slow-call-rate-threshold: 50                    # percentage (default: 50)
        slow-call-duration-threshold: 60s               # default: 60s
        minimum-number-of-calls: 10                     # default: 10
        sliding-window-size: 10                         # default: 10
        wait-duration-in-open-state: 60s                # default: 60s
        permitted-number-of-calls-in-half-open-state: 3 # default: 3
      retry:
        enabled: true                                   # default: true
        max-attempts: 3                                 # default: 3
        wait-duration: 500ms                            # default: 500ms
        exponential-backoff-multiplier: 2.0             # default: 2.0
      rate-limiter:
        enabled: false                                  # default: false (opt-in)
        limit-for-period: 100                           # default: 100
        limit-refresh-period: 1s                        # default: 1s
        timeout-duration: 5s                            # default: 5s
```

Client module `CircuitBreakerConfig` defaults:

| Parameter | Default | Description |
|-----------|---------|-------------|
| `failureRateThreshold` | 50.0 | Percentage of failures to open the circuit |
| `minimumNumberOfCalls` | 5 | Minimum calls before evaluating failure rate |
| `slidingWindowSize` | 10 | Calls in the sliding window |
| `waitDurationInOpenState` | 60s | Time in OPEN before trying HALF_OPEN |
| `permittedNumberOfCallsInHalfOpenState` | 3 | Trial calls allowed in HALF_OPEN |
| `callTimeout` | 10s | Timeout for individual calls |
| `slowCallDurationThreshold` | 5s | Calls slower than this count as slow |
| `slowCallRateThreshold` | 100.0 | Slow call percentage to open (100 = disabled) |

---

## Health and Observability

`ResilientEventPublisher.getHealth()` enriches delegate health with resilience state:
- `circuitBreaker.state` -- CLOSED / OPEN / HALF_OPEN
- `circuitBreaker.failureRate` -- current failure rate percentage
- `rateLimiter.availablePermissions` -- permits remaining
- `rateLimiter.waitingThreads` -- threads waiting for permits

`isAvailable()` combines delegate status with circuit breaker state:
```java
public boolean isAvailable() {
    return delegate.isAvailable() &&
           (circuitBreaker == null || circuitBreaker.getState() != CircuitBreaker.State.OPEN);
}
```

---

## Testing Resilience Patterns

Use Reactor `StepVerifier` with Mockito. Key patterns from `ResiliencePatternTest`:

```java
// Test retry succeeds after transient failures
Mono<Void> retryMono = Mono.fromSupplier(() -> delegate.publish(event, "topic", headers))
    .flatMap(mono -> mono)
    .transformDeferred(RetryOperator.of(retry));
StepVerifier.create(retryMono).verifyComplete();

// Test circuit breaker opens after failure threshold
StepVerifier.create(operation.transformDeferred(CircuitBreakerOperator.of(cb)))
    .expectError(RuntimeException.class).verify();
assertThat(circuitBreaker.getState()).isEqualTo(CircuitBreaker.State.OPEN);

// Test combined patterns
Mono<Void> combined = operation
    .transformDeferred(RetryOperator.of(retry))
    .transformDeferred(CircuitBreakerOperator.of(cb))
    .transformDeferred(RateLimiterOperator.of(rl));
StepVerifier.create(combined).verifyComplete();
```

---

## Anti-Patterns

### 1. Applying resilience at assembly time instead of subscription time

```java
// BAD: State evaluated once when the Mono is built
if (circuitBreaker.getState() != CircuitBreaker.State.OPEN) { return operation; }

// GOOD: State evaluated each time the Mono is subscribed to
return operation.transformDeferred(CircuitBreakerOperator.of(circuitBreaker));
```

### 2. Wrong decorator ordering

```java
// BAD: Retry outermost means each retry counts against circuit breaker
operation.transformDeferred(CircuitBreakerOperator.of(cb))
    .transformDeferred(RetryOperator.of(retry));

// GOOD: Rate limiter > circuit breaker > retry
operation.transformDeferred(RateLimiterOperator.of(rl))
    .transformDeferred(CircuitBreakerOperator.of(cb))
    .transformDeferred(RetryOperator.of(retry));
```

### 3. Retrying non-retryable errors

```java
// BAD: Retry everything including validation errors
RetryConfig.custom().maxAttempts(3).retryExceptions(Exception.class).build();

// GOOD: Only retry transient failures
RetryConfig.custom().maxAttempts(3)
    .retryExceptions(RuntimeException.class)
    .ignoreExceptions(IllegalArgumentException.class, ValidationException.class).build();

// GOOD: Filter with retryable marker (client module)
Retry.backoff(3, Duration.ofMillis(500))
    .filter(t -> t instanceof RetryableError && ((RetryableError) t).isRetryable());
```

### 4. Not releasing bulkhead permits on error

```java
// BAD: Permits leak on error paths
semaphore.acquire();
return operation.get().doOnSuccess(r -> semaphore.release());

// GOOD: doFinally releases on success, error, AND cancellation
return Mono.fromCallable(() -> { semaphore.tryAcquire(...); return true; })
    .flatMap(acquired -> operation.get())
    .doFinally(signal -> semaphore.release());
```

### 5. Hardcoding resilience parameters

```java
// BAD: No ability to tune without redeployment
CircuitBreakerConfig.custom().failureRateThreshold(50).build();

// GOOD: Configuration-driven via EdaProperties
CircuitBreakerConfig.custom()
    .failureRateThreshold(config.getFailureRateThreshold())
    .slidingWindowSize(config.getSlidingWindowSize())
    .waitDurationInOpenState(config.getWaitDurationInOpenState())
    .build();
```

### 6. Missing circuit breaker state in availability checks

```java
// BAD: Reports available while circuit is open
public boolean isAvailable() { return delegate.isAvailable(); }

// GOOD: Combine delegate status with circuit breaker state
public boolean isAvailable() {
    return delegate.isAvailable() &&
           (circuitBreaker == null || circuitBreaker.getState() != CircuitBreaker.State.OPEN);
}
```

---

## Key Files Reference

| File | Module | Purpose |
|------|--------|---------|
| `ResilientEventPublisher.java` | eda | Decorator adding Resilience4j patterns to any EventPublisher |
| `ResilientEventPublisherFactory.java` | eda | Factory creating resilient publisher wrappers with registry support |
| `EdaProperties.java` | eda | Configuration properties including resilience settings |
| `CircuitBreakerManager.java` | client | Custom circuit breaker with sliding window and reactive integration |
| `CircuitBreakerConfig.java` | client | Builder-pattern config with preset profiles |
| `AdvancedResilienceManager.java` | client | Bulkhead, rate limiter, adaptive timeout, and load shedding |
| `RestClientBuilder.java` | client | Fluent builder wiring circuit breaker and retry into REST clients |
| `RestServiceClientImpl.java` | client | REST client impl with retry-then-circuit-breaker execution |
