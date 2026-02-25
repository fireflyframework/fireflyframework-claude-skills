---
name: testing-reactive-services
description: Use when writing tests for Firefly Framework reactive services — covers StepVerifier patterns, Testcontainers setup for R2DBC/Kafka/Redis, WireMock for service mocking, WebTestClient for controller tests, and reactive context testing
---

# Testing Reactive Services

## Test Pyramid

Firefly Framework services are fully reactive (Spring WebFlux, R2DBC, Project Reactor). Every layer of the test pyramid must account for this.

- **Unit tests (StepVerifier)** -- Test individual service methods, command/query handlers, and reactive operators in isolation. Mock dependencies with Mockito. Use `StepVerifier` for all `Mono`/`Flux` assertions. Fast, no Spring context needed. This is where most tests should live.
- **Integration tests (Testcontainers)** -- Test repository-to-database flows, event publishing through Kafka/RabbitMQ, and event store operations. Use `@Testcontainers` with PostgreSQL, Kafka, or RabbitMQ containers. Use `@DynamicPropertySource` to inject container connection details. Slower, but validates real infrastructure interactions.
- **Controller tests (WebTestClient)** -- Test HTTP endpoints including serialization, error handling, header propagation, and status codes. Use `@SpringBootTest(webEnvironment = RANDOM_PORT)` with `WebTestClient` or test directly with `MockServerWebExchange`. Validates the full request-response cycle.

Choose the lowest level that validates the behavior you care about. Test business logic with unit tests; reserve integration tests for infrastructure boundaries.

## StepVerifier Patterns

All reactive return types must be verified with `StepVerifier`. Never call `.block()` in assertions.

### Testing Mono -- expect a single value

```java
StepVerifier.create(commandBus.send(command))
    .expectNextMatches(result -> {
        AccountCreatedResult accountResult = (AccountCreatedResult) result;
        return accountResult.getAccountNumber().startsWith("ACC-") &&
               accountResult.getCustomerId().equals("CUST-123") &&
               accountResult.getAccountType().equals("SAVINGS") &&
               accountResult.getInitialBalance().equals(new BigDecimal("1000.00")) &&
               accountResult.getStatus().equals("ACTIVE");
    })
    .verifyComplete();
```

### Testing Mono -- assertNext with AssertJ (preferred for complex assertions)

```java
StepVerifier.create(filter.filter(request))
    .assertNext(response -> {
        assertThat(response.getContent()).hasSize(1);
        assertThat(response.getContent().get(0).getName()).isEqualTo("test name");
        assertThat(response.getTotalElements()).isEqualTo(1);
    })
    .verifyComplete();
```

### Testing error signals

```java
// Expect a specific exception type
StepVerifier.create(commandBus.send(command))
    .expectError(AuthorizationException.class)
    .verify();

// Expect error after exhausting retries
StepVerifier.create(retryMono)
    .expectError(RuntimeException.class)
    .verify();
```

### Testing Mono<Void> completion (filters, event publishing)

```java
StepVerifier.create(kafkaPublisher.publish(event, topic))
    .verifyComplete();

StepVerifier.create(exceptionHandler.handle(exchange, exception))
    .verifyComplete();
```

### Testing Flux -- expectNextCount and stream assertions

```java
StepVerifier.create(eventStore.streamAllEvents().take(Duration.ofSeconds(5)))
    .expectNextCount(2)
    .verifyComplete();

StepVerifier.create(eventStore.streamEventsByType(List.of("test.money.withdrawn")))
    .assertNext(envelope -> {
        assertEquals("test.money.withdrawn", envelope.getEventType());
    })
    .verifyComplete();
```

### Testing boolean/scalar Mono results

```java
StepVerifier.create(eventStore.aggregateExists(aggregateId, "Account"))
    .expectNext(true)
    .verifyComplete();

StepVerifier.create(eventStore.getAggregateVersion(aggregateId, "Account"))
    .expectNext(1L)
    .verifyComplete();
```

## Testcontainers Setup

### PostgreSQL R2DBC Container

Used for event store and repository integration tests. Configure the R2DBC connection factory from container properties.

