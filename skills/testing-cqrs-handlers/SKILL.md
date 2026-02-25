---
name: testing-cqrs-handlers
description: Use when writing tests for CQRS command or query handlers — covers unit tests with mocked bus, integration tests with authorization and validation, event emission assertions, and ExecutionContext testing
---

# Testing CQRS Handlers

Testing patterns for command handlers, query handlers, and the CQRS pipeline. For general reactive testing fundamentals (StepVerifier, Testcontainers, WireMock, WebTestClient), see the `testing-reactive-services` skill.

## Unit Testing Command Handlers

Build the full pipeline manually. Mock only `ApplicationContext`; use real service instances:

```java
class CommandBusTest {
    private CommandBus commandBus;
    private MeterRegistry meterRegistry;

    @BeforeEach
    void setUp() {
        ApplicationContext ctx = mock(ApplicationContext.class);
        meterRegistry = new SimpleMeterRegistry();

        CommandHandlerRegistry registry = new CommandHandlerRegistry(ctx);
        CommandValidationService validation = new CommandValidationService(new AutoValidationProcessor(null));
        CommandMetricsService metrics = new CommandMetricsService(meterRegistry);
        AuthorizationService authz = new AuthorizationService(
            TestAuthorizationProperties.createDefault(), Optional.empty());

        commandBus = new DefaultCommandBus(registry, validation, authz, metrics, new CorrelationContext());

        // Register handlers manually (simulates @CommandHandlerComponent auto-discovery)
        ((DefaultCommandBus) commandBus).registerHandler(new CreateAccountHandler());
        ((DefaultCommandBus) commandBus).registerHandler(new TransferMoneyHandler());
    }

    @Test
    void testCreateAccountCommand() {
        var command = new CreateAccountCommand("CUST-123", "SAVINGS", new BigDecimal("1000.00"));

        StepVerifier.create(commandBus.send(command))
            .expectNextMatches(result -> {
                AccountCreatedResult r = (AccountCreatedResult) result;
                return r.getAccountNumber().startsWith("ACC-") &&
                       r.getCustomerId().equals("CUST-123") &&
                       r.getAccountType().equals("SAVINGS") &&
                       r.getStatus().equals("ACTIVE");
            })
            .verifyComplete();
    }

    @Test
    void testBuiltInMetrics() {
        var command = new CreateAccountCommand("METRICS-TEST", "CHECKING", new BigDecimal("250.00"));
        StepVerifier.create(commandBus.send(command)).expectNextCount(1).verifyComplete();
        assertThat(meterRegistry.getMeters()).isNotEmpty();
    }
}
```

### Testing a handler directly (bypass the bus)

When you only need handler business logic, instantiate the handler directly:

```java
@Test
void testHandlerWithoutContext() {
    var command = new CreateAccountCommand("CUST-123", "SAVINGS", new BigDecimal("1000.00"));
    var handler = new FlexibleAccountHandler();

    StepVerifier.create(handler.doHandle(command))
        .expectNextMatches(result -> {
            assertThat(result.getAccountNumber()).startsWith("ACC-");
            assertThat(result.getCustomerId()).isEqualTo("CUST-123");
            assertThat(result.getStatus()).isEqualTo("ACTIVE");
            return true;
        })
        .verifyComplete();
}
```

## Unit Testing Query Handlers

Pass `null` for cache adapter and MeterRegistry to isolate handler logic:

```java
class QueryBusTest {
    private QueryBus queryBus;

    @BeforeEach
    void setUp() {
        ApplicationContext ctx = mock(ApplicationContext.class);
        queryBus = new DefaultQueryBus(ctx, new CorrelationContext(), new AutoValidationProcessor(null),
            new AuthorizationService(TestAuthorizationProperties.createDefault(), Optional.empty()),
            null, null); // no cache, no metrics

        ((DefaultQueryBus) queryBus).registerHandler(new GetAccountBalanceHandler());
    }

    @Test
    void testGetAccountBalanceQuery() {
        StepVerifier.create(queryBus.query(new GetAccountBalanceQuery("ACC-123", "CUST-456")))
            .expectNextMatches(result -> {
                AccountBalance b = (AccountBalance) result;
                return b.getAccountNumber().equals("ACC-123") &&
                       b.getCurrentBalance().equals(new BigDecimal("2500.00")) &&
                       b.getCurrency().equals("USD");
            })
            .verifyComplete();
    }

    @Test
    void testPagination() {
        StepVerifier.create(queryBus.query(new GetTransactionHistoryQuery("ACC-789", 5, 1)))
            .expectNextMatches(result -> {
                TransactionHistory h = (TransactionHistory) result;
                return h.getPageSize() == 5 && h.getPageNumber() == 1;
            })
            .verifyComplete();
    }
}
```

## Mocking the CommandBus in Higher-Level Tests

```java
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {
    @Mock private CommandBus commandBus;

    @Test
    void testExecuteWithCommandBus() {
        when(commandBus.send(any())).thenReturn(Mono.just("EXECUTED"));

        StepVerifier.create(CommandBuilder.create(TestCommand.class)
                .withCustomerId("CUST-EXEC").withAmount(new BigDecimal("100.00")).executeWith(commandBus))
            .expectNext("EXECUTED")
            .verifyComplete();
    }
}
```

## Authorization Tests

### AuthorizationResult unit tests (non-reactive)

```java
@Test
void shouldCreateSuccessfulResult() {
    AuthorizationResult result = AuthorizationResult.success();
    assertThat(result.isAuthorized()).isTrue();
    assertThat(result.getErrors()).isEmpty();
}

@Test
void shouldCreateFailedResultWithMultipleErrors() {
    AuthorizationResult result = AuthorizationResult.failure(List.of(
        AuthorizationError.of("account", "Account access denied"),
        AuthorizationError.of("permission", "Insufficient permissions")));

    assertThat(result.isAuthorized()).isFalse();
    assertThat(result.getErrors()).hasSize(2);
    assertThat(result.getErrorMessages()).isEqualTo("Account access denied; Insufficient permissions");
}

@Test
void shouldBuildWithBuilder() {
    AuthorizationResult result = AuthorizationResult.builder()
        .addError("account", "Account denied", "ACCOUNT_DENIED")
        .addError("permission", "Insufficient", "PERMISSION_DENIED")
        .summary("Custom failure").build();

    assertThat(result.getErrors().get(0).getErrorCode()).isEqualTo("ACCOUNT_DENIED");
}

@Test
void shouldCombineResults() {
    AuthorizationResult combined = AuthorizationResult.failure("account", "Error 1")
        .combine(AuthorizationResult.failure("permission", "Error 2"));
    assertThat(combined.getErrors()).hasSize(2);
}
```

### AuthorizationService reactive tests

Commands implement `authorize()` and `authorize(ExecutionContext)`. The bus calls them automatically:

```java
class AuthorizationServiceTest {
    private AuthorizationService authorizationService;

    @BeforeEach
    void setUp() {
        authorizationService = new AuthorizationService(
            TestAuthorizationProperties.createDefault(), Optional.empty());
    }

    @Test
    void shouldAuthorizeValidCommand() {
        StepVerifier.create(authorizationService.authorizeCommand(new TestCommand("ACC-123", new BigDecimal("1000"))))
            .verifyComplete();
    }

    @Test
    void shouldRejectForbiddenCommand() {
        StepVerifier.create(authorizationService.authorizeCommand(new TestCommand("ACC-999", new BigDecimal("1000"))))
            .expectError(AuthorizationException.class).verify();
    }

    @Test
    void shouldAuthorizeWithContext() {
        ExecutionContext ctx = ExecutionContext.builder().withUserId("USER-123").withTenantId("TENANT-456").build();
        StepVerifier.create(authorizationService.authorizeCommand(new TestCommand("ACC-123", new BigDecimal("1000")), ctx))
            .verifyComplete();
    }

    @Test
    void shouldRejectForbiddenUser() {
        ExecutionContext ctx = ExecutionContext.builder().withUserId("USER-999").withTenantId("TENANT-999").build();
        StepVerifier.create(authorizationService.authorizeCommand(new TestCommand("ACC-123", new BigDecimal("1000")), ctx))
            .expectError(AuthorizationException.class).verify();
    }
}
```

