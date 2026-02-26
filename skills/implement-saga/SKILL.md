---
name: implement-saga
description: Use when implementing distributed transactions with Saga, Workflow, or TCC patterns in a Firefly Framework service — guides creation of orchestrated multi-step processes with compensation, DAG execution, and pluggable persistence
---

# Implement Saga / Workflow / TCC

## 1. Choosing the Pattern

The orchestration module (`org.fireflyframework.orchestration`) provides three execution patterns via the `ExecutionPattern` enum: `SAGA`, `WORKFLOW`, and `TCC`.

| Criteria | Saga | Workflow | TCC |
|----------|------|----------|-----|
| **Primary use** | Long-running distributed transactions with compensating actions | Multi-step business processes with signals, timers, and child workflows | Short-lived distributed transactions requiring strong isolation |
| **Rollback model** | Backward compensation (undo completed steps in reverse) | Compensation steps + dry-run mode | Two-phase: Cancel all tried participants |
| **Step coordination** | DAG-based parallel layers via `dependsOn` | DAG-based with signals (`@WaitForSignal`), timers (`@WaitForTimer`), child workflows (`@ChildWorkflow`) | Ordered participants with Try -> Confirm/Cancel phases |
| **Annotation** | `@Saga` + `@SagaStep` | `@Workflow` + `@WorkflowStep` | `@Tcc` + `@TccParticipant` |
| **Builder DSL** | `SagaBuilder` | `WorkflowBuilder` | `TccBuilder` |
| **Timeout** | Per-step via `timeoutMs` | Global + per-step | Global + per-participant |
| **Best for** | Order fulfillment, customer onboarding, payment processing | ETL pipelines, approval chains, scheduled batch jobs | Inventory reservation, seat booking, money transfers |

**Rule of thumb:** Use Saga when steps are independent services that need undo logic. Use TCC when you need resource reservation with confirm/cancel semantics. Use Workflow for complex process flows with human interaction, signals, or timers.

## 2. Saga Anatomy

A saga consists of **steps** organized into a DAG (directed acyclic graph). The engine resolves step ordering using Kahn's algorithm (`TopologyBuilder.buildLayers`) to determine which steps can execute in parallel within each layer.

```
            [validate-customer]        Layer 0
                    |
            [create-profile]           Layer 1
                  /    \
    [create-account]  [setup-kyc]      Layer 2  (parallel)
                  \    /
          [send-welcome]               Layer 3
```

**Execution flow:**
1. Engine resolves DAG layers from `dependsOn` declarations
2. Steps within the same layer execute concurrently (bounded by `layerConcurrency`)
3. Step results are stored in `ExecutionContext` and accessible to downstream steps via `@FromStep`
4. On failure: compensation runs for all completed steps according to the active `CompensationPolicy`
5. Failed steps are sent to the dead-letter queue (`DeadLetterService`)
6. Lifecycle events are published via `OrchestrationEventPublisher`

## 3. Annotation Approach

### @Saga (class-level)

```java
import org.fireflyframework.orchestration.saga.annotation.Saga;

@Component
@Saga(
    name = "order-fulfillment",      // Required: unique saga identifier
    layerConcurrency = 0,            // 0 = unbounded parallel within layer
    triggerEventType = ""            // Event type to trigger via EventGateway
)
public class OrderFulfillmentSaga { ... }
```

**Parameters:**
- `name` (String, required) -- Unique saga name used by `SagaEngine.execute(sagaName, inputs)`
- `layerConcurrency` (int, default 0) -- Max concurrent steps per topology layer. 0 means all steps in a layer run in parallel
- `triggerEventType` (String, default "") -- When non-empty, registers this saga with `EventGateway` for event-driven triggering

### @SagaStep (method-level)

```java
import org.fireflyframework.orchestration.saga.annotation.SagaStep;
import org.fireflyframework.orchestration.core.argument.FromStep;
import org.fireflyframework.orchestration.core.argument.Input;

@SagaStep(
    id = "create-profile",
    dependsOn = "validate-customer",
    compensate = "deleteProfile",
    retry = 3,
    backoffMs = 1000,
    timeoutMs = 30000,
    jitter = true,
    jitterFactor = 0.5,
    idempotencyKey = "",
    cpuBound = false,
    compensationRetry = -1,
    compensationTimeoutMs = -1,
    compensationBackoffMs = -1,
    compensationCritical = false
)
public Mono<ProfileResult> createProfile(
        @FromStep("validate-customer") ValidationResult validation) {
    // ...
}
```

