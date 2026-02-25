---
name: debugging-reactive-chains
description: Use when debugging errors in reactive chains in Firefly Framework services — covers Reactor debugging techniques including log(), checkpoint(), Hooks.onOperatorDebug, reactive stack traces, and common errors (empty signal, timeout, backpressure)
---

# Debugging Reactive Chains

## When to Use

Use this guide when you encounter errors inside reactive pipelines in Firefly Framework services. Reactive stack traces are unhelpful because the exception is thrown on a different thread from the one that assembled the chain.

Typical symptoms: a `Mono`/`Flux` completes empty unexpectedly, a timeout fires with no clear source, a `CommandProcessingException` or `EdaException` wraps a cause pointing to framework internals, backpressure errors appear in EDA/orchestration, saga steps fail with hidden root causes, or context (correlation ID, tenant ID) disappears mid-chain.

---

## 1 -- Reactor Debugging Fundamentals

### 1.1 -- The `log()` Operator

Insert `.log()` at any point to print every signal (onSubscribe, onNext, onError, onComplete, cancel, request).

```java
commandBus.send(command)
    .log("CreateAccount")
    .subscribe();

// Controlled logging -- specific level and signals only
repository.findById(id)
    .log("AccountLookup", Level.FINE, SignalType.ON_NEXT, SignalType.ON_ERROR)
    .flatMap(account -> processAccount(account))
    .subscribe();
```

Give every `log()` a unique label. Restrict signals in production-adjacent profiles to avoid `request(n)` noise. Remove before merging.

### 1.2 -- The `checkpoint()` Operator

Annotates the assembly stack trace so errors include a human-readable location.

```java
commandBus.send(command)
    .checkpoint("after-command-send")
    .flatMap(result -> publishEvent(result))
    .checkpoint("after-event-publish")
    .subscribe();
```

Error output includes: `checkpoint(after-event-publish)`. Use `checkpoint(description, true)` to capture the full assembly trace (expensive, active debugging only).

### 1.3 -- `Hooks.onOperatorDebug()` vs `ReactorDebugAgent`

`Hooks.onOperatorDebug()` enables global assembly tracing. **Never use in production** -- 10-30x performance degradation.

```java
// DEVELOPMENT ONLY -- call once at startup
Hooks.onOperatorDebug();
```

`ReactorDebugAgent` uses bytecode instrumentation with near-zero overhead. Safe for production.

```xml
<dependency>
    <groupId>io.projectreactor</groupId>
    <artifactId>reactor-tools</artifactId>
</dependency>
```

```java
// Safe for production -- call once at startup
ReactorDebugAgent.init();
```

### 1.4 -- `doOn*` Side-Effect Operators for Tracing

```java
repository.findById(accountId)
    .doOnSubscribe(sub -> log.debug("Subscribing to findById for {}", accountId))
    .doOnNext(account -> log.debug("Found account: {}", account.getId()))
    .doOnError(err -> log.error("findById failed: {}", err.getMessage()))
    .doOnCancel(() -> log.warn("findById was cancelled for {}", accountId))
    .doFinally(signal -> log.debug("findById signal: {}", signal));
```

---

## 2 -- Common Reactive Errors and Solutions

### 2.1 -- Empty Mono / "No value present"

**Symptom**: `Mono` completes without emitting; downstream `.single()` or `switchIfEmpty` expectation fails.

**Firefly root causes**: handler returns `Mono.empty()` instead of `Mono.just(result)`, `Mono.defer()` lambda returns null, authorization step returns `Mono.empty()` swallowed by `.then()`.

```java
// Diagnose with log()
queryBus.query(getAccountQuery)
    .log("QueryResult")  // see if onNext fires or only onComplete
    .subscribe();

// Fix: always provide switchIfEmpty
@Override
protected Mono<AccountBalance> doHandle(GetAccountBalanceQuery query) {
    return accountRepository.findByAccountId(query.getAccountId())
        .map(account -> new AccountBalance(account.getId(), account.getBalance()))
        .switchIfEmpty(Mono.error(
            new FireflyException("Account not found: " + query.getAccountId(), "ACCOUNT_NOT_FOUND")));
}
```

### 2.2 -- Timeout Errors