```java
@Testcontainers
class PostgreSqlEventStoreIntegrationTest {

    @Container
    static final PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15-alpine")
            .withDatabaseName("firefly_test")
            .withUsername("firefly_test")
            .withPassword("test_password")
            .withReuse(false);

    private ConnectionFactory connectionFactory;
    private DatabaseClient databaseClient;

    @BeforeEach
    void setUp() {
        connectionFactory = new PostgresqlConnectionFactory(
            PostgresqlConnectionConfiguration.builder()
                .host(postgres.getHost())
                .port(postgres.getFirstMappedPort())
                .database(postgres.getDatabaseName())
                .username(postgres.getUsername())
                .password(postgres.getPassword())
                .build()
        );
        databaseClient = DatabaseClient.create(connectionFactory);
        // Create schema before tests
        createSchema().block();
    }

    @AfterEach
    void tearDown() {
        databaseClient.sql("TRUNCATE TABLE events RESTART IDENTITY CASCADE")
            .fetch().rowsUpdated().block();
    }
}
```

### Kafka Container with @ServiceConnection

The framework uses `@TestConfiguration` with `@ServiceConnection` for cleaner container wiring.

```java
@TestConfiguration(proxyBeanMethods = false)
public class TestContainersConfiguration {

    @Bean
    @ServiceConnection
    public KafkaContainer kafkaContainer() {
        KafkaContainer kafka = new KafkaContainer(
                DockerImageName.parse("confluentinc/cp-kafka:7.5.0")
        );
        kafka.withReuse(true);
        return kafka;
    }

    @Bean
    @ServiceConnection
    public RabbitMQContainer rabbitMQContainer() {
        RabbitMQContainer rabbitmq = new RabbitMQContainer(
                DockerImageName.parse("rabbitmq:3.12-management-alpine")
        );
        rabbitmq.withReuse(true);
        return rabbitmq;
    }
}
```

### @DynamicPropertySource for custom properties

When `@ServiceConnection` does not cover all needed properties, use `@DynamicPropertySource`.

```java
@DynamicPropertySource
static void kafkaProperties(DynamicPropertyRegistry registry) {
    registry.add("firefly.eda.publishers.kafka.default.enabled", () -> "true");
    registry.add("firefly.eda.publishers.kafka.default.bootstrap-servers",
        kafka::getBootstrapServers);
    registry.add("firefly.eda.publishers.kafka.default.default-topic",
        () -> "test-events");
}
```

### Reusable containers

Enable reusable containers in `src/test/resources/testcontainers.properties`:

```properties
testcontainers.reuse.enable=true
```

Then set `.withReuse(true)` on the container instance. This keeps the container running between test runs during development.

## Repository Tests

### R2DBC repository testing with H2

For fast unit-level repository tests, use H2 in-memory R2DBC:

```java
@BeforeEach
void setUp() {
    connectionFactory = new H2ConnectionFactory(
        H2ConnectionConfiguration.builder()
            .url("mem:testdb;DB_CLOSE_DELAY=-1")
            .username("sa")
            .build()
    );
    databaseClient = DatabaseClient.create(connectionFactory);
    entityTemplate = new R2dbcEntityTemplate(connectionFactory);
    transactionManager = new R2dbcTransactionManager(connectionFactory);
    transactionalOperator = TransactionalOperator.create(transactionManager);
}
```

### Schema initialization in tests

Create tables directly in `@BeforeEach` for integration tests:

```java
private Mono<Void> createSchema() {
    String sql = """
        CREATE TABLE IF NOT EXISTS events (
            event_id UUID PRIMARY KEY,
            aggregate_id UUID NOT NULL,
            aggregate_type VARCHAR(255) NOT NULL,
            aggregate_version BIGINT NOT NULL,
            global_sequence BIGSERIAL UNIQUE,
            event_type VARCHAR(255) NOT NULL,
            event_data TEXT NOT NULL,
            metadata TEXT,
            created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
            CONSTRAINT unique_aggregate_version UNIQUE(aggregate_id, aggregate_version)
        )
        """;
    return databaseClient.sql(sql).fetch().rowsUpdated().then();
}
```

Or place SQL files in `src/test/resources/db/test-schema.sql` for more complex schemas.

### Filter and query testing with mocked R2dbcEntityTemplate