**All @SagaStep parameters:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `id` | String | (required) | Unique step identifier within the saga |
| `compensate` | String | "" | Name of the compensation method in the same class |
| `dependsOn` | String[] | {} | Step IDs that must complete before this step executes |
| `retry` | int | 0 | Number of retry attempts on failure (0 = no retries) |
| `backoffMs` | long | -1 | Base backoff delay between retries in ms (-1 = use default 100ms) |
| `timeoutMs` | long | -1 | Step execution timeout in ms (-1 = no timeout) |
| `jitter` | boolean | false | Whether to randomize the backoff delay |
| `jitterFactor` | double | 0.5 | Jitter range (0.0 to 1.0) around the base backoff |
| `idempotencyKey` | String | "" | Key for deduplication; skips execution if key was already processed |
| `cpuBound` | boolean | false | If true, schedules on the parallel scheduler instead of bounded-elastic |
| `compensationRetry` | int | -1 | Retry attempts for the compensation method (-1 = inherit from step) |
| `compensationTimeoutMs` | long | -1 | Compensation timeout in ms (-1 = inherit from step) |
| `compensationBackoffMs` | long | -1 | Compensation backoff in ms (-1 = inherit from step) |
| `compensationCritical` | boolean | false | In `CIRCUIT_BREAKER` policy, a critical failure halts the entire compensation chain |

### Parameter injection annotations

Step methods support automatic argument resolution via annotations from `org.fireflyframework.orchestration.core.argument`:

| Annotation | Target | Description |
|------------|--------|-------------|
| `@Input` | Parameter | Injects the step's input payload (or a map key if `value` is specified) |
| `@FromStep("stepId")` | Parameter | Injects the result of a previously completed step |
| `@Variable("name")` | Parameter | Injects a variable from `ExecutionContext` |
| `@Header("name")` | Parameter | Injects a header value from `ExecutionContext` |
| `@CorrelationId` | Parameter | Injects the current execution's correlation ID |
| `@SetVariable("name")` | Method | Stores the method's return value as a context variable |

### Lifecycle callbacks

```java
@OnSagaComplete(async = false, priority = 0)
public void onSuccess(ExecutionContext ctx, SagaResult result) { ... }

@OnSagaError(
    errorTypes = {PaymentException.class},
    suppressError = false,
    async = false,
    priority = 0
)
public void onPaymentFailure(Throwable error, ExecutionContext ctx) { ... }
```

### External steps and compensation

Steps and compensation handlers can reside in separate beans:

```java
@Component
public class ExternalPaymentStep {

    @ExternalSagaStep(saga = "order-fulfillment", id = "charge-payment",
                      dependsOn = "create-profile", compensate = "refundPayment",
                      retry = 3, backoffMs = 2000)
    public Mono<PaymentResult> charge(@Input OrderRequest order) { ... }

    public Mono<Void> refundPayment(PaymentResult result) { ... }
}

// Or a standalone compensation handler in yet another bean:
@Component
public class InventoryCompensation {

    @CompensationSagaStep(saga = "order-fulfillment", forStepId = "reserve-inventory")
    public Mono<Void> releaseInventory(@FromStep("reserve-inventory") ReservationResult res) { ... }
}
```

### Scheduled sagas

```java
@Saga(name = "nightly-reconciliation")
@ScheduledSaga(cron = "0 0 2 * * *", zone = "UTC", enabled = true,
               input = "{}", description = "Nightly account reconciliation")
public class NightlyReconciliationSaga { ... }
```

`@ScheduledSaga` parameters: `cron`, `zone`, `enabled`, `fixedDelay`, `fixedRate`, `initialDelay`, `input` (JSON), `description`. It is `@Repeatable` via `@ScheduledSagas`.

## 4. Builder DSL

For programmatic or dynamic saga definitions, use `SagaBuilder`:

```java
import org.fireflyframework.orchestration.saga.builder.SagaBuilder;
import org.fireflyframework.orchestration.saga.registry.SagaDefinition;

SagaDefinition def = SagaBuilder.saga("dynamic-order", 3)  // name, layerConcurrency
    .triggerEventType("order.created")
    .step("validate")
        .handler((order, ctx) -> validateOrder(order))
        .retry(3).backoffMs(500).jitter()
        .add()
    .step("reserve")
        .dependsOn("validate")
        .handler((input, ctx) -> reserveInventory(ctx.getResult("validate")))
        .compensation((result, ctx) -> releaseInventory(result))
        .compensationRetry(2)
        .compensationCritical(true)
        .timeoutMs(10000)
        .add()
    .step("charge")
        .dependsOn("validate")
        .handlerCtx(ctx -> chargePayment(ctx))
        .compensationCtx(ctx -> refundPayment(ctx))
        .stepEvent("payments", "payment.charged", "orderId")
        .add()
    .step("ship")
        .dependsOn("reserve", "charge")
        .handler(() -> initiateShipping())
        .add()
    .build();
```

**Step builder methods:**
- `handler(BiFunction<I, ExecutionContext, Mono<O>>)` -- Full handler with input and context
- `handlerCtx(Function<ExecutionContext, Mono<O>>)` -- Context-only handler
- `handlerInput(Function<I, Mono<O>>)` -- Input-only handler
- `handler(Supplier<Mono<O>>)` -- No-argument handler
- `handler(StepHandler<?,?>)` -- Raw `StepHandler` interface
- `compensation(BiFunction<Object, ExecutionContext, Mono<Void>>)` -- Full compensation
- `compensationCtx(Function<ExecutionContext, Mono<Void>>)` -- Context-only compensation
- `compensation(Supplier<Mono<Void>>)` -- No-argument compensation
- `stepEvent(topic, type, key)` -- Publishes an `OrchestrationEvent` on step completion

Execute a builder-defined saga by registering it or passing it directly:

```java
@Autowired SagaEngine sagaEngine;

StepInputs inputs = StepInputs.builder()
    .forStepId("validate", orderRequest)
    .forStepId("reserve", ctx -> ctx.getResult("validate"))  // lazy resolver
    .build();

Mono<SagaResult> result = sagaEngine.execute(def, inputs);
```

### Fan-out with ExpandEach

To dynamically clone a step for each item in a collection:

```java
StepInputs inputs = StepInputs.builder()
    .forStepId("process-item", ExpandEach.of(lineItems, item -> item.getSku()))
    .build();
// Creates process-item:SKU001, process-item:SKU002, etc.
```

## 5. The 5 Compensation Policies

Configured via `CompensationPolicy` enum (`org.fireflyframework.orchestration.core.model`):

| Policy | Behavior | When to use |
|--------|----------|-------------|
| `STRICT_SEQUENTIAL` | Compensates completed steps **one by one in reverse** completion order. Stops on failure (respects error handler). | **Default.** Use when compensation order matters (e.g., refund before releasing inventory). |
| `GROUPED_PARALLEL` | Compensates in **reverse DAG layers** -- steps in the same layer compensate concurrently, then the previous layer. | Use when same-layer compensations are independent (e.g., cancel hotel + cancel flight in parallel). |
| `RETRY_WITH_BACKOFF` | Like `STRICT_SEQUENTIAL` but with automatic retries using each step's `compensationRetry`/`compensationBackoffMs` settings. | Use for compensations against unreliable external services. |
| `CIRCUIT_BREAKER` | Like `RETRY_WITH_BACKOFF` but halts the entire compensation chain when a step marked `compensationCritical = true` fails. | Use when some compensations are prerequisites for others (e.g., must refund before closing account). |
| `BEST_EFFORT_PARALLEL` | Fires **all** compensations concurrently with no ordering guarantees. Errors are swallowed. | Use only for independent, idempotent cleanup (e.g., cache invalidation, notification cancellation). |

**Configuration:**

```yaml
firefly:
  orchestration:
    saga:
      compensation-policy: STRICT_SEQUENTIAL  # or GROUPED_PARALLEL, RETRY_WITH_BACKOFF, etc.
      compensation-error-handler: default       # default, fail-fast, log-and-continue, retry-with-backoff
```

**Custom error handling:** Implement `CompensationErrorHandler` to control behavior per-step:

```java
import org.fireflyframework.orchestration.saga.compensation.CompensationErrorHandler;

public class CustomCompensationHandler implements CompensationErrorHandler {
    @Override
    public CompensationErrorResult handle(String sagaName, String stepId,
                                           Throwable error, int attempt) {
        if (error instanceof TransientException && attempt < 3) return CompensationErrorResult.RETRY;
        if (error instanceof NonCriticalException) return CompensationErrorResult.SKIP_STEP;
        return CompensationErrorResult.CONTINUE;
    }
}
```