### Command-level authorize() pattern

```java
static class BankTransferCommand implements Command<BankTransferResult> {
    private final String sourceAccountId, targetAccountId;
    private final BigDecimal amount;

    @Override
    public Mono<AuthorizationResult> authorize() {
        if ("ACC-FORBIDDEN".equals(sourceAccountId)) {
            return Mono.just(AuthorizationResult.failure("account", "Account access forbidden"));
        }
        return Mono.just(AuthorizationResult.success());
    }

    @Override
    public Mono<AuthorizationResult> authorize(ExecutionContext context) {
        if ("USER-FORBIDDEN".equals(context.getUserId())) {
            return Mono.just(AuthorizationResult.failure("user", "User access forbidden"));
        }
        if (amount.compareTo(new BigDecimal("5000")) > 0 &&
            !context.getFeatureFlag("high-value-transfers", false)) {
            return Mono.just(AuthorizationResult.failure("amount", "High-value transfers not enabled"));
        }
        return authorize();
    }
}
```

### Authorization integration test through the full pipeline

```java
@Test
void shouldProcessWhenAuthorized() {
    StepVerifier.create(commandBus.send(new BankTransferCommand("ACC-123", "ACC-456", new BigDecimal("1000"))))
        .expectNextMatches(r -> ((BankTransferResult) r).getStatus().equals("COMPLETED"))
        .verifyComplete();
}

@Test
void shouldFailWhenAccountForbidden() {
    StepVerifier.create(commandBus.send(new BankTransferCommand("ACC-FORBIDDEN", "ACC-456", new BigDecimal("1000"))))
        .expectError(AuthorizationException.class).verify();
}

@Test
void shouldFailWithContextWhenUserForbidden() {
    ExecutionContext ctx = ExecutionContext.builder().withUserId("USER-FORBIDDEN").withTenantId("TENANT-456").build();
    StepVerifier.create(commandBus.send(new BankTransferCommand("ACC-123", "ACC-456", new BigDecimal("1000")), ctx))
        .expectError(AuthorizationException.class).verify();
}
```

## Validation Tests

### Jakarta Bean Validation on Commands

Commands declare Jakarta annotations. `CommandValidationService` processes them via `AutoValidationProcessor`:

```java
public class CreateAccountCommand implements Command<AccountCreatedResult> {
    @NotBlank private final String customerId;
    @NotBlank private final String accountType;
    @NotNull @Positive private final BigDecimal initialBalance;
}
```

### Custom business validation via customValidate()

```java
@Override
public Mono<ValidationResult> customValidate() {
    ValidationResult.Builder builder = ValidationResult.builder();
    if (transferAmount.compareTo(accountBalance) > 0) {
        builder.addError("transferAmount", "Insufficient funds", "INSUFFICIENT_FUNDS");
    }
    return Mono.just(builder.build());
}
```

### Testing CommandValidationService directly

```java
@Test
void shouldPassValidation() {
    var svc = new CommandValidationService(new AutoValidationProcessor(null));
    StepVerifier.create(svc.validateCommand(new CreateAccountCommand("CUST-123", "SAVINGS", new BigDecimal("1000.00"))))
        .verifyComplete();
}

@Test
void shouldReturnResultWithoutException() {
    var svc = new CommandValidationService(new AutoValidationProcessor(null));
    StepVerifier.create(svc.validateCommandWithResult(new CreateAccountCommand("CUST-123", "SAVINGS", new BigDecimal("1000.00"))))
        .assertNext(result -> {
            assertThat(result.isValid()).isTrue();
            assertThat(result.getErrors()).isEmpty();
        }).verifyComplete();
}
```

### ValidationResult and ValidationException assertions

