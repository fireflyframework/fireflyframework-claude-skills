---
name: testing-sagas
description: Use when writing tests for sagas and compensation flows -- covers happy path tests, compensation at each step, compensation policy tests, external service mocks, and saga persistence verification
---

# Testing Sagas and Compensation Flows

## 1. Test Infrastructure Setup

Every saga test requires a `SagaEngine` built from lightweight components -- no Spring context needed.

### Imports and engine factory

```java
import org.fireflyframework.orchestration.core.argument.ArgumentResolver;
import org.fireflyframework.orchestration.core.context.ExecutionContext;
import org.fireflyframework.orchestration.core.event.NoOpEventPublisher;
import org.fireflyframework.orchestration.core.model.*;
import org.fireflyframework.orchestration.core.observability.OrchestrationEvents;
import org.fireflyframework.orchestration.core.persistence.*;
import org.fireflyframework.orchestration.core.step.*;
import org.fireflyframework.orchestration.saga.builder.SagaBuilder;
import org.fireflyframework.orchestration.saga.compensation.*;
import org.fireflyframework.orchestration.saga.compensation.CompensationErrorHandler.CompensationErrorResult;
import org.fireflyframework.orchestration.saga.engine.*;
import org.fireflyframework.orchestration.saga.registry.SagaDefinition;
import reactor.core.publisher.Mono;
import reactor.test.StepVerifier;
import static org.assertj.core.api.Assertions.*;
```

```java
private SagaEngine createEngine() {
    return createEngine(CompensationPolicy.STRICT_SEQUENTIAL);
}

private SagaEngine createEngine(CompensationPolicy policy) {
    var events = new OrchestrationEvents() {};
    var stepInvoker = new StepInvoker(new ArgumentResolver());
    var noOpPublisher = new NoOpEventPublisher();
    var orchestrator = new SagaExecutionOrchestrator(stepInvoker, events, noOpPublisher);
    var compensator = new SagaCompensator(events, policy, stepInvoker);
    return new SagaEngine(null, events, orchestrator, null, null, compensator, noOpPublisher);
}

// Variant with custom CompensationErrorHandler
private SagaEngine createEngine(CompensationPolicy policy, CompensationErrorHandler handler) {
    var events = new OrchestrationEvents() {};
    var stepInvoker = new StepInvoker(new ArgumentResolver());
    var noOpPublisher = new NoOpEventPublisher();
    var orchestrator = new SagaExecutionOrchestrator(stepInvoker, events, noOpPublisher);
    var compensator = new SagaCompensator(events, policy, stepInvoker, handler);
    return new SagaEngine(null, events, orchestrator, null, null, compensator, noOpPublisher);
}
```

- `NoOpEventPublisher` silences event publishing. Pass `null` for registry, persistence, and DLQ to keep tests minimal.
- Add persistence or DLQ only when testing those features specifically.

### Reusable compensating step handler

```java
private static StepHandler<Object, String> compensatingHandler(String id, List<String> tracker) {
    return new StepHandler<>() {
        @Override
        public Mono<String> execute(Object input, ExecutionContext ctx) {
            return Mono.just(id);
        }
        @Override
        public Mono<Void> compensate(String result, ExecutionContext ctx) {
            tracker.add(id);
            return Mono.empty();
        }
    };
}
```

## 2. Happy Path Tests

All assertions go inside `StepVerifier.create(...).assertNext(result -> { ... }).verifyComplete()`.

```java
@Test
void execute_successfulSaga_completesAllSteps() {
    SagaDefinition saga = SagaBuilder.saga("OrderSaga")
            .step("reserve").handler((StepHandler<Object, String>) (input, ctx) -> Mono.just("reserved")).add()
            .step("charge").dependsOn("reserve")
                .handler((StepHandler<Object, String>) (input, ctx) -> Mono.just("charged")).add()
            .build();

    StepVerifier.create(engine.execute(saga, StepInputs.empty()))
            .assertNext(result -> {
                assertThat(result.isSuccess()).isTrue();
                assertThat(result.sagaName()).isEqualTo("OrderSaga");
                assertThat(result.steps()).containsKeys("reserve", "charge");
                assertThat(result.steps().get("reserve").status()).isEqualTo(StepStatus.DONE);
                assertThat(result.steps().get("charge").status()).isEqualTo(StepStatus.DONE);
            })
            .verifyComplete();
}
```