```java
@ExtendWith(MockitoExtension.class)
public class FilterUtilsTest {

    @Mock
    private R2dbcEntityTemplate entityTemplate;
    @Mock
    private ReactiveSelect<TestEntity> reactiveSelect;
    @Mock
    private TerminatingSelect<TestEntity> terminatingSelect;

    @BeforeEach
    void setUp() {
        FilterUtils.initializeTemplate(entityTemplate);
        when(entityTemplate.select(TestEntity.class)).thenReturn(reactiveSelect);
        when(reactiveSelect.matching(any(Query.class))).thenReturn(terminatingSelect);
    }

    @Test
    void testStringFiltering() {
        TestEntity entity = TestEntity.builder().id(TEST_ID).name("test name").build();
        when(terminatingSelect.all()).thenReturn(Flux.just(entity));
        when(terminatingSelect.count()).thenReturn(Mono.just(1L));

        StepVerifier.create(filter.filter(request))
            .assertNext(response -> {
                assertThat(response.getContent()).hasSize(1);
                assertThat(response.getContent().get(0).getName()).isEqualTo("test name");
            })
            .verifyComplete();

        // Verify generated SQL criteria
        ArgumentCaptor<Query> queryCaptor = ArgumentCaptor.forClass(Query.class);
        verify(reactiveSelect, atLeastOnce()).matching(queryCaptor.capture());
        Criteria criteria = (Criteria) queryCaptor.getValue().getCriteria().orElse(Criteria.empty());
        assertThat(criteria.toString()).contains("name LIKE '%test%'");
    }
}
```

## Service Tests

### Testing CQRS command handlers

Register handlers manually, mock infrastructure, use StepVerifier for all assertions.

```java
class CommandBusTest {

    private CommandBus commandBus;

    @BeforeEach
    void setUp() {
        ApplicationContext applicationContext = mock(ApplicationContext.class);
        CorrelationContext correlationContext = new CorrelationContext();
        AutoValidationProcessor validationProcessor = new AutoValidationProcessor(null);
        SimpleMeterRegistry meterRegistry = new SimpleMeterRegistry();

        CommandHandlerRegistry handlerRegistry = new CommandHandlerRegistry(applicationContext);
        CommandValidationService validationService = new CommandValidationService(validationProcessor);
        CommandMetricsService metricsService = new CommandMetricsService(meterRegistry);

        commandBus = new DefaultCommandBus(handlerRegistry, validationService,
            new AuthorizationService(TestAuthorizationProperties.createDefault(), Optional.empty()),
            metricsService, correlationContext);

        ((DefaultCommandBus) commandBus).registerHandler(new CreateAccountHandler());
    }

    @Test
    void testCreateAccountCommand() {
        var command = new CreateAccountCommand("CUST-123", "SAVINGS", new BigDecimal("1000.00"));

        StepVerifier.create(commandBus.send(command))
            .expectNextMatches(result -> {
                AccountCreatedResult r = (AccountCreatedResult) result;
                return r.getAccountNumber().startsWith("ACC-") &&
                       r.getCustomerId().equals("CUST-123");
            })
            .verifyComplete();
    }
}
```

### Testing reactive services with Mockito mocks

Use `@ExtendWith(MockitoExtension.class)` and `@Mock`. Return `Mono.just()`, `Mono.empty()`, or `Mono.error()` from mocked reactive dependencies.

```java
@ExtendWith(MockitoExtension.class)
class ResiliencePatternTest {

    @Mock
    private EventPublisher delegatePublisher;

    @Test
    void shouldRetryOnFailureAndSucceedEventually() {
        AtomicInteger attemptCount = new AtomicInteger(0);
        when(delegatePublisher.publish(eq(testEvent), eq("test-topic"), eq(testHeaders)))
            .thenAnswer(invocation -> {
                int attempt = attemptCount.incrementAndGet();
                if (attempt < 3) {
                    return Mono.error(new RuntimeException("Temporary failure"));
                }
                return Mono.empty();
            });

        Mono<Void> retryMono = Mono.fromSupplier(
                () -> delegatePublisher.publish(testEvent, "test-topic", testHeaders))
            .flatMap(mono -> mono)
            .transformDeferred(RetryOperator.of(retry));

        StepVerifier.create(retryMono)
            .verifyComplete();

        verify(delegatePublisher, times(3)).publish(testEvent, "test-topic", testHeaders);
    }
}
```