**Firefly sources**: `EventListenerProcessor` applies 30-second global timeout, per-listener `@EventListener(timeoutMs = ...)`, saga `ExecutionTimeoutException`, unprotected external calls.

**Fix**: Set explicit layered timeouts -- inner operations shorter than outer ones:

```java
public Mono<TransferResult> doHandle(TransferMoneyCommand cmd) {
    return validateAccounts(cmd)
        .timeout(Duration.ofSeconds(2))       // inner: validation
        .flatMap(valid -> executeTransfer(cmd))
        .timeout(Duration.ofSeconds(5))       // middle: transfer
        .flatMap(result -> notifyParties(cmd, result))
        .timeout(Duration.ofSeconds(3));      // inner: notification
}
```

### 2.3 -- Backpressure Overflow

**Symptom**: `OverflowException: Could not emit buffer due to lack of requests`.

**Firefly sources**: orchestration `BackpressureStrategy` misconfigured, EDA consumers outpacing listeners, unbounded `Flux.flatMap()` concurrency.

```java
// Fix: limit concurrency and buffer
Flux.fromIterable(accounts)
    .flatMap(account -> processAccount(account), 8)  // max 8 concurrent
    .subscribe();

eventFlux
    .onBackpressureBuffer(256, dropped -> log.warn("Dropped event: {}", dropped))
    .flatMap(event -> processEvent(event), 4)
    .subscribe();
```

### 2.4 -- Context Loss Across Scheduler Boundaries

**Root cause**: `CorrelationContext` uses `ThreadLocal`. Switching threads via `subscribeOn`/`publishOn` loses values. Also, `DefaultCommandBus` calls `correlationContext.clear()` in `doFinally`.

```java
// WRONG -- context cleared by doFinally inside DefaultCommandBus
commandBus.send(command)
    .doOnNext(r -> log.info("CorrId: {}", correlationContext.getCorrelationId()))  // null

// RIGHT -- capture before sending
String corrId = command.getCorrelationId();
commandBus.send(command)
    .doOnNext(r -> log.info("CorrId: {}", corrId))

// OR use Reactor Context for cross-scheduler propagation
return commandBus.send(command)
    .contextWrite(ctx -> ctx.put("correlationId", correlationId))
    .doOnEach(signal -> signal.getContextView().getOrEmpty("correlationId")
        .ifPresent(id -> correlationContext.setCorrelationId((String) id)));
```

### 2.5 -- "Operator called default onErrorDropped"

An error occurs after the subscriber already received a terminal signal. Install a hook to surface these:

```java
Hooks.onErrorDropped(error ->
    log.error("Dropped error (investigate chain): {}", error.getMessage(), error));
```

Fix: ensure fallback paths do not throw. Wrap risky fallbacks in `Mono.defer` with their own `onErrorResume`.

---

## 3 -- Firefly CQRS Debugging

### 3.1 -- CommandBus Pipeline Errors

`DefaultCommandBus.send()` pipeline: handler lookup -> correlation context -> validation -> authorization -> execution -> domain event publish -> metrics. Each step has a distinct exception:

| Step | Exception | Check |
|------|-----------|-------|
| Handler lookup | `CommandHandlerNotFoundException` | `@CommandHandlerComponent` annotation? Package scanned? Generic types exact? |
| Validation | `ValidationException` | Jakarta annotations? `customValidate()` returns proper `ValidationResult`? |
| Authorization | `AuthorizationException` | `authorize()` on command? `AuthorizationService` configured? |
| Execution | `CommandProcessingException` | Call `getCause()` for real error. `getProcessingTime()` for slow queries. |
| Event publish | Warning log only | `commandEventPublisher` wired? `@PublishDomainEvent` destination correct? |

```java
// Extract full context from CommandProcessingException
commandBus.send(command)
    .doOnError(CommandProcessingException.class, ex -> {
        log.error("Command failed: type={}, id={}, duration={}ms, rootCause={}",
            ex.getCommandType(), ex.getCommandId(),
            ex.getProcessingTime() != null ? ex.getProcessingTime().toMillis() : "?",
            ex.getCause() != null ? ex.getCause().getClass().getSimpleName() : "none");
    }).subscribe();
```

Check startup logs: `DefaultCommandBus ready with N registered handlers`. If your handler is missing, verify it extends `CommandHandler<YourCommand, YourResult>` with exact generic types.