```java
@Test
void shouldBuildFailedValidationResult() {
    ValidationResult result = ValidationResult.builder()
        .addError("amount", "Must be positive", "POSITIVE_REQUIRED")
        .addError("customerId", "Required field", "NOT_BLANK")
        .build();
    assertThat(result.isInvalid()).isTrue();
    assertThat(result.getErrors()).hasSize(2);
    assertThat(result.getFirstError().getFieldName()).isEqualTo("amount");
}

@Test
void shouldThrowValidationException() {
    StepVerifier.create(commandBus.send(invalidCommand))
        .expectErrorSatisfies(error -> {
            assertThat(error).isInstanceOf(ValidationException.class);
            ValidationException ve = (ValidationException) error;
            assertThat(ve.getValidationResult().isInvalid()).isTrue();
            assertThat(ve.hasFieldErrors()).isTrue();
        }).verify();
}
```

## Event Emission Tests

### @PublishDomainEvent annotation

Handlers annotated with `@PublishDomainEvent` auto-publish results. The `DefaultCommandBus` calls `CommandEventPublisher.publish()` if present. Publishing failures are logged but do NOT fail the command.

```java
@CommandHandlerComponent
@PublishDomainEvent(destination = "user-events", eventType = "UserCreated")
public class CreateUserCommandHandler extends CommandHandler<CreateUserCommand, UserCreatedResult> {
    @Override
    protected Mono<UserCreatedResult> doHandle(CreateUserCommand command) {
        return userService.createUser(command);
    }
}
```

### Mocking CommandEventPublisher for event verification

Inject a mock via `ReflectionTestUtils` (the field is `@Autowired(required=false)` on `DefaultCommandBus`):

```java
@ExtendWith(MockitoExtension.class)
class EventEmissionTest {
    @Mock private CommandEventPublisher eventPublisher;
    private CommandBus commandBus;

    @BeforeEach
    void setUp() {
        // Build bus as usual (see command handler wiring above)
        DefaultCommandBus bus = new DefaultCommandBus(registry, validation, authz, metrics, correlation);
        ReflectionTestUtils.setField(bus, "commandEventPublisher", eventPublisher);
        bus.registerHandler(new CreateUserCommandHandler());
        commandBus = bus;
    }

    @Test
    void shouldPublishDomainEventAfterSuccess() {
        when(eventPublisher.publish(any(), any(), any())).thenReturn(Mono.empty());

        StepVerifier.create(commandBus.send(new CreateUserCommand("user@example.com", "John")))
            .expectNextCount(1).verifyComplete();

        verify(eventPublisher).publish(any(CreateUserCommand.class), any(UserCreatedResult.class),
            argThat(a -> a.destination().equals("user-events") && a.eventType().equals("UserCreated")));
    }

    @Test
    void shouldNotPublishWhenHandlerLacksAnnotation() {
        ((DefaultCommandBus) commandBus).registerHandler(new PlainCommandHandler());
        StepVerifier.create(commandBus.send(new PlainCommand())).expectNextCount(1).verifyComplete();
        verifyNoInteractions(eventPublisher);
    }

    @Test
    void shouldSucceedEvenWhenPublishingFails() {
        when(eventPublisher.publish(any(), any(), any()))
            .thenReturn(Mono.error(new RuntimeException("Kafka unavailable")));
        StepVerifier.create(commandBus.send(new CreateUserCommand("user@example.com", "John")))
            .expectNextCount(1).verifyComplete();
    }
}
```

### EdaCommandEventPublisher header verification

When testing the real EDA publisher, verify headers (`eventType`, `commandId`, `commandType`, `correlationId`):