## Controller Tests

### Testing with MockServerWebExchange (unit-level)

For WebFlux filters and exception handlers, use `MockServerHttpRequest` and `MockServerWebExchange`:

```java
@Test
void handleBusinessException() {
    BusinessException exception = new BusinessException(
        HttpStatus.BAD_REQUEST, "ERROR_CODE", "Error message");
    MockServerWebExchange exchange = MockServerWebExchange.from(
        MockServerHttpRequest.get("/api/test").build());

    StepVerifier.create(exceptionHandler.handle(exchange, exception))
        .verifyComplete();

    assertEquals(HttpStatus.BAD_REQUEST, exchange.getResponse().getStatusCode());
    assertEquals(MediaType.APPLICATION_JSON,
        exchange.getResponse().getHeaders().getContentType());
}
```

### Testing WebFlux filters

```java
@Test
void shouldAllowFirstRequestWithIdempotencyKey() {
    String idempotencyKey = "test-key-1";
    MockServerHttpRequest request = MockServerHttpRequest
        .post("/test")
        .header("X-Idempotency-Key", idempotencyKey)
        .contentType(MediaType.APPLICATION_JSON)
        .build();
    MockServerWebExchange exchange = MockServerWebExchange.from(request);

    WebFilterChain filterChain = mock(WebFilterChain.class);
    when(filterChain.filter(any())).thenReturn(Mono.empty());

    StepVerifier.create(idempotencyWebFilter.filter(exchange, filterChain))
        .verifyComplete();

    verify(filterChain, times(1)).filter(any());
}
```

### Testing with WebTestClient (full integration)

For full integration tests that boot the application context:

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@ActiveProfiles("test")
class AccountControllerIntegrationTest {

    @Autowired
    private WebTestClient webTestClient;

    @Test
    void shouldCreateAccount() {
        webTestClient.post()
            .uri("/api/accounts")
            .contentType(MediaType.APPLICATION_JSON)
            .bodyValue(new CreateAccountRequest("CUST-123", "SAVINGS", new BigDecimal("1000")))
            .exchange()
            .expectStatus().isCreated()
            .expectHeader().contentType(MediaType.APPLICATION_JSON)
            .expectBody()
            .jsonPath("$.accountNumber").isNotEmpty()
            .jsonPath("$.customerId").isEqualTo("CUST-123")
            .jsonPath("$.status").isEqualTo("ACTIVE");
    }

    @Test
    void shouldReturnNotFoundForMissingResource() {
        webTestClient.get()
            .uri("/api/accounts/{id}", UUID.randomUUID())
            .exchange()
            .expectStatus().isNotFound()
            .expectBody()
            .jsonPath("$.code").isEqualTo("RESOURCE_NOT_FOUND")
            .jsonPath("$.status").isEqualTo(404);
    }

    @Test
    void shouldReturnValidationErrorForInvalidRequest() {
        webTestClient.post()
            .uri("/api/accounts")
            .contentType(MediaType.APPLICATION_JSON)
            .bodyValue("{\"customerId\": \"\"}")
            .exchange()
            .expectStatus().isBadRequest()
            .expectBody()
            .jsonPath("$.code").isEqualTo("VALIDATION_ERROR");
    }
}
```

## WireMock

### External service mocking setup

Use WireMock to simulate downstream services in integration tests.

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@ActiveProfiles("test")
class ExternalServiceIntegrationTest {

    private static WireMockServer wireMockServer;

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("external-service.base-url",
            () -> "http://localhost:" + wireMockServer.port());
    }

    @BeforeAll
    static void startWireMock() {
        wireMockServer = new WireMockServer(WireMockConfiguration.wireMockConfig()
            .dynamicPort());
        wireMockServer.start();
    }

    @AfterAll
    static void stopWireMock() {
        wireMockServer.stop();
    }

    @BeforeEach
    void resetStubs() {
        wireMockServer.resetAll();
    }

    @Test
    void shouldFetchCustomerFromExternalService() {
        // Stub the external service
        wireMockServer.stubFor(get(urlEqualTo("/api/customers/123"))
            .willReturn(aResponse()
                .withStatus(200)
                .withHeader("Content-Type", "application/json")
                .withBody("""
                    {"id": "123", "name": "John Doe", "email": "john@example.com"}
                    """)));

        // Test the reactive client call
        StepVerifier.create(customerClient.getCustomer("123"))
            .assertNext(customer -> {
                assertThat(customer.getName()).isEqualTo("John Doe");
                assertThat(customer.getEmail()).isEqualTo("john@example.com");
            })
            .verifyComplete();

        // Verify the request was made
        wireMockServer.verify(getRequestedFor(urlEqualTo("/api/customers/123")));
    }

    @Test
    void shouldHandleExternalServiceTimeout() {
        wireMockServer.stubFor(get(urlEqualTo("/api/customers/123"))
            .willReturn(aResponse()
                .withStatus(200)
                .withFixedDelay(5000)));  // Simulate timeout

        StepVerifier.create(customerClient.getCustomer("123"))
            .expectError(OperationTimeoutException.class)
            .verify(Duration.ofSeconds(10));
    }

    @Test
    void shouldHandleExternalService5xx() {
        wireMockServer.stubFor(get(urlEqualTo("/api/customers/123"))
            .willReturn(aResponse()
                .withStatus(503)
                .withBody("Service Unavailable")));

        StepVerifier.create(customerClient.getCustomer("123"))
            .expectError(ThirdPartyServiceException.class)
            .verify();
    }
}
```

