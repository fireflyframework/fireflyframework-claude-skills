---
name: create-handler
description: "Generates a CQRS command or query handler with boilerplate. Usage: /create-handler <command|query> <name>"
---

# Create Handler

Generate a CQRS command or query handler following the Firefly Framework conventions. Refer to the `implement-cqrs` skill for full pattern details.

## Parameters

| Parameter | Required | Description | Example |
|-----------|----------|-------------|---------|
| `type` | Yes | `command` or `query` | `command` |
| `name` | Yes | Handler name in PascalCase (without Command/Query suffix) | `CreateAccount` |

If either parameter is missing, prompt the user before generating.

## Derived Values

- **commandClass**: `{name}Command` (e.g., `CreateAccountCommand`)
- **queryClass**: `{name}Query` (e.g., `GetAccountBalanceQuery`)
- **handlerClass**: `{name}Handler` (e.g., `CreateAccountHandler`)
- **resultClass**: `{name}Result` (e.g., `CreateAccountResult`)
- **Detect the base package** from the existing project structure (scan for `Application.java` or existing `pom.xml` groupId)
- **Detect the module**: place commands/queries in the `interfaces` module and handlers in the `core` module

## File Placement

For a multi-module project:

| File | Location |
|------|----------|
| Command/Query class | `{service}-interfaces/src/main/java/{packagePath}/interfaces/commands/` or `.../queries/` |
| Result class | `{service}-interfaces/src/main/java/{packagePath}/interfaces/dtos/` |
| Handler class | `{service}-core/src/main/java/{packagePath}/core/handlers/` |

For a single-module project, place all files under the appropriate sub-packages within the same module.

## Command Generation

### 1. Command Class

```java
package {basePackage}.interfaces.commands;

import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotNull;
import org.fireflyframework.cqrs.command.Command;
import org.fireflyframework.cqrs.validation.ValidationResult;
import reactor.core.publisher.Mono;

import java.util.UUID;

public class {name}Command implements Command<{name}Result> {

    private final String commandId = UUID.randomUUID().toString();

    @NotBlank(message = "{field} is required")
    private final String exampleField;

    public {name}Command(String exampleField) {
        this.exampleField = exampleField;
    }

    @Override
    public String getCommandId() {
        return commandId;
    }

    @Override
    public Mono<ValidationResult> customValidate() {
        // Add business validation rules here
        return Mono.just(ValidationResult.success());
    }

    public String getExampleField() {
        return exampleField;
    }
}
```

### 2. Handler Class

```java
package {basePackage}.core.handlers;

import {basePackage}.interfaces.commands.{name}Command;
import {basePackage}.interfaces.dtos.{name}Result;
import org.fireflyframework.cqrs.annotations.CommandHandlerComponent;
import org.fireflyframework.cqrs.command.CommandHandler;
import reactor.core.publisher.Mono;

@CommandHandlerComponent(timeout = 30000, retries = 3, metrics = true, tracing = true)
public class {name}Handler extends CommandHandler<{name}Command, {name}Result> {

    // Inject required services via constructor

    @Override
    protected Mono<{name}Result> doHandle({name}Command command) {
        // TODO: Implement business logic
        return Mono.just(new {name}Result());
    }
}
```

### 3. Result Class

```java
package {basePackage}.interfaces.dtos;

import lombok.*;

import java.time.LocalDateTime;
import java.util.UUID;

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class {name}Result {

    private UUID id;
    private String status;
    private LocalDateTime timestamp;
}
```

## Query Generation

### 1. Query Class

```java
package {basePackage}.interfaces.queries;

import jakarta.validation.constraints.NotBlank;
import org.fireflyframework.cqrs.query.Query;
import reactor.core.publisher.Mono;

public class {name}Query implements Query<{name}Result> {

    @NotBlank(message = "ID is required")
    private final String id;

    public {name}Query(String id) {
        this.id = id;
    }

    @Override
    public String getCacheKey() {
        return "{name}Query:" + id;
    }

    public String getId() {
        return id;
    }
}
```

### 2. Handler Class

```java
package {basePackage}.core.handlers;

import {basePackage}.interfaces.queries.{name}Query;
import {basePackage}.interfaces.dtos.{name}Result;
import org.fireflyframework.cqrs.annotations.QueryHandlerComponent;
import org.fireflyframework.cqrs.query.QueryHandler;
import reactor.core.publisher.Mono;

@QueryHandlerComponent(cacheable = true, cacheTtl = 300, metrics = true)
public class {name}Handler extends QueryHandler<{name}Query, {name}Result> {

    // Inject required services via constructor

    @Override
    protected Mono<{name}Result> doHandle({name}Query query) {
        // TODO: Implement read logic
        return Mono.empty();
    }
}
```

### 3. Result Class

```java
package {basePackage}.interfaces.dtos;

import lombok.*;

import java.time.LocalDateTime;
import java.util.UUID;

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class {name}Result {

    private UUID id;
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
}
```

## Authorization

If the user requests authorization, add an `authorize` override to the Command or Query class:

```java
@Override
public Mono<AuthorizationResult> authorize(ExecutionContext context) {
    if (context.getUserId() == null) {
        return Mono.just(AuthorizationResult.failure("userId", "User must be authenticated"));
    }
    return Mono.just(AuthorizationResult.success());
}
```

Import `org.fireflyframework.cqrs.authorization.AuthorizationResult` and `org.fireflyframework.cqrs.context.ExecutionContext`.

## After Generation

1. Replace `exampleField` placeholders in the Command/Query with actual domain fields.
2. Implement the business logic in the Handler's `doHandle` method.
3. Wire the handler through the `CommandBus` or `QueryBus` in a controller or application service.
4. Remind the user: never call `handler.handle()` directly -- always use the bus.