**Input propagation** -- use `StepInputs.of(stepId, value)` and verify with `result.resultOf(stepId, Type.class)`:

```java
StepInputs inputs = StepInputs.of("greet", "World");
// In handler: (input, ctx) -> Mono.just("Hello " + input)
// Assert: assertThat(result.resultOf("greet", String.class)).hasValue("Hello World");
```

**Parallel steps** -- steps without mutual `dependsOn` run concurrently in the same layer. Use a synchronized list to track order, then assert the join step runs after all parallel predecessors:

```java
assertThat(order.indexOf("join")).isGreaterThan(order.indexOf("a"));
assertThat(order.indexOf("join")).isGreaterThan(order.indexOf("b"));
```

**Handler overloads** -- `SagaBuilder` supports `handler(() -> Mono.just(...))` (supplier), `handlerCtx(ctx -> ...)` (context-only), and `handlerInput(input -> ...)`. The `ctx.getResult("stepId")` method retrieves upstream step results.

## 3. Compensation Tests Per Step

### Core pattern: failing step triggers compensation of prior steps

```java
@Test
void execute_failingStep_triggersCompensation() {
    AtomicBoolean compensated = new AtomicBoolean(false);
    StepHandler<Object, String> reserveHandler = new StepHandler<>() {
        @Override public Mono<String> execute(Object input, ExecutionContext ctx) { return Mono.just("reserved"); }
        @Override public Mono<Void> compensate(String result, ExecutionContext ctx) {
            compensated.set(true);
            return Mono.empty();
        }
    };

    SagaDefinition saga = SagaBuilder.saga("FailingSaga")
            .step("reserve").handler(reserveHandler).add()
            .step("charge").dependsOn("reserve")
                .handler((StepHandler<Object, String>) (input, ctx) ->
                        Mono.error(new RuntimeException("payment failed"))).add()
            .build();

    StepVerifier.create(engine.execute(saga, StepInputs.empty()))
            .assertNext(result -> {
                assertThat(result.isSuccess()).isFalse();
                assertThat(result.failedSteps()).contains("charge");
                assertThat(result.error()).isPresent();
                assertThat(result.error().get().getMessage()).isEqualTo("payment failed");
                assertThat(compensated.get()).isTrue();
                assertThat(result.compensatedSteps()).contains("reserve");
            })
            .verifyComplete();
}
```

**Compensation receives the original step result** -- the `compensate(String result, ...)` parameter is the value returned by `execute()`. Capture it in a list to verify the correct result is passed.

**Single step failure** -- when the only step fails, `compensatedSteps()` is empty (nothing to compensate).

### Systematic per-step failure testing

Build a helper that constructs a chain with a configurable failure point:

```java
private SagaDefinition buildChainWithFailureAt(String failAtStep, List<String> compensated) {
    var builder = SagaBuilder.saga("ChainSaga");
    String[] steps = {"validate", "create-profile", "create-account", "send-welcome"};
    String prev = null;
    for (String stepId : steps) {
        var stepBuilder = builder.step(stepId);
        if (prev != null) stepBuilder.dependsOn(prev);
        if (stepId.equals(failAtStep)) {
            stepBuilder.handler((StepHandler<Object, String>) (input, ctx) ->
                    Mono.error(new RuntimeException(stepId + " failed")));
        } else {
            stepBuilder.handler(compensatingHandler(stepId, compensated));
        }
        stepBuilder.add();
        prev = stepId;
    }
    return builder.build();
}
```