## Reactive Context

### Testing with ExecutionContext (tenant, user, feature flags)

The framework propagates `ExecutionContext` through the CQRS pipeline. Tests must construct the context explicitly.

```java
@Test
void testCommandWithExecutionContext() {
    var command = new CreateTenantAccountCommand("CUST-123", "SAVINGS", new BigDecimal("1000.00"));

    ExecutionContext context = ExecutionContext.builder()
        .withUserId("user-456")
        .withTenantId("premium-tenant")
        .withSource("mobile-app")
        .withFeatureFlag("premium-features", true)
        .withFeatureFlag("auto-approve", true)
        .withProperty("priority", "HIGH")
        .build();

    StepVerifier.create(commandBus.send(command, context))
        .assertNext(result -> {
            assertThat(result.getTenantId()).isEqualTo("premium-tenant");
            assertThat(result.getStatus()).isEqualTo("ACTIVE");
            assertThat(result.isPremiumFeatures()).isTrue();
        })
        .verifyComplete();
}
```

### Testing context validation (missing required fields)

```java
@Test
void testContextValidation() {
    var command = new CreateTenantAccountCommand("CUST-123", "SAVINGS", new BigDecimal("1000.00"));

    // Context without required tenant ID
    ExecutionContext context = ExecutionContext.builder()
        .withUserId("user-456")
        .withSource("mobile-app")
        .build();

    StepVerifier.create(handler.doHandle(command, context))
        .expectError(IllegalArgumentException.class)
        .verify();
}
```

### Testing feature flag behavior

```java
@Test
void testFeatureFlagBehavior() {
    var command = new CreateTenantAccountCommand("CUST-123", "SAVINGS", new BigDecimal("15000.00"));

    ExecutionContext context = ExecutionContext.builder()
        .withUserId("user-456")
        .withTenantId("basic-tenant")
        .withFeatureFlag("premium-features", false)
        .build();

    StepVerifier.create(handler.doHandle(command, context))
        .expectNextMatches(result -> {
            assertThat(result.getStatus()).isEqualTo("PENDING_APPROVAL");
            assertThat(result.isPremiumFeatures()).isFalse();
            return true;
        })
        .verifyComplete();
}
```

### Testing authorization in the reactive pipeline

```java
@Test
void shouldFailCommandWhenAuthorizationFails() {
    var command = new BankTransferCommand("ACC-FORBIDDEN", "ACC-456", new BigDecimal("1000"));

    StepVerifier.create(commandBus.send(command))
        .expectError(AuthorizationException.class)
        .verify();
}

@Test
void shouldFailWithContextWhenUserIsForbidden() {
    var command = new BankTransferCommand("ACC-123", "ACC-456", new BigDecimal("1000"));
    ExecutionContext context = ExecutionContext.builder()
        .withUserId("USER-FORBIDDEN")
        .withTenantId("TENANT-456")
        .build();

    StepVerifier.create(commandBus.send(command, context))
        .expectError(AuthorizationException.class)
        .verify();
}
```

