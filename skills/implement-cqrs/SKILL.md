---
name: implement-cqrs
description: Use when implementing command or query handlers in a Firefly Framework service — guides creation of Commands, Queries, Handlers with proper bus wiring, authorization, validation, caching, and reactive patterns
---

# Implement CQRS

## When to Use

Use the **Command** side when the operation changes state (creates, updates, deletes). Use the **Query** side when the operation reads data without side effects.

- **Commands** represent intentions to change state. They are processed through the `CommandBus`, validated, authorized, and dispatched to exactly one `CommandHandler`. Commands return a result type `R` wrapped in `Mono<R>`.
- **Queries** represent requests for data. They are processed through the `QueryBus`, validated, authorized, optionally cached, and dispatched to exactly one `QueryHandler`. Queries return a result type `R` wrapped in `Mono<R>`.

Never mix reads and writes in the same handler. A command handler must not return data that was merely read; a query handler must not mutate state.

## Command Flow

### Step 1 -- Define the Command

Implement `org.fireflyframework.cqrs.command.Command<R>` where `R` is the result type. Add Jakarta Bean Validation annotations on fields. The `Command` interface provides default methods for `getCommandId()`, `getTimestamp()`, `getCorrelationId()`, `getInitiatedBy()`, `getMetadata()`, `validate()`, `customValidate()`, `authorize()`, and `authorize(ExecutionContext)`.

```java
package com.example.account.command;

import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotNull;
import jakarta.validation.constraints.Positive;
import org.fireflyframework.cqrs.command.Command;
import org.fireflyframework.cqrs.validation.ValidationResult;
import reactor.core.publisher.Mono;

import java.math.BigDecimal;

public class CreateAccountCommand implements Command<AccountCreatedResult> {

    @NotBlank(message = "Customer ID is required")
    private final String customerId;

    @NotBlank(message = "Account type is required")
    private final String accountType;

    @NotNull
    @Positive(message = "Initial balance must be positive")
    private final BigDecimal initialBalance;

    public CreateAccountCommand(String customerId, String accountType, BigDecimal initialBalance) {
        this.customerId = customerId;
        this.accountType = accountType;
        this.initialBalance = initialBalance;
    }

    // Override customValidate() for business rules beyond Jakarta annotations
    @Override
    public Mono<ValidationResult> customValidate() {
        if (initialBalance.compareTo(new BigDecimal("1000000")) > 0) {
            return Mono.just(ValidationResult.failure("initialBalance", "Exceeds maximum deposit limit"));
        }
        return Mono.just(ValidationResult.success());
    }

    public String getCustomerId() { return customerId; }
    public String getAccountType() { return accountType; }
    public BigDecimal getInitialBalance() { return initialBalance; }
}
```

**Important defaults**: `getCommandId()` and `getTimestamp()` generate new values on every call. If you need a stable ID (for idempotency, logging, retry), store the value in a field:

```java
private final String commandId = UUID.randomUUID().toString();
private final Instant timestamp = Instant.now();

@Override public String getCommandId() { return commandId; }
@Override public Instant getTimestamp() { return timestamp; }
```

### Step 2 -- Create the Handler

Extend `org.fireflyframework.cqrs.command.CommandHandler<C, R>` and annotate with `@CommandHandlerComponent`. Implement only `doHandle(C command)`. The base class provides automatic type detection from generics, logging, metrics, pre/post processing hooks, and error handling.

```java
package com.example.account.command;

import org.fireflyframework.cqrs.annotations.CommandHandlerComponent;
import org.fireflyframework.cqrs.command.CommandHandler;
import reactor.core.publisher.Mono;

@CommandHandlerComponent(timeout = 30000, retries = 3, metrics = true, tracing = true)
public class CreateAccountHandler extends CommandHandler<CreateAccountCommand, AccountCreatedResult> {

    private final AccountService accountService;

    public CreateAccountHandler(AccountService accountService) {
        this.accountService = accountService;
    }

    @Override
    protected Mono<AccountCreatedResult> doHandle(CreateAccountCommand command) {
        return accountService.createAccount(
            command.getCustomerId(),
            command.getAccountType(),
            command.getInitialBalance()
        );
    }
}
```

### Step 3 -- Send via the CommandBus

Inject `CommandBus` and call `send()`. The bus runs the full pipeline: handler lookup, validation (Jakarta + custom), authorization, execution, metrics.