### 3.2 -- QueryBus Pipeline Errors

`DefaultQueryBus.query()` wraps unrecognized errors in `QueryProcessingException`. If caching is enabled and the handler returns empty, the `switchIfEmpty` re-executes the handler. Add logging in the handler to distinguish cache miss from genuine empty.

---

## 4 -- Firefly EDA Debugging

### 4.1 -- Event Publishing Failures

`EventPublisher.publish()` returns `Mono<Void>`. Not subscribing means errors are silently lost.

```java
// WRONG -- fire and forget
eventPublisher.publish(event, "account-events");

// RIGHT -- compose into the reactive chain
return processCommand(command)
    .flatMap(result -> eventPublisher.publish(event, "account-events").thenReturn(result));
```

### 4.2 -- Resilience-Wrapped Publishers

`ResilientEventPublisher` applies circuit breaker, retry, and rate limiter (Resilience4j). Check health when publishes fail:

```java
eventPublisher.getHealth()
    .doOnNext(health -> log.info("Status: {}, details: {}", health.getStatus(), health.getDetails()))
    .subscribe();
```

### 4.3 -- Event Listener Error Strategies

The `EventListenerProcessor` handles errors per `@EventListener(errorStrategy = ...)`:

| Strategy | Behavior |
|----------|----------|
| `LOG_AND_CONTINUE` | Default. Logs error, acknowledges message. |
| `LOG_AND_RETRY` | Retries with backoff per `maxRetries`/`retryDelayMs`. |
| `REJECT_AND_STOP` | Propagates error, may stop consumer. |
| `DEAD_LETTER` | Forwards to `<destination>.dlq`. |
| `IGNORE` | Silently acknowledges. |
| `CUSTOM` | Delegates to `CustomErrorHandler` beans in priority order. |

If a listener silently fails, look for: `ERROR ... Error in event listener: MyListener.handleEvent, strategy: LOG_AND_CONTINUE`.

### 4.4 -- Serialization and Dead Letter Queue Errors

`SerializationException` (extends `EdaException`): check that event classes have no-arg constructors, compatible field types, and matching schemas (Avro/Protobuf). When DLQ publishing fails: `Dead letter queue publishing failed, event will be lost` -- verify broker connectivity and DLQ destination exists.

---

## 5 -- Saga / Orchestration Debugging

### 5.1 -- Step Execution Failures

`StepExecutionException` includes the `stepId`. Always check `getCause()`:

```java
sagaEngine.execute(mySaga, input)
    .doOnError(StepExecutionException.class, ex ->
        log.error("Step '{}' failed: {}", ex.getStepId(), ex.getCause().getMessage()))
    .subscribe();
```

### 5.2 -- Compensation Failures

`CompensationException` means the system may be partially compensated. Treat as critical:

```java
.doOnError(CompensationException.class, ex -> {
    log.error("CRITICAL: Compensation failed. Context: {}", ex.getContext());
    alertService.sendCriticalAlert("Saga compensation failure", ex);
})
```

### 5.3 -- Execution Timeouts

`ExecutionTimeoutException` message includes execution name, instance ID, and duration. Debug with per-step checkpoints and explicit step timeouts:

```java
@SagaStep(order = 1)
public Mono<DebitResult> debitAccount(TransferInput input) {
    return accountService.debit(input.getSourceAccountId(), input.getAmount())
        .checkpoint("saga-step-debitAccount")
        .timeout(Duration.ofSeconds(10));
}
```

---

## 6 -- Debugging in Tests

### 6.1 -- StepVerifier

```java
@Test
void shouldFailWhenAccountNotFound() {
    StepVerifier.create(queryBus.query(new GetAccountBalanceQuery("nonexistent")))
        .expectErrorMatches(err -> err instanceof FireflyException
            && err.getMessage().contains("Account not found"))
        .verify(Duration.ofSeconds(5));
}
```

### 6.2 -- Virtual Time for Timeouts

```java
@Test
void shouldTimeoutSlowCall() {
    StepVerifier.withVirtualTime(() ->
        externalService.fetchAccount("acc-1").timeout(Duration.ofSeconds(5)))
        .expectSubscription()
        .thenAwait(Duration.ofSeconds(5))
        .expectError(TimeoutException.class)
        .verify();
}
```