## Test Configuration

### Test application (minimal Spring Boot context)

```java
@SpringBootConfiguration
@EnableAutoConfiguration
public class TestApplication {
}
```

Or for scanning specific packages:

```java
@SpringBootApplication
@ComponentScan(basePackages = "org.fireflyframework.eda")
public class TestApplication {
    public static void main(String[] args) {
        SpringApplication.run(TestApplication.class, args);
    }
}
```

### Base integration test class

```java
@SpringBootTest(classes = TestApplication.class)
@ActiveProfiles("test")
public abstract class BaseIntegrationTest {

    protected static final Duration TEST_TIMEOUT = Duration.ofSeconds(30);
    protected static final Duration SHORT_TIMEOUT = Duration.ofSeconds(5);

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("firefly.eda.enabled", () -> "true");
        registry.add("spring.main.allow-bean-definition-overriding", () -> "true");
    }
}
```

### Test application-test.yml

```yaml
spring:
  main:
    banner-mode: off
  r2dbc:
    pool:
      enabled: true
      initial-size: 2
      max-size: 5
  flyway:
    enabled: false  # Use init script in Testcontainers

firefly:
  cqrs:
    enabled: true
  eda:
    enabled: true

logging:
  level:
    org.fireflyframework: DEBUG
    org.springframework: WARN
    reactor: WARN
    io.r2dbc: DEBUG
    org.testcontainers: INFO
```

### Test event models and data builders

Create test fixtures as static factory methods or builder classes in a shared test package:

```java
public class TestEventModels {

    public static class SimpleTestEvent {
        private String id;
        private String message;
        private Instant timestamp;

        public static SimpleTestEvent create(String message) {
            return new SimpleTestEvent(UUID.randomUUID().toString(), message, Instant.now());
        }
    }

    public static class OrderCreatedEvent {
        public static OrderCreatedEvent create(String customerId, Double amount) {
            return new OrderCreatedEvent(
                UUID.randomUUID().toString(), customerId, amount, "USD", Instant.now());
        }
    }

    public static class UserRegisteredEvent {
        public static Builder builder() { return new Builder(); }

        public static class Builder {
            public Builder userId(String userId) { /* ... */ return this; }
            public Builder email(String email) { /* ... */ return this; }
            public Builder tenantId(String tenantId) { /* ... */ return this; }
            public Builder premium(boolean premium) { /* ... */ return this; }
            public UserRegisteredEvent build() { /* ... */ }
        }
    }
}
```

## Anti-patterns

**Do not call `.block()` in test assertions.** Use `StepVerifier` for all reactive type verification. The only acceptable use of `.block()` is for test setup/teardown (schema creation, data cleanup) where the operation is not under test.

**Do not use `Thread.sleep()` for async coordination.** Use `Awaitility` with `await().atMost(...).untilAsserted(...)` when waiting for side effects like Kafka message delivery:

```java
await().atMost(Duration.ofSeconds(10))
    .pollInterval(Duration.ofMillis(100))
    .untilAsserted(() -> {
        var records = testConsumer.poll(Duration.ofMillis(100));
        records.forEach(consumedMessages::add);
        assertThat(consumedMessages).isNotEmpty();
    });
```

**Do not over-mock the reactive chain.** If you find yourself mocking `Mono.flatMap()` or chaining stubbed reactive operators, test at a higher level instead. Mock at the boundary (repository, external client), not in the middle of the pipeline.

**Do not test framework plumbing.** Test business behavior, not that Spring wiring works. If a test only verifies that `@Autowired` injected a bean, it adds cost without value.

**Do not share mutable state across tests.** Each test must construct its own `StepVerifier`. Do not reuse `Mono`/`Flux` instances across tests as they may carry subscription state.

**Do not skip `verifyComplete()` or `verify()`.** Every `StepVerifier` chain must terminate with `.verifyComplete()` (for success) or `.verify()` / `.expectError(...).verify()` (for errors). Omitting this means the reactive chain was never actually subscribed to.

**Do not ignore test isolation for Testcontainers.** Always truncate tables in `@AfterEach` for PostgreSQL integration tests. Use unique consumer group IDs for Kafka tests (`"test-group-" + UUID.randomUUID()`). Clear caches between test runs.