```java
@Service
public class AccountApplicationService {

    private final CommandBus commandBus;

    public Mono<AccountCreatedResult> createAccount(CreateAccountRequest request) {
        CreateAccountCommand command = new CreateAccountCommand(
            request.getCustomerId(),
            request.getAccountType(),
            request.getInitialBalance()
        );
        return commandBus.send(command);
    }
}
```

## Query Flow

### Step 1 -- Define the Query

Implement `org.fireflyframework.cqrs.query.Query<R>`. The `Query` interface provides default methods for `getQueryId()`, `getTimestamp()`, `getCorrelationId()`, `getInitiatedBy()`, `getMetadata()`, `isCacheable()`, `getCacheKey()`, `authorize()`, and `authorize(ExecutionContext)`.

```java
package com.example.account.query;

import jakarta.validation.constraints.NotBlank;
import org.fireflyframework.cqrs.query.Query;

public class GetAccountBalanceQuery implements Query<AccountBalance> {

    @NotBlank
    private final String accountNumber;

    @NotBlank
    private final String customerId;

    public GetAccountBalanceQuery(String accountNumber, String customerId) {
        this.accountNumber = accountNumber;
        this.customerId = customerId;
    }

    public String getAccountNumber() { return accountNumber; }
    public String getCustomerId() { return customerId; }
}
```

By default, `isCacheable()` returns `true` and `getCacheKey()` returns the query class simple name plus a metadata hash. Override these to customize caching behavior per query instance.

### Step 2 -- Create the Handler

Extend `org.fireflyframework.cqrs.query.QueryHandler<Q, R>` and annotate with `@QueryHandlerComponent`. Implement only `doHandle(Q query)`.

```java
package com.example.account.query;

import org.fireflyframework.cqrs.annotations.QueryHandlerComponent;
import org.fireflyframework.cqrs.query.QueryHandler;
import reactor.core.publisher.Mono;

@QueryHandlerComponent(cacheable = true, cacheTtl = 300, metrics = true)
public class GetAccountBalanceHandler extends QueryHandler<GetAccountBalanceQuery, AccountBalance> {

    private final AccountReadService accountReadService;

    public GetAccountBalanceHandler(AccountReadService accountReadService) {
        this.accountReadService = accountReadService;
    }

    @Override
    protected Mono<AccountBalance> doHandle(GetAccountBalanceQuery query) {
        return accountReadService.getBalance(query.getAccountNumber());
    }
}
```

### Step 3 -- Execute via the QueryBus

```java
@Service
public class AccountQueryService {

    private final QueryBus queryBus;

    public Mono<AccountBalance> getBalance(String accountNumber, String customerId) {
        return queryBus.query(new GetAccountBalanceQuery(accountNumber, customerId));
    }
}
```

The `QueryBus` also exposes `clearCache(String cacheKey)` and `clearAllCache()` for manual cache invalidation.

## Complete Command Example

This example shows a money transfer command with authorization, validation, context propagation, and domain event publishing.