```java
@Test
void shouldPublishWithCorrectHeaders() {
    when(publisherFactory.getPublisher(PublisherType.AUTO)).thenReturn(mockPublisher);
    when(mockPublisher.isAvailable()).thenReturn(true);
    when(mockPublisher.publish(any(), eq("user-events"), any())).thenReturn(Mono.empty());

    StepVerifier.create(edaPublisher.publish(command, result, annotation)).verifyComplete();

    ArgumentCaptor<Map<String, Object>> captor = ArgumentCaptor.forClass(Map.class);
    verify(mockPublisher).publish(eq(result), eq("user-events"), captor.capture());
    assertThat(captor.getValue()).containsEntry("eventType", "UserCreated")
        .containsEntry("commandType", "CreateUserCommand").containsKey("commandId");
}
```

## ExecutionContext Tests

### Building, verifying, and edge cases

```java
@Test
void testFullContext() {
    ExecutionContext ctx = ExecutionContext.builder()
        .withUserId("user-123").withTenantId("tenant-456").withSource("mobile-app")
        .withFeatureFlag("new-feature", true).withProperty("priority", "HIGH")
        .withProperty("customData", 42).build();

    assertThat(ctx.getFeatureFlag("new-feature", false)).isTrue();
    assertThat(ctx.getFeatureFlag("unknown", true)).isTrue(); // returns default
    assertThat(ctx.getProperty("priority")).isEqualTo(Optional.of("HIGH"));
    assertThat(ctx.getProperty("customData", Integer.class)).isEqualTo(Optional.of(42));
    assertThat(ctx.getProperty("missing")).isEqualTo(Optional.empty());
}

@Test
void testEmptyContext() {
    ExecutionContext ctx = ExecutionContext.empty();
    assertThat(ctx.getUserId()).isNull();
    assertThat(ctx.hasProperties()).isFalse();
}

@Test
void testImmutability() {
    ExecutionContext ctx = ExecutionContext.builder().withProperty("k", "v").withFeatureFlag("f", true).build();
    assertThatThrownBy(() -> ctx.getProperties().clear()).isInstanceOf(UnsupportedOperationException.class);
}

@Test
void testPropertyTypeConversion() {
    ExecutionContext ctx = ExecutionContext.builder().withProperty("str", "hello").withProperty("num", 42).build();
    assertThat(ctx.getProperty("str", String.class)).isEqualTo(Optional.of("hello"));
    assertThat(ctx.getProperty("str", Integer.class)).isEqualTo(Optional.empty()); // wrong type
}
```

### Passing context through the bus

Both `commandBus.send(command, context)` and `queryBus.query(query, context)` propagate context to the handler's `doHandle(command, context)` overload:

```java
@Test
void testCommandWithContext() {
    ExecutionContext ctx = ExecutionContext.builder()
        .withUserId("user-456").withTenantId("premium-tenant").withSource("mobile-app")
        .withFeatureFlag("premium-features", true).withFeatureFlag("auto-approve", true).build();

    StepVerifier.create(handler.doHandle(new CreateTenantAccountCommand("CUST-123", "SAVINGS", new BigDecimal("1000.00")), ctx))
        .assertNext(r -> {
            assertThat(r.getTenantId()).isEqualTo("premium-tenant");
            assertThat(r.getStatus()).isEqualTo("ACTIVE");
            assertThat(r.isPremiumFeatures()).isTrue();
        }).verifyComplete();
}

@Test
void testMissingRequiredContext() {
    ExecutionContext ctx = ExecutionContext.builder().withUserId("user-456").build(); // no tenantId
    StepVerifier.create(handler.doHandle(command, ctx))
        .expectError(IllegalArgumentException.class).verify();
}

@Test
void testContextAwareHandlerWithoutContext() {
    StepVerifier.create(handler.handle(command))
        .expectError(UnsupportedOperationException.class).verify();
}

@Test
void testFeatureFlagBranching() {
    ExecutionContext ctx = ExecutionContext.builder()
        .withUserId("user-456").withTenantId("basic-tenant")
        .withFeatureFlag("premium-features", false).build();

    StepVerifier.create(handler.doHandle(new CreateTenantAccountCommand("CUST-123", "SAVINGS", new BigDecimal("15000.00")), ctx))
        .assertNext(r -> {
            assertThat(r.getStatus()).isEqualTo("PENDING_APPROVAL");
            assertThat(r.isPremiumFeatures()).isFalse();
        }).verifyComplete();
}
```

