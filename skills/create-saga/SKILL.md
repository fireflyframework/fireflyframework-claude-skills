---
name: create-saga
description: "Generates a saga with placeholder steps and compensation. Usage: /create-saga <name>"
---

# Create Saga

Generate a saga class with placeholder steps and compensation handlers following the Firefly Framework orchestration patterns. Refer to the `implement-saga` skill for full details on DAG execution, persistence, and compensation policies.

## Parameters

| Parameter | Required | Description | Example |
|-----------|----------|-------------|---------|
| `name` | Yes | Saga name in PascalCase (without Saga suffix) | `CustomerRegistration` |

If the parameter is missing, prompt the user before generating.

## Derived Values

- **sagaClass**: `{name}Saga` (e.g., `CustomerRegistrationSaga`)
- **sagaId**: kebab-case of name (e.g., `customer-registration`)
- **Detect the base package** from the existing project structure
- **Detect the module**: place the saga in the `core` module under `workflows/`

## File Placement

| File | Location |
|------|----------|
| Saga class | `{service}-core/src/main/java/{packagePath}/core/workflows/{name}Saga.java` |
| Step event constants | `{service}-interfaces/src/main/java/{packagePath}/interfaces/events/{name}Events.java` |

For single-module projects, place under the appropriate sub-packages within the same module.

## Saga Class Template

```java
package {basePackage}.core.workflows;

import org.fireflyframework.cqrs.command.CommandBus;
import org.fireflyframework.cqrs.query.QueryBus;
import org.fireflyframework.orchestration.core.argument.FromStep;
import org.fireflyframework.orchestration.core.argument.Input;
import org.fireflyframework.orchestration.saga.annotation.Saga;
import org.fireflyframework.orchestration.saga.annotation.SagaStep;
import org.fireflyframework.orchestration.saga.annotation.OnSagaComplete;
import org.fireflyframework.orchestration.saga.annotation.OnSagaError;
import org.fireflyframework.orchestration.saga.event.StepEvent;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Component;
import reactor.core.publisher.Mono;

@Slf4j
@Component
@Saga(name = "{sagaId}")
public class {name}Saga {

    private final CommandBus commandBus;
    private final QueryBus queryBus;

    public {name}Saga(CommandBus commandBus, QueryBus queryBus) {
        this.commandBus = commandBus;
        this.queryBus = queryBus;
    }

    // ---- Steps ----

    @SagaStep(id = "validate", retry = 3, backoffMs = 1000)
    public Mono<ValidationStepResult> validate(@Input Object request) {
        // TODO: Implement validation logic
        log.info("[{sagaId}] Validating request");
        return Mono.just(new ValidationStepResult(true));
    }

    @SagaStep(
        id = "create-resource",
        dependsOn = "validate",
        compensate = "deleteResource",
        timeoutMs = 30000
    )
    @StepEvent(topic = "{sagaId}-events", type = "resource.created")
    public Mono<ResourceStepResult> createResource(
            @FromStep("validate") ValidationStepResult validation) {
        // TODO: Implement resource creation (e.g., via commandBus.send(...))
        log.info("[{sagaId}] Creating resource");
        return Mono.just(new ResourceStepResult("resource-id"));
    }

    @SagaStep(
        id = "configure-resource",
        dependsOn = "create-resource",
        compensate = "unconfigureResource"
    )
    public Mono<ConfigStepResult> configureResource(
            @FromStep("create-resource") ResourceStepResult resource) {
        // TODO: Implement configuration step
        log.info("[{sagaId}] Configuring resource: {}", resource.getResourceId());
        return Mono.just(new ConfigStepResult(true));
    }

    @SagaStep(id = "notify", dependsOn = "configure-resource")
    public Mono<Void> sendNotification(
            @FromStep("create-resource") ResourceStepResult resource) {
        // Notification has no compensation -- it is non-destructive
        log.info("[{sagaId}] Sending notification for resource: {}", resource.getResourceId());
        return Mono.empty();
    }

    // ---- Compensation Handlers ----

    public Mono<Void> deleteResource(
            @FromStep("create-resource") ResourceStepResult resource) {
        log.info("[{sagaId}] Compensating: deleting resource {}", resource.getResourceId());
        // TODO: Implement compensation (e.g., via commandBus.send(new DeleteCommand(...)))
        return Mono.empty();
    }

    public Mono<Void> unconfigureResource(
            @FromStep("configure-resource") ConfigStepResult config) {
        log.info("[{sagaId}] Compensating: reverting configuration");
        // TODO: Implement compensation
        return Mono.empty();
    }

    // ---- Lifecycle Callbacks ----

    @OnSagaComplete
    public void onComplete() {
        log.info("[{sagaId}] Saga completed successfully");
    }

    @OnSagaError
    public void onError(Throwable error) {
        log.error("[{sagaId}] Saga failed: {}", error.getMessage(), error);
    }

    // ---- Step Result Records ----
    // Replace these with proper classes in the interfaces module as the saga matures

    public record ValidationStepResult(boolean valid) {}
    public record ResourceStepResult(String resourceId) {
        public String getResourceId() { return resourceId; }
    }
    public record ConfigStepResult(boolean configured) {}
}
```

## Step Event Constants Template

```java
package {basePackage}.interfaces.events;

/**
 * Event type constants for the {name} saga.
 * Use these with @StepEvent annotations and EDA consumers.
 */
public final class {name}Events {

    private {name}Events() {}

    public static final String TOPIC = "{sagaId}-events";

    public static final String RESOURCE_CREATED = "resource.created";
    public static final String RESOURCE_CONFIGURED = "resource.configured";
    public static final String SAGA_COMPLETED = "{sagaId}.completed";
    public static final String SAGA_FAILED = "{sagaId}.failed";
}
```

## Execution Example

Include this as a comment or tell the user how to execute the saga:

```java
@Autowired
private SagaEngine sagaEngine;

public Mono<SagaResult> execute(Object request) {
    StepInputs inputs = StepInputs.of("validate", request);
    return sagaEngine.execute("{sagaId}", inputs);
}
```

## DAG Visualization

Show the user the step dependency graph for the generated saga:

```
[validate]              Layer 0
    |
[create-resource]       Layer 1
    |
[configure-resource]    Layer 2
    |
[notify]                Layer 3
```

## After Generation

1. Replace placeholder step result records with proper classes in the interfaces module.
2. Implement actual business logic in each step method using `CommandBus`/`QueryBus`.
3. Add compensation logic for every step that creates or modifies state.
4. Configure persistence for production: set `firefly.orchestration.persistence.provider` to `redis` or `event-sourced`.
5. Remind the user: steps must return `Mono<T>` -- never call `.block()` inside a step.