```java
// --- Command ---
package com.example.transfer.command;

import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotNull;
import jakarta.validation.constraints.Positive;
import org.fireflyframework.cqrs.authorization.AuthorizationResult;
import org.fireflyframework.cqrs.command.Command;
import org.fireflyframework.cqrs.context.ExecutionContext;
import org.fireflyframework.cqrs.validation.ValidationResult;
import reactor.core.publisher.Mono;

import java.math.BigDecimal;

public class TransferMoneyCommand implements Command<TransferResult> {

    @NotBlank private final String sourceAccountId;
    @NotBlank private final String targetAccountId;
    @NotNull @Positive private final BigDecimal amount;
    private final String description;

    // stable ID for idempotency
    private final String commandId = java.util.UUID.randomUUID().toString();

    public TransferMoneyCommand(String sourceAccountId, String targetAccountId,
                                BigDecimal amount, String description) {
        this.sourceAccountId = sourceAccountId;
        this.targetAccountId = targetAccountId;
        this.amount = amount;
        this.description = description;
    }

    @Override
    public String getCommandId() { return commandId; }

    @Override
    public Mono<ValidationResult> customValidate() {
        if (sourceAccountId.equals(targetAccountId)) {
            return Mono.just(ValidationResult.failure("targetAccountId",
                "Cannot transfer to the same account"));
        }
        return Mono.just(ValidationResult.success());
    }

    @Override
    public Mono<AuthorizationResult> authorize(ExecutionContext context) {
        // Context-aware authorization -- verify the user owns the source account
        // Implement actual ownership check via a service
        if (context.getUserId() == null) {
            return Mono.just(AuthorizationResult.failure("userId", "User must be authenticated"));
        }
        return Mono.just(AuthorizationResult.success());
    }

    public String getSourceAccountId() { return sourceAccountId; }
    public String getTargetAccountId() { return targetAccountId; }
    public BigDecimal getAmount() { return amount; }
    public String getDescription() { return description; }
}

// --- Handler ---
package com.example.transfer.command;

import org.fireflyframework.cqrs.annotations.CommandHandlerComponent;
import org.fireflyframework.cqrs.command.CommandHandler;
import org.fireflyframework.cqrs.event.annotation.PublishDomainEvent;
import reactor.core.publisher.Mono;

@CommandHandlerComponent(timeout = 30000, retries = 3, metrics = true)
@PublishDomainEvent(destination = "transfer-events", eventType = "MoneyTransferred")
public class TransferMoneyHandler extends CommandHandler<TransferMoneyCommand, TransferResult> {

    private final TransferService transferService;

    public TransferMoneyHandler(TransferService transferService) {
        this.transferService = transferService;
    }

    @Override
    protected Mono<TransferResult> doHandle(TransferMoneyCommand command) {
        return transferService.execute(
            command.getSourceAccountId(),
            command.getTargetAccountId(),
            command.getAmount(),
            command.getDescription()
        );
    }
}

// --- Sending with ExecutionContext ---
ExecutionContext context = ExecutionContext.builder()
    .withUserId("user-123")
    .withTenantId("tenant-456")
    .withSource("mobile-app")
    .withFeatureFlag("premium-transfers", true)
    .build();

commandBus.send(transferMoneyCommand, context)
    .subscribe(result -> log.info("Transfer completed: {}", result));
```

## Complete Query Example

This example shows a cached, authorized query with event-driven cache invalidation.

```java
// --- Query ---
package com.example.transaction.query;

import jakarta.validation.constraints.NotBlank;
import org.fireflyframework.cqrs.authorization.AuthorizationResult;
import org.fireflyframework.cqrs.context.ExecutionContext;
import org.fireflyframework.cqrs.query.Query;
import reactor.core.publisher.Mono;

import java.util.Map;

public class GetTransactionHistoryQuery implements Query<TransactionHistory> {

    @NotBlank private final String accountNumber;
    private final int pageSize;
    private final int pageNumber;

    public GetTransactionHistoryQuery(String accountNumber, int pageSize, int pageNumber) {
        this.accountNumber = accountNumber;
        this.pageSize = pageSize;
        this.pageNumber = pageNumber;
    }

    @Override
    public String getCacheKey() {
        return "TransactionHistory:" + accountNumber + ":" + pageSize + ":" + pageNumber;
    }

    @Override
    public Mono<AuthorizationResult> authorize(ExecutionContext context) {
        if (context.getUserId() == null) {
            return Mono.just(AuthorizationResult.failure("userId", "Authentication required"));
        }
        return Mono.just(AuthorizationResult.success());
    }

    public String getAccountNumber() { return accountNumber; }
    public int getPageSize() { return pageSize; }
    public int getPageNumber() { return pageNumber; }
}

// --- Handler ---
package com.example.transaction.query;

import org.fireflyframework.cqrs.annotations.QueryHandlerComponent;
import org.fireflyframework.cqrs.cache.annotation.InvalidateCacheOn;
import org.fireflyframework.cqrs.query.QueryHandler;
import reactor.core.publisher.Mono;

@QueryHandlerComponent(cacheable = true, cacheTtl = 300, metrics = true)
@InvalidateCacheOn(eventTypes = {"MoneyTransferred", "DepositCompleted", "WithdrawalCompleted"})
public class GetTransactionHistoryHandler
        extends QueryHandler<GetTransactionHistoryQuery, TransactionHistory> {

    private final TransactionReadService transactionReadService;

    public GetTransactionHistoryHandler(TransactionReadService transactionReadService) {
        this.transactionReadService = transactionReadService;
    }

    @Override
    protected Mono<TransactionHistory> doHandle(GetTransactionHistoryQuery query) {
        return transactionReadService.getHistory(
            query.getAccountNumber(), query.getPageSize(), query.getPageNumber()
        );
    }
}
```