## Metrics Assertions

### Spring integration test with named metric counters

```java
@SpringBootTest(classes = TestConfiguration.class)
class SpringCqrsIntegrationTest {
    @Autowired private CommandBus commandBus;
    @Autowired private MeterRegistry meterRegistry;

    @Test
    void shouldCollectCommandMetrics() {
        StepVerifier.create(commandBus.send(new CreateAccountCommand("BIZ-123", "CHECKING", new BigDecimal("2500.00"))))
            .expectNextCount(1).verifyComplete();

        assertThat(meterRegistry.find("firefly.cqrs.command.processed").counter()).isNotNull();
        assertThat(meterRegistry.find("firefly.cqrs.command.processed").counter().count()).isGreaterThan(0);
        assertThat(meterRegistry.find("firefly.cqrs.command.processing.time").timer().count()).isGreaterThan(0);
    }

    @Configuration
    @Import({ CqrsAutoConfiguration.class, CreateAccountHandler.class, GetAccountBalanceHandler.class })
    static class TestConfiguration { }
}
```

### Metric names reference

| Metric | Type | Description |
|---|---|---|
| `firefly.cqrs.command.processed` | Counter | Successful commands |
| `firefly.cqrs.command.failed` | Counter | Failed commands |
| `firefly.cqrs.command.validation.failed` | Counter | Validation failures |
| `firefly.cqrs.command.processing.time` | Timer | Processing duration |
| `firefly.cqrs.command.type.processed` | Counter | Per-type success (tag: `command.type`) |
| `firefly.cqrs.command.type.failed` | Counter | Per-type failure (tag: `command.type`) |
| `firefly.cqrs.query.processed` | Counter | Successful queries |
| `firefly.cqrs.query.processing.time` | Timer | Query duration |

### TestAuthorizationProperties utility

`createDefault()` -- authorization enabled, custom on. `createDisabled()` -- bypass all. `createStrict()` -- shorter timeouts. `createVerbose()` -- log successes.

## Anti-patterns

**Do not test the bus pipeline when you only need handler logic.** Call `handler.doHandle(command)` directly. Reserve `commandBus.send()` for integration tests that verify the full pipeline.

**Do not create a Spring context for every handler test.** The manual wiring pattern (`new DefaultCommandBus(...)` + `registerHandler()`) is faster. Use `@SpringBootTest` only when testing auto-discovery or full EDA integration.

**Do not conflate AuthorizationException with ValidationException.** They are distinct. Authorization runs after validation. Assert the correct type:

```java
.expectError(AuthorizationException.class)  // correct
.expectError(ValidationException.class)     // correct
.expectError(RuntimeException.class)        // too broad
```

**Do not forget to verify event publishing side effects.** If a handler has `@PublishDomainEvent`, verify the mock publisher was called. If it lacks the annotation, assert `verifyNoInteractions(eventPublisher)`.

**Do not hardcode `System.currentTimeMillis()` expectations.** Use `startsWith("ACC-")` for non-deterministic IDs.

**Do not skip `.verify()` or `.verifyComplete()`.** Without a terminal call the reactive chain is never subscribed.

**Do not mutate ExecutionContext across tests.** It is immutable. Build fresh per test via `ExecutionContext.builder()`.

**Do not test caching without a cache adapter.** Pass `null` to isolate handler logic. To test caching, mock `FireflyCacheManager` and verify interactions:

```java
@Mock private FireflyCacheManager cacheManager;

@Test
void shouldUseCachedValue() {
    when(cacheManager.get(":cqrs:test-key", TestResult.class)).thenReturn(Mono.just(Optional.of(expectedResult)));
    StepVerifier.create(queryCacheAdapter.get("test-key", TestResult.class))
        .assertNext(r -> assertThat(r.getId()).isEqualTo("123")).verifyComplete();
    verify(cacheManager).get(":cqrs:test-key", TestResult.class);
}
```