### 6.3 -- Debugging Empty Chains

```java
StepVerifier.create(suspiciousMethod().log("TEST-DEBUG").checkpoint("test-checkpoint"))
    .expectNextCount(1)
    .verifyComplete();
```

---

## 7 -- Anti-Patterns

### 7.1 -- Blocking Inside a Reactive Chain

```java
// WRONG -- deadlock risk
Account account = repository.findById(id).block();

// RIGHT -- stay reactive
return repository.findById(id).map(account -> process(account));

// If blocking is unavoidable, isolate on boundedElastic
Mono.fromCallable(() -> legacyService.call(param))
    .subscribeOn(Schedulers.boundedElastic());
```

### 7.2 -- Swallowing Errors

```java
// WRONG
.onErrorResume(err -> Mono.empty());

// RIGHT -- handle specific types, propagate unknowns
.onErrorResume(ValidationException.class, err -> Mono.error(err))
.onErrorResume(FireflyInfrastructureException.class, err -> fallbackMono());
```

### 7.3 -- Nested Subscribe

```java
// WRONG -- fire-and-forget inside doOnNext, errors lost
.doOnNext(result -> eventPublisher.publish(event, "topic").subscribe());

// RIGHT -- compose with flatMap
.flatMap(result -> eventPublisher.publish(event, "topic").thenReturn(result));
```

### 7.4 -- Ignoring Mono Return Values

```java
// WRONG -- never executes
repository.deleteById(accountId);

// RIGHT -- return the Mono
return repository.deleteById(accountId);
```

### 7.5 -- Losing Reactor Context

```java
// WRONG -- independent subscription loses context
inner.subscribe(val -> log.info("Tenant: {}", val));

// RIGHT -- compose so context propagates downstream-to-upstream
return outer.flatMap(result -> Mono.deferContextual(ctx -> {
    String tenantId = ctx.getOrDefault("tenantId", "unknown");
    return processForTenant(result, tenantId);
})).contextWrite(Context.of("tenantId", "tenant-abc"));
```

---

## 8 -- Firefly Exception Hierarchy

```
RuntimeException
  └── FireflyException                          [kernel] errorCode, context map
        ├── FireflyInfrastructureException      [kernel] DB, cache, messaging, network
        │     └── EdaException                  [eda]    event-driven errors
        │           └── SerializationException  [eda]    ser/deser failures
        ├── FireflySecurityException            [kernel] security errors
        │     └── AuthorizationException        [cqrs]   auth failures
        ├── ValidationException                 [cqrs]   validation failures
        ├── CommandProcessingException          [cqrs]   command failures (with context)
        ├── CommandHandlerNotFoundException     [cqrs]   missing handler
        └── OrchestrationException (sealed)     [orchestration]
              ├── StepExecutionException         step failures
              ├── CompensationException          compensation failures
              ├── ExecutionTimeoutException       saga timeout
              ├── ExecutionNotFoundException      instance not found
              ├── TopologyValidationException     invalid topology
              ├── DuplicateDefinitionException    duplicate definition
              └── TccPhaseException              TCC phase failures
```

Always check `getCause()` on wrapper exceptions -- the real error is in the cause chain.

---

## 9 -- Debugging Checklist

1. **Read the full exception chain.** Call `getCause()` recursively. The leaf cause is usually the real problem.
2. **Check the error code.** `FireflyException.getErrorCode()` and `CommandProcessingException.getCommandType()` narrow the search.
3. **Add `checkpoint()` operators** around the suspected area for assembly trace info.
4. **Add `.log()` operators** to see which signals fire and where the chain stops.
5. **Verify subscriptions.** Every `Mono`/`Flux` must be returned, composed, or explicitly subscribed.
6. **Check scheduler boundaries.** Null correlation context or MDC means you crossed an async boundary.
7. **Check timeouts.** EventListenerProcessor: 30s global. Saga steps: configurable. Set inner timeouts shorter than outer.
8. **Use `StepVerifier`** to reproduce failures in controlled tests.
9. **Enable `ReactorDebugAgent`** if stack traces lack assembly-line information.
10. **Check resilience wrappers.** Circuit breaker state and rate limiter via `publisher.getHealth()`.