## Authorization

Authorization in the CQRS module is handled through the `authorize()` method on `Command` and `Query` interfaces. There is no separate `@Secure` or `@Authorized` annotation -- authorization logic lives on the message itself.

**How it works**:
1. The `DefaultCommandBus`/`DefaultQueryBus` calls `AuthorizationService.authorizeCommand(command)` or `authorizeQuery(query)` before handler execution.
2. `AuthorizationService` delegates to the command/query's `authorize()` method (or `authorize(ExecutionContext)` if context is provided).
3. If `AuthorizationResult.isUnauthorized()`, an `AuthorizationException` is thrown (extends `FireflySecurityException`).
4. Authorization can be globally disabled via `firefly.cqrs.authorization.enabled=false`.

**Patterns**:

```java
// Simple authorization on a command
@Override
public Mono<AuthorizationResult> authorize() {
    return accountService.verifyOwnership(accountId, getCurrentUserId())
        .map(isOwner -> isOwner
            ? AuthorizationResult.success()
            : AuthorizationResult.failure("account", "Account does not belong to user"));
}

// Context-aware authorization
@Override
public Mono<AuthorizationResult> authorize(ExecutionContext context) {
    String tenantId = context.getTenantId();
    String userId = context.getUserId();
    return accountService.verifyAccountBelongsToTenant(accountId, tenantId)
        .map(belongs -> belongs
            ? AuthorizationResult.success()
            : AuthorizationResult.failure("account", "Account does not belong to tenant"));
}

// Multiple authorization checks combined
@Override
public Mono<AuthorizationResult> authorize(ExecutionContext context) {
    return Mono.zip(
        accountService.verifyOwnership(sourceAccountId, context.getUserId()),
        permissionService.hasPermission(context.getUserId(), "TRANSFER_MONEY")
    ).map(tuple -> {
        AuthorizationResult.Builder builder = AuthorizationResult.builder();
        if (!tuple.getT1()) {
            builder.addError("sourceAccount", "Not owned by user", "OWNERSHIP_VIOLATION");
        }
        if (!tuple.getT2()) {
            builder.addError("permission", "Missing transfer permission", "PERMISSION_DENIED");
        }
        return builder.build();
    });
}
```

The `@CustomAuthorization` annotation (`org.fireflyframework.cqrs.authorization.annotation.CustomAuthorization`) can be placed on command/query classes for documentation and debugging metadata (description, priority) but does not change runtime behavior.

**Configuration** (`application.yml`):

```yaml
firefly:
  cqrs:
    authorization:
      enabled: true
      custom:
        enabled: true
        timeout-ms: 5000
      logging:
        enabled: true
        log-successful: false
```

## Validation

Validation runs in two phases, both automatic:

1. **Jakarta Bean Validation** -- The `AutoValidationProcessor` uses `jakarta.validation.Validator` to process annotations (`@NotNull`, `@NotBlank`, `@Email`, `@Min`, `@Max`, `@Size`, `@Pattern`, `@Positive`, `@Negative`, `@Future`, `@Past`, etc.) on command/query fields. This happens automatically in the bus before handler execution.
2. **Custom validation** -- The bus calls `command.validate()` which delegates to `customValidate()`. Override `customValidate()` for business rules that cannot be expressed as annotations.

If validation fails, a `ValidationException` is thrown (extends `FireflyException`) containing a `ValidationResult` with a list of `ValidationError` objects (each with `fieldName`, `message`, `errorCode`, `severity`).

```java
// Jakarta annotations handle structural validation
public class CreateOrderCommand implements Command<OrderResult> {
    @NotNull(message = "Customer ID is required")
    private final String customerId;

    @NotNull @Positive(message = "Amount must be positive")
    private final BigDecimal amount;

    // Business validation in customValidate()
    @Override
    public Mono<ValidationResult> customValidate() {
        ValidationResult.Builder builder = ValidationResult.builder();
        if (amount.compareTo(new BigDecimal("1000000")) > 0) {
            builder.addError("amount", "Exceeds maximum order limit", "MAX_AMOUNT_EXCEEDED");
        }
        return Mono.just(builder.build());
    }
}
```

The `ValidationResult` supports combining results: `result1.combine(result2)`.