Then write one test per step: failure at step N compensates steps 1..N-1 in reverse order.

## 4. Compensation Policy Tests

Create the engine with each policy and verify the compensation ordering and behavior.

### STRICT_SEQUENTIAL (default) -- reverse completion order

```java
SagaEngine engine = createEngine(CompensationPolicy.STRICT_SEQUENTIAL);
// Given: first -> second -> third (third fails)
// Assert: compensationOrder == ["second", "first"]
assertThat(compensationOrder).containsExactly("second", "first");
```

### GROUPED_PARALLEL -- reverse layers, concurrent within layer

```java
SagaEngine engine = createEngine(CompensationPolicy.GROUPED_PARALLEL);
// Given: s1 and s2 in layer 0 (parallel), s3 in layer 1 (fails)
// Assert: s1 and s2 both compensated, order non-deterministic
assertThat(compensationOrder).containsExactlyInAnyOrder("s1", "s2");
```

### RETRY_WITH_BACKOFF -- retries failing compensation

```java
SagaEngine engine = createEngine(CompensationPolicy.RETRY_WITH_BACKOFF);
// Step config: .compensationRetry(3).compensationBackoff(Duration.ofMillis(10))
// Compensation fails on first attempt, succeeds on second
assertThat(attempts.get()).isGreaterThanOrEqualTo(2);
assertThat(result.compensatedSteps()).contains("retryable");
```

### CIRCUIT_BREAKER -- halts on compensationCritical failure

```java
SagaEngine engine = createEngine(CompensationPolicy.CIRCUIT_BREAKER);
// Step config: .compensationCritical(true) on the critical step
// Critical step's compensation fails -> circuit opens -> s1 NOT compensated
assertThat(compensated).doesNotContain("s1");
```

### BEST_EFFORT_PARALLEL -- all compensations fire concurrently, errors swallowed

```java
SagaEngine engine = createEngine(CompensationPolicy.BEST_EFFORT_PARALLEL);
// All completed steps compensated concurrently, no ordering guarantee
assertThat(compensationOrder).containsExactlyInAnyOrder("a", "b");
```

## 5. Compensation Error Handler Tests

### Built-in handlers (unit tests, no engine needed)

```java
assertThat(new FailFastErrorHandler().handle("s", "s1", new RuntimeException(), 0))
        .isEqualTo(CompensationErrorResult.FAIL_SAGA);

assertThat(new LogAndContinueErrorHandler().handle("s", "s1", new RuntimeException(), 0))
        .isEqualTo(CompensationErrorResult.CONTINUE);

var retry = new RetryWithBackoffErrorHandler(3);
assertThat(retry.handle("s", "s1", new RuntimeException(), 0)).isEqualTo(CompensationErrorResult.RETRY);
assertThat(retry.handle("s", "s1", new RuntimeException(), 3)).isEqualTo(CompensationErrorResult.FAIL_SAGA);

// With exception type filtering
var filtered = new RetryWithBackoffErrorHandler(3, Set.of(IOException.class));
assertThat(filtered.handle("s", "s1", new IOException(), 0)).isEqualTo(CompensationErrorResult.RETRY);
assertThat(filtered.handle("s", "s1", new RuntimeException(), 0)).isEqualTo(CompensationErrorResult.FAIL_SAGA);
```

### CompositeCompensationErrorHandler

```java
var composite = new CompositeCompensationErrorHandler(List.of(
        new LogAndContinueErrorHandler(),   // returns CONTINUE -- falls through
        new FailFastErrorHandler()          // returns FAIL_SAGA -- stops here
));
assertThat(composite.handle("s", "s1", new RuntimeException(), 0))
        .isEqualTo(CompensationErrorResult.FAIL_SAGA);
```

### Wiring handlers into the engine (all 5 result codes)

**SKIP_STEP** -- compensation error is skipped, subsequent compensations continue:

```java
CompensationErrorHandler handler = (saga, step, err, attempt) -> CompensationErrorResult.SKIP_STEP;
SagaEngine engine = createEngine(CompensationPolicy.STRICT_SEQUENTIAL, handler);
// s2's compensation fails but is skipped; s1's compensation succeeds
assertThat(compensated).contains("s1");
```

**MARK_COMPENSATED** -- step is marked compensated despite the error:

```java
CompensationErrorHandler handler = (saga, step, err, attempt) -> CompensationErrorResult.MARK_COMPENSATED;
// s1's compensation fails but is marked compensated anyway
assertThat(result.compensatedSteps()).contains("s1");
```

**RETRY** -- compensation is re-attempted once:

```java
CompensationErrorHandler handler = (saga, step, err, attempt) -> CompensationErrorResult.RETRY;
// Compensation fails first time, succeeds on retry
assertThat(compensationAttempts.get()).isEqualTo(2);
assertThat(result.compensatedSteps()).contains("s1");
```

**FAIL_SAGA** -- the compensation error propagates as the saga error:

```java
CompensationErrorHandler handler = (saga, step, err, attempt) -> CompensationErrorResult.FAIL_SAGA;
assertThat(result.isFailed()).isTrue();
assertThat(result.error().get().getMessage()).isEqualTo("compensation exploded");
```

## 6. External Service Mocking

When steps call external services, mock them at the boundary with Mockito.

```java
@ExtendWith(MockitoExtension.class)
class CustomerRegistrationSagaTest {
    @Mock CommandBus commandBus;
    @Mock QueryBus queryBus;

    @Test
    void accountCreationFails_compensatesProfile() {
        when(queryBus.send(any(ValidateCustomerQuery.class)))
                .thenReturn(Mono.just(new CustomerValidationResult(true, "CUST-1", List.of())));
        when(commandBus.send(any(CreateCustomerProfileCommand.class)))
                .thenReturn(Mono.just(new CustomerProfileResult("CUST-1", "PROF-1")));
        when(commandBus.send(any(CreateAccountCommand.class)))
                .thenReturn(Mono.error(new RuntimeException("account service unavailable")));
        when(commandBus.send(any(DeleteCustomerProfileCommand.class)))
                .thenReturn(Mono.empty());

        // Build saga with mocked service calls in handlers...
        // Assert: result.failedSteps() contains "create-account"
        // Assert: result.compensatedSteps() contains "create-profile"
        verify(commandBus).send(any(DeleteCustomerProfileCommand.class));
        verify(commandBus, never()).send(any(CloseAccountCommand.class));
    }
}
```

### @CompensationSagaStep wiring verification

Mock `ApplicationContext` to verify that `SagaRegistry` correctly wires external compensation beans:

```java
@Test
void externalCompensation_wiredToCorrectStep() {
    var appCtx = mock(ApplicationContext.class);
    when(appCtx.getBeansWithAnnotation(Saga.class)).thenReturn(Map.of("saga", sagaBean));
    when(appCtx.getBeansOfType(Object.class))
            .thenReturn(Map.of("saga", sagaBean, "comp", compBean));

    var registry = new SagaRegistry(appCtx);
    var stepDef = registry.getSaga("MySaga").steps.get("processOrder");
    assertThat(stepDef.compensateMethod.getName()).isEqualTo("undoProcessOrder");
    assertThat(stepDef.compensateBean).isSameAs(compBean);
}
```

Unknown saga or step references throw `IllegalStateException` with descriptive messages.

## 7. Persistence Verification

### Checkpoint save counting

Wrap `InMemoryPersistenceProvider` in a delegating implementation that counts saves:

```java
@Test
void checkpoint_savedAfterEachStepSuccess() {
    InMemoryPersistenceProvider persistence = new InMemoryPersistenceProvider();
    AtomicInteger saveCount = new AtomicInteger(0);
    ExecutionPersistenceProvider tracking = new DelegatingProvider(persistence) {
        @Override public Mono<Void> save(ExecutionState state) {
            saveCount.incrementAndGet();
            return super.save(state);
        }
    };

    var orchestrator = new SagaExecutionOrchestrator(stepInvoker, events, noOpPublisher, tracking);
    var engine = new SagaEngine(null, events, orchestrator, tracking, null, compensator, noOpPublisher);

    // 3-step saga
    StepVerifier.create(engine.execute(saga, StepInputs.empty()))
            .assertNext(result -> {
                assertThat(result.isSuccess()).isTrue();
                // 1 initial + 3 step checkpoints + 1 final = 5
                assertThat(saveCount.get()).isEqualTo(5);
            })
            .verifyComplete();
}
```

### Checkpoint content verification

Capture `ExecutionState` objects in a list, then verify step results and statuses accumulate:

```java
ExecutionState afterStep1 = savedStates.get(1);
assertThat(afterStep1.stepResults()).containsKey("step1");
assertThat(afterStep1.stepStatuses().get("step1")).isEqualTo(StepStatus.DONE);

ExecutionState afterStep2 = savedStates.get(2);
assertThat(afterStep2.stepResults()).containsKeys("step1", "step2");
```

### Null persistence -- saga still works

Use the 3-arg `SagaExecutionOrchestrator` constructor (no persistence). The saga must complete without errors:

```java
var orchestrator = new SagaExecutionOrchestrator(stepInvoker, events, noOpPublisher);
// engine with null persistence -- verify saga completes successfully
```

### InMemoryPersistenceProvider unit tests

```java
// save + findById
StepVerifier.create(provider.save(state).then(provider.findById("1")))
        .assertNext(opt -> assertThat(opt).isPresent()).verifyComplete();

// updateStatus
StepVerifier.create(provider.save(state)
        .then(provider.updateStatus("1", ExecutionStatus.COMPLETED))
        .then(provider.findById("1")))
        .assertNext(opt -> assertThat(opt.get().status()).isEqualTo(ExecutionStatus.COMPLETED))
        .verifyComplete();

// findInFlight returns only RUNNING
// cleanup removes terminal entries older than threshold
```

## 8. StepEvent and Lifecycle Callback Tests

**StepEvent config** -- verify via `def.steps.get("stepId").stepEvent`:

```java
SagaDefinition def = SagaBuilder.saga("event-saga")
        .step("reserve").handler(() -> Mono.just("reserved"))
            .stepEvent("inventory-events", "InventoryReserved", "orderId").add()
        .build();

assertThat(def.steps.get("reserve").stepEvent.topic()).isEqualTo("inventory-events");
assertThat(def.steps.get("reserve").stepEvent.type()).isEqualTo("InventoryReserved");
```

**Lifecycle callbacks** -- create a `SagaDefinition` directly with a bean that has `@OnSagaComplete` / `@OnSagaError` methods. Set `saga.onSagaCompleteMethods` and `saga.onSagaErrorMethods` via reflection. Verify the callbacks receive `ExecutionContext` and `SagaResult`.

## 9. ExpandEach (Fan-out) Tests

```java
StepInputs inputs = StepInputs.builder()
        .forStepId("process", ExpandEach.of(List.of("apple", "banana", "cherry")))
        .build();
// Creates 3 cloned steps; result.steps() has size 3
// processedItems contains all 3 in any order

// With custom ID suffix:
ExpandEach.of(List.of("item-A", "item-B"), obj -> obj.toString())
// Step IDs: "insert:item-A", "insert:item-B"

// Dependent steps wait for ALL expanded clones:
// .step("aggregate").dependsOn("load") -- waits for load:x AND load:y

// Empty list produces 0 clones; downstream steps with no unmet deps still run
```

## 10. Layer Concurrency Tests

Test that `layerConcurrency` limits parallel execution within a layer:

```java
SagaDefinition saga = SagaBuilder.saga("BoundedSaga", 2)  // layerConcurrency = 2
        .step("s1").handler(boundedStep).add()
        .step("s2").handler(boundedStep).add()
        .step("s3").handler(boundedStep).add()
        .step("s4").handler(boundedStep).add()
        .build();

// Track concurrent count with AtomicInteger, flag if > limit
// With concurrency=2 and 4 steps at 150ms each, elapsed >= 250ms
assertThat(exceededLimit.get()).isFalse();
```

`layerConcurrency = 0` or `-1` means unbounded -- all 4 steps run concurrently.

## 11. TCC Pattern Tests

TCC tests use `TccBuilder`, `TccEngine`, and `TccInputs`. The result type is `TccResult`.

```java
TccDefinition tcc = TccBuilder.tcc("TransferFunds")
        .participant("debit")
            .tryHandler((input, ctx) -> Mono.just("reserved"))
            .confirmHandler((input, ctx) -> Mono.just("confirmed"))
            .cancelHandler((input, ctx) -> Mono.just("cancelled"))
            .add()
        .participant("credit")
            .tryHandler((input, ctx) -> Mono.error(new RuntimeException("insufficient")))
            .confirmHandler((input, ctx) -> Mono.just("confirmed"))
            .cancelHandler((input, ctx) -> Mono.just("cancelled"))
            .add()
        .build();

StepVerifier.create(engine.execute(tcc, TccInputs.empty()))
        .assertNext(result -> {
            assertThat(result.isCanceled()).isTrue();
            assertThat(result.failedParticipantId()).hasValue("credit");
            assertThat(debitCancelled.get()).isTrue();
        })
        .verifyComplete();
```

Key assertions: `result.isConfirmed()`, `result.isCanceled()`, `result.failedParticipantId()`, `result.tryResultOf(id, Type.class)`. Cancel phase runs in reverse participant order.

## 12. Validation Tests

Verify invalid definitions are rejected at build time:

```java
// Duplicate step ID
assertThatThrownBy(() -> SagaBuilder.saga("Dup")
        .step("same").handler(() -> Mono.just("a")).add()
        .step("same").handler(() -> Mono.just("b")).add().build())
    .isInstanceOf(IllegalStateException.class).hasMessageContaining("Duplicate step id");

// Missing handler
assertThatThrownBy(() -> SagaBuilder.saga("NoHandler").step("broken").add())
    .isInstanceOf(IllegalStateException.class).hasMessageContaining("Missing handler");

// @CompensationSagaStep referencing unknown saga or step
assertThatThrownBy(() -> registry.getSaga("MySaga"))
    .isInstanceOf(IllegalStateException.class).hasMessageContaining("unknown saga");
```

## 13. Anti-Patterns

**Calling .block() instead of StepVerifier** -- blocks the reactive thread and risks deadlock. Always use `StepVerifier.create(engine.execute(...)).assertNext(...).verifyComplete()`.

**Using containsExactly for unordered policies** -- `GROUPED_PARALLEL` and `BEST_EFFORT_PARALLEL` have non-deterministic order within a layer. Use `containsExactlyInAnyOrder`.

**Shared mutable state without synchronization** -- parallel steps execute on different threads. Always use `Collections.synchronizedList(new ArrayList<>())`.

**Testing only the happy path** -- a complete saga test suite requires:
1. Happy path (all steps succeed)
2. Failure at each step (N tests for N steps)
3. Compensation policy behavior
4. Compensation error handling
5. Persistence checkpoint correctness (when enabled)

**Not verifying what was compensated** -- always assert on `result.compensatedSteps()` AND verify which steps were NOT compensated (failed steps, steps without compensation).

**Forgetting null-persistence graceful handling** -- verify sagas work with `null` persistence to catch misconfigurations.

**Not testing all CompensationErrorResult codes** -- the framework has 5 result codes (`CONTINUE`, `RETRY`, `FAIL_SAGA`, `SKIP_STEP`, `MARK_COMPENSATED`). Test each one both in isolation and wired into the engine.