`CompensationErrorResult` values: `CONTINUE`, `RETRY`, `FAIL_SAGA`, `SKIP_STEP`, `MARK_COMPENSATED`.

Built-in handlers: `DefaultCompensationErrorHandler`, `FailFastErrorHandler`, `LogAndContinueErrorHandler`, `RetryWithBackoffErrorHandler`, `CompositeCompensationErrorHandler`.

## 6. Step Dependencies (DAG)

Dependencies are declared with the `dependsOn` attribute. The `TopologyBuilder` resolves the DAG into execution layers using Kahn's BFS algorithm.

```java
@SagaStep(id = "A")                          // Layer 0
@SagaStep(id = "B", dependsOn = "A")         // Layer 1
@SagaStep(id = "C", dependsOn = "A")         // Layer 1 (parallel with B)
@SagaStep(id = "D", dependsOn = {"B", "C"})  // Layer 2 (waits for both B and C)
```

**Validation rules** (enforced by `TopologyBuilder.validate`):
- No duplicate step IDs
- No self-dependencies
- No references to non-existent steps
- No circular dependencies (detected via DFS cycle detection)

Violations throw `TopologyValidationException` at registration time.

**Accessing topology at runtime:**

```java
ExecutionContext ctx = ...;
List<List<String>> layers = ctx.getTopologyLayers();
// layers = [["A"], ["B", "C"], ["D"]]
```

## 7. Persistence Configuration

The `ExecutionPersistenceProvider` interface (`org.fireflyframework.orchestration.core.persistence`) stores saga execution state for recovery, auditing, and observability.

**Four implementations:**

| Provider | Class | Configuration | Use case |
|----------|-------|---------------|----------|
| In-Memory | `InMemoryPersistenceProvider` | `provider: in-memory` (default) | Development, testing |
| Redis | `RedisPersistenceProvider` | `provider: redis` + `ReactiveRedisTemplate` bean | Production with fast state access |
| Cache | `CachePersistenceProvider` | `provider: cache` + `CacheAdapter` bean | Flexible backend (Caffeine or Redis) via `fireflyframework-cache` |
| Event-Sourced | `EventSourcedPersistenceProvider` | `provider: event-sourced` + `EventStore` bean | Full audit trail via `fireflyframework-eventsourcing` |

**Configuration:**

```yaml
firefly:
  orchestration:
    persistence:
      provider: redis             # in-memory | redis | cache | event-sourced
      key-prefix: orchestration:
      key-ttl: 24h
      retention-period: 7d
      cleanup-interval: 1h
    recovery:
      enabled: true
      stale-threshold: 1h
```

Auto-configuration (`OrchestrationPersistenceAutoConfiguration`) selects the provider based on available beans and the `provider` property. If no specialized beans are present, `InMemoryPersistenceProvider` is used.

## 8. Event Integration

### Step-level events with @StepEvent

Annotate a step method to publish an `OrchestrationEvent` each time the step completes:

```java
@SagaStep(id = "charge-payment", dependsOn = "validate", compensate = "refund")
@StepEvent(topic = "payments", type = "payment.charged", key = "orderId")
public Mono<PaymentResult> chargePayment(@Input OrderRequest order) { ... }
```

`@StepEvent` parameters: `topic`, `type`, `key` (all String, default "").

### Saga-level events

The `SagaEngine` automatically publishes `OrchestrationEvent` records through `OrchestrationEventPublisher`:
- `EXECUTION_STARTED` -- when the saga begins
- `EXECUTION_COMPLETED` -- when the saga finishes (with `ExecutionStatus.COMPLETED` or `FAILED`)
- `STEP_COMPLETED` / `STEP_FAILED` -- per-step outcomes

### Event-triggered sagas

Set `triggerEventType` on `@Saga` (or via `SagaBuilder.triggerEventType`) to register the saga with the `EventGateway`:

```java
@Saga(name = "order-fulfillment", triggerEventType = "order.placed")
```

Route events programmatically:

```java
@Autowired EventGateway eventGateway;

eventGateway.routeEvent("order.placed", Map.of("orderId", "ORD-123"));
```

## 9. Real-World Example: Customer Registration Saga

Based on the customer registration saga from the Firefly Framework codebase:

```java
@Component
@Saga(name = "customer-registration")
public class CustomerRegistrationSaga {

    private final CommandBus commandBus;
    private final QueryBus queryBus;

    public CustomerRegistrationSaga(CommandBus commandBus, QueryBus queryBus) {
        this.commandBus = commandBus;
        this.queryBus = queryBus;
    }

    @SagaStep(id = "validate-customer", retry = 3, backoffMs = 1000)
    public Mono<CustomerValidationResult> validateCustomer(
            @Input CustomerRegistrationRequest request) {
        return queryBus.send(ValidateCustomerQuery.builder()
            .email(request.getEmail())
            .phoneNumber(request.getPhoneNumber())
            .build());
    }

    @SagaStep(id = "create-profile", dependsOn = "validate-customer",
              compensate = "deleteProfile", timeoutMs = 30000)
    public Mono<CustomerProfileResult> createProfile(
            @FromStep("validate-customer") CustomerValidationResult validation) {
        if (!validation.isValid()) {
            return Mono.error(new CustomerValidationException(validation.getValidationErrors()));
        }
        return commandBus.send(CreateCustomerProfileCommand.builder()
            .customerId(validation.getCustomerId())
            .build());
    }

    @SagaStep(id = "create-account", dependsOn = "create-profile",
              compensate = "closeAccount")
    public Mono<AccountCreationResult> createInitialAccount(
            @FromStep("create-profile") CustomerProfileResult profile) {
        return commandBus.send(CreateAccountCommand.builder()
            .customerId(profile.getCustomerId())
            .accountType("CHECKING")
            .build());
    }

    @SagaStep(id = "send-welcome", dependsOn = "create-account")
    public Mono<NotificationResult> sendWelcomeNotification(
            @FromStep("create-profile") CustomerProfileResult profile,
            @FromStep("create-account") AccountCreationResult account) {
        return notificationService.sendWelcome(profile, account);
    }

    // --- Compensation methods ---

    public Mono<Void> deleteProfile(
            @FromStep("create-profile") CustomerProfileResult profile) {
        return commandBus.send(new DeleteCustomerProfileCommand(profile.getProfileId())).then();
    }

    public Mono<Void> closeAccount(
            @FromStep("create-account") AccountCreationResult account) {
        return commandBus.send(new CloseAccountCommand(account.getAccountNumber())).then();
    }
}
```

**Key observations from this real saga:**
- Steps form a linear chain: validate -> create-profile -> create-account -> send-welcome
- Only steps that create state have compensation methods (`create-profile`, `create-account`)
- The final notification step (`send-welcome`) has no compensation since it is non-destructive
- `@FromStep` is used to pass results between steps without manual context manipulation
- CQRS `CommandBus`/`QueryBus` integration is natural inside step methods

**Execution:**

```java
@Autowired SagaEngine sagaEngine;

StepInputs inputs = StepInputs.of("validate-customer", registrationRequest);
SagaResult result = sagaEngine.execute("customer-registration", inputs).block();

if (result.isSuccess()) {
    String profileId = result.resultOf("create-profile", CustomerProfileResult.class)
        .map(CustomerProfileResult::getProfileId).orElse(null);
}
```

**Inspecting results:**

```java
SagaResult result = ...;
result.sagaName();           // "customer-registration"
result.correlationId();      // Auto-generated UUID
result.isSuccess();          // true/false
result.duration();           // Duration between start and end
result.steps();              // Map<String, StepOutcome> with status, attempts, latency, result, error
result.failedSteps();        // List of step IDs that failed
result.compensatedSteps();   // List of step IDs that were compensated
result.firstErrorStepId();   // Optional<String>
result.report();             // Optional<ExecutionReport> with detailed execution report
```

## 10. Testing Sagas

Use `InMemoryPersistenceProvider` and the builder DSL for isolated tests:

```java
@SpringBootTest
class OrderSagaTest {

    @Autowired SagaEngine sagaEngine;

    @Test
    void happyPath() {
        StepInputs inputs = StepInputs.of("validate-customer", validRequest());
        SagaResult result = sagaEngine.execute("customer-registration", inputs).block();

        assertThat(result.isSuccess()).isTrue();
        assertThat(result.steps().get("create-profile").status()).isEqualTo(StepStatus.DONE);
        assertThat(result.compensatedSteps()).isEmpty();
    }

    @Test
    void compensationOnFailure() {
        StepInputs inputs = StepInputs.of("validate-customer", invalidRequest());
        SagaResult result = sagaEngine.execute("customer-registration", inputs).block();

        assertThat(result.isFailed()).isTrue();
        assertThat(result.firstErrorStepId()).contains("create-profile");
        assertThat(result.compensatedSteps()).isEmpty(); // validate has no compensation
    }

    @Test
    void middleStepFailureTriggersPriorCompensation() {
        // Force create-account to fail
        StepInputs inputs = StepInputs.of("validate-customer", requestThatFailsAtAccount());
        SagaResult result = sagaEngine.execute("customer-registration", inputs).block();

        assertThat(result.isFailed()).isTrue();
        assertThat(result.compensatedSteps()).contains("create-profile");
        assertThat(result.steps().get("create-profile").compensated()).isTrue();
    }
}
```

**Testing builder-defined sagas in isolation:**

```java
@Test
void builderSagaTest() {
    AtomicBoolean compensated = new AtomicBoolean(false);

    SagaDefinition saga = SagaBuilder.saga("test-saga")
        .step("succeed").handler(() -> Mono.just("ok")).add()
        .step("fail").dependsOn("succeed")
            .handler(() -> Mono.error(new RuntimeException("boom")))
            .compensation(() -> { compensated.set(true); return Mono.empty(); })
            .add()
        .build();

    SagaResult result = sagaEngine.execute(saga, StepInputs.empty()).block();
    assertThat(result.isFailed()).isTrue();
    assertThat(compensated.get()).isTrue();
}
```

For comprehensive testing patterns including StepVerifier-based reactive assertions, see the `testing-reactive-services` skill.

## 11. Common Mistakes

**Non-idempotent steps.** Saga steps may be retried. If a step calls an external API that charges money, a retry creates a double charge. Always use `idempotencyKey` or design steps to be naturally idempotent (check-then-act pattern).

**Missing compensation.** Every step that creates or modifies state MUST have a `compensate` method. If a step after yours fails, your step's side effects remain without compensation. The `send-welcome` pattern (no compensation) is only safe for non-destructive actions.

**Blocking calls in step methods.** Step methods must return `Mono<T>`. Never call `.block()` inside a step -- this deadlocks the bounded-elastic scheduler. Wrap blocking calls with `Mono.fromCallable(...).subscribeOn(Schedulers.boundedElastic())`.

**Wrong compensation policy.** Using `BEST_EFFORT_PARALLEL` when compensation order matters leads to data inconsistency. Use `STRICT_SEQUENTIAL` (the default) unless you have a specific reason to change it.

**Circular dependencies.** `TopologyBuilder` validates the DAG at startup and throws `TopologyValidationException` for cycles, but dangling `dependsOn` references to misspelled step IDs also fail validation. Always verify step IDs match exactly.

**Forgetting `dependsOn` for data flow.** Using `@FromStep("X")` without `dependsOn = "X"` means the step may execute before X completes, resulting in a null injection. The argument resolver does not enforce ordering -- `dependsOn` does.

**Not configuring persistence for production.** The default `InMemoryPersistenceProvider` loses all state on restart. For production, configure `redis`, `cache`, or `event-sourced` persistence so that the `RecoveryService` can resume stale executions.

**Compensation that throws and is not retried.** If a compensation method fails and the policy is `STRICT_SEQUENTIAL` with the default error handler (`CONTINUE`), the failure is silently swallowed. For critical compensations, use `compensationCritical = true` with `CIRCUIT_BREAKER` or implement a custom `CompensationErrorHandler`.

**Wrong StepStatus enum value.** The `StepStatus` enum values are: `PENDING`, `RUNNING`, `DONE`, `FAILED`, `SKIPPED`, `TIMED_OUT`, `RETRYING`. A common mistake is using `StepStatus.COMPLETED` -- this value does not exist. Always use `StepStatus.DONE` to indicate a successfully completed step.

```java
// WRONG -- StepStatus.COMPLETED does not exist
ctx.setStepStatus(stepId, StepStatus.COMPLETED);

// CORRECT
ctx.setStepStatus(stepId, StepStatus.DONE);
```

**Returning mock/static data from service methods.** When building a domain service that calls core services via SDK, never return static data or hardcoded values because the core service endpoint "doesn't exist yet". Either create the endpoint in the core service, or define a port interface with a stub adapter that throws `NotImplementedException`. Empty or mock returns in production code hide integration gaps and create silent failures.