## Caching

Query caching is configured through the `@QueryHandlerComponent` annotation and the `Query` interface.

**Annotation-level configuration** on the handler:
- `cacheable` (default `true`) -- whether results are cached
- `cacheTtl` (default `-1`, uses global default) -- TTL in seconds
- `cacheKeyFields` -- specific query fields to include in the cache key
- `cacheKeyPrefix` -- custom prefix (defaults to query class name)
- `autoEvictCache` -- enable automatic eviction when related commands execute
- `evictOnCommands` -- command class names that trigger eviction

**Query-level configuration**:
- `isCacheable()` returns `true` by default; override to return `false` for specific query instances
- `getCacheKey()` returns `ClassName` or `ClassName:metadataHash`; override for custom keys

**Cache infrastructure**: The `QueryCacheAdapter` wraps `FireflyCacheManager` from fireflyframework-cache. All CQRS cache keys are prefixed with `:cqrs:`, resulting in keys like `firefly:cache:cqrs-queries::cqrs:GetAccountBalanceQuery`.

**Event-driven cache invalidation**: Annotate query handlers with `@InvalidateCacheOn` to clear the cache when specific domain events arrive from the EDA module:

```java
@QueryHandlerComponent(cacheable = true, cacheTtl = 300)
@InvalidateCacheOn(eventTypes = {"UserCreated", "UserUpdated", "UserDeleted"})
public class GetUserHandler extends QueryHandler<GetUserQuery, UserDTO> {
    // Cache is automatically cleared when any of these events are received
}
```

**Global cache configuration** (`application.yml`):

```yaml
firefly:
  cqrs:
    query:
      caching-enabled: true
      cache-ttl: 15m
```

## Context and Tracing

### ExecutionContext

`ExecutionContext` carries request-scoped metadata through the command/query pipeline. Build it with the fluent builder and pass it to `commandBus.send(command, context)` or `queryBus.query(query, context)`.

```java
ExecutionContext context = ExecutionContext.builder()
    .withUserId("user-123")
    .withTenantId("tenant-456")
    .withOrganizationId("org-789")
    .withSessionId("session-abc")
    .withRequestId("req-def")
    .withSource("mobile-app")
    .withClientIp("192.168.1.1")
    .withUserAgent("MyApp/2.0")
    .withFeatureFlag("premium-features", true)
    .withProperty("customKey", customValue)
    .build();
```

Handlers that always require context should extend `ContextAwareCommandHandler<C, R>` or `ContextAwareQueryHandler<Q, R>` instead of the standard base classes. These enforce that `doHandle(command, context)` is the only entry point -- calling `doHandle(command)` throws `UnsupportedOperationException`.

### CorrelationContext

`CorrelationContext` manages correlation IDs and trace IDs for distributed tracing across service boundaries. It is auto-configured as a Spring bean.

- Stores IDs in `ThreadLocal` and SLF4J `MDC` (`correlationId`, `traceId`)
- `getOrCreateCorrelationId()` generates an ID if none exists
- `createContextHeaders()` builds a header map (`X-Correlation-ID`, `X-Trace-ID`, `timestamp`, `source`) for event publishing
- `extractContextFromHeaders(Map)` restores context from received event headers
- `withContext(Runnable)` wraps a `Runnable` to propagate correlation context to another thread
- `storeContext(key)` / `restoreContext(key)` for async context propagation

The `DefaultCommandBus` automatically sets the correlation ID from the command before handler execution and clears it in `doFinally`.

## Domain Event Publishing

Command handlers can automatically publish domain events after successful execution by adding the `@PublishDomainEvent` annotation:

```java
@CommandHandlerComponent
@PublishDomainEvent(destination = "user-events", eventType = "UserCreated")
public class CreateUserHandler extends CommandHandler<CreateUserCommand, UserCreatedResult> {
    @Override
    protected Mono<UserCreatedResult> doHandle(CreateUserCommand command) {
        return userService.createUser(command);
        // The result is automatically published to "user-events" topic
    }
}
```

- `destination` -- the topic, queue, or exchange to publish to
- `eventType` -- defaults to the result class simple name if empty
- Requires `fireflyframework-eda` on the classpath; the `CommandEventPublisher` interface is backed by `EdaCommandEventPublisher`
- Publishing failures are logged but do not fail the command

## Fluent API

The fluent builders `CommandBuilder` and `QueryBuilder` provide an alternative way to create and execute commands/queries with reduced boilerplate. They use reflection (Lombok builder, constructor, factory method, or setter strategies) to construct instances.

```java
// Fluent command execution
CommandBuilder.create(CreateAccountCommand.class)
    .with("customerId", "CUST-123")
    .with("accountType", "SAVINGS")
    .with("initialBalance", new BigDecimal("1000.00"))
    .correlatedBy("REQ-456")
    .initiatedBy("user@example.com")
    .withMetadata("channel", "MOBILE")
    .executeWith(commandBus)
    .subscribe(result -> log.info("Account created: {}", result));

// Fluent query execution
QueryBuilder.create(GetAccountBalanceQuery.class)
    .with("accountNumber", "ACC-123")
    .with("customerId", "CUST-123")
    .correlatedBy("REQ-456")
    .cached(true)
    .withCacheTtl(600)
    .executeWith(queryBus)
    .subscribe(balance -> log.info("Balance: {}", balance));
```

## Package Structure

```
com.example.myservice/
  command/
    CreateAccountCommand.java          # implements Command<AccountCreatedResult>
    CreateAccountHandler.java          # extends CommandHandler, @CommandHandlerComponent
    AccountCreatedResult.java          # result DTO
    TransferMoneyCommand.java
    TransferMoneyHandler.java
    TransferResult.java
  query/
    GetAccountBalanceQuery.java        # implements Query<AccountBalance>
    GetAccountBalanceHandler.java      # extends QueryHandler, @QueryHandlerComponent
    AccountBalance.java                # result DTO
    GetTransactionHistoryQuery.java
    GetTransactionHistoryHandler.java
    TransactionHistory.java
  service/
    AccountApplicationService.java     # injects CommandBus, orchestrates commands
    AccountQueryService.java           # injects QueryBus, orchestrates queries
```

Group commands with their handlers and result types in a `command` package. Group queries with their handlers and result types in a `query` package. Application services that compose bus calls go in `service`.

## Common Mistakes

1. **Blocking in handlers** -- `doHandle()` must return `Mono<R>`. Never call `.block()`, `Thread.sleep()`, or synchronous I/O inside a handler. Use reactive operators (`flatMap`, `map`, `zipWith`) to compose async calls.

2. **Missing authorization** -- By default, `authorize()` returns `AuthorizationResult.success()`. If a command or query accesses user-scoped resources, you must override `authorize()` or `authorize(ExecutionContext)` with actual ownership/permission checks. Forgetting this means any user can access any resource.

3. **Bypassing the bus** -- Never call `handler.handle(command)` directly. Always go through `commandBus.send()` or `queryBus.query()`. The bus provides the validation, authorization, correlation context, metrics, and event publishing pipeline.

4. **Mixing command and query** -- A command handler must not perform read-only queries to return data to the caller beyond its result. A query handler must not write state. If you need both, send a command and then issue a separate query.

5. **Unstable command/query IDs** -- The default `getCommandId()` and `getQueryId()` return a new UUID on every call. If you log, retry, or need idempotency, store the ID in a `final` field.

6. **Forgetting `@CommandHandlerComponent` or `@QueryHandlerComponent`** -- Without the annotation, the handler is not registered as a Spring bean and will not be discovered by the bus. You will get `CommandHandlerNotFoundException` at runtime.

7. **Using `ContextAwareCommandHandler` without `ExecutionContext`** -- If your handler extends `ContextAwareCommandHandler`, callers must use `commandBus.send(command, context)`. Using `commandBus.send(command)` will throw `UnsupportedOperationException`.

8. **Not wiring cache invalidation** -- If you cache query results but commands mutate the same data, stale reads occur. Use `@InvalidateCacheOn(eventTypes = {...})` on the query handler, or use `@QueryHandlerComponent(autoEvictCache = true, evictOnCommands = {...})`, or manually call `queryBus.clearCache(key)`.

9. **Ignoring validation results in `customValidate()`** -- Always return `Mono.just(ValidationResult.success())` on the happy path. Returning `Mono.empty()` or `null` will cause a `NullPointerException` in the validation pipeline.

10. **Heavy logic in `authorize()`** -- Authorization runs before every command/query execution. Keep it fast. Avoid database round-trips in `authorize()` when possible; prefer caching authorization decisions or using lightweight token-based checks.
