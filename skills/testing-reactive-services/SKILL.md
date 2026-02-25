---
name: testing-reactive-services
description: Use when writing tests for Firefly Framework reactive services — covers StepVerifier patterns, Testcontainers setup for R2DBC/Kafka/Redis, WireMock for service mocking, WebTestClient for controller tests, and reactive context testing
---

# Testing Reactive Services

## Test Pyramid

Firefly services are fully reactive (Spring WebFlux, R2DBC, Project Reactor).

- **Unit tests (StepVerifier)** -- Service methods, command/query handlers, reactive operators. Mock with Mockito. Fast, no Spring context. Most tests live here.
- **Integration tests (Testcontainers)** -- Repository-to-database, Kafka/RabbitMQ publishing, event store. `@Testcontainers` + `@DynamicPropertySource`. Validates real infrastructure.
- **Controller tests (WebTestClient)** -- HTTP endpoints, serialization, error handling, headers. `MockServerWebExchange` for unit-level or `WebTestClient` for full integration.

Test business logic at the unit level; reserve integration tests for infrastructure boundaries.

## StepVerifier Patterns

All reactive return types must be verified with `StepVerifier`. Never call `.block()` in assertions.

```java
// Mono with complex assertions (preferred: assertNext + AssertJ)
StepVerifier.create(commandBus.send(command))
    .assertNext(result -> {
        assertThat(result.getAccountNumber()).startsWith("ACC-");
        assertThat(result.getCustomerId()).isEqualTo("CUST-123");
        assertThat(result.getStatus()).isEqualTo("ACTIVE");
    })
    .verifyComplete();

// Mono with inline predicate (expectNextMatches)
StepVerifier.create(queryBus.query(query))
    .expectNextMatches(result -> result.getBalance().equals(new BigDecimal("2500.00")))
    .verifyComplete();

// Error signals
StepVerifier.create(commandBus.send(forbiddenCommand))
    .expectError(AuthorizationException.class)
    .verify();

// Mono<Void> completion (filters, publishers)
StepVerifier.create(kafkaPublisher.publish(event, topic))
    .verifyComplete();

// Flux count and element assertions
StepVerifier.create(eventStore.streamAllEvents().take(Duration.ofSeconds(5)))
    .expectNextCount(2)
    .verifyComplete();

// Scalar values
StepVerifier.create(eventStore.aggregateExists(aggregateId, "Account"))
    .expectNext(true)
    .verifyComplete();
```

## Testcontainers Setup

### PostgreSQL R2DBC Container

```java
@Testcontainers
class PostgreSqlEventStoreIntegrationTest {

    @Container
    static final PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15-alpine")
            .withDatabaseName("firefly_test")
            .withUsername("firefly_test")
            .withPassword("test_password");

    @BeforeEach
    void setUp() {
        connectionFactory = new PostgresqlConnectionFactory(
            PostgresqlConnectionConfiguration.builder()
                .host(postgres.getHost())
                .port(postgres.getFirstMappedPort())
                .database(postgres.getDatabaseName())
                .username(postgres.getUsername())
                .password(postgres.getPassword())
                .build());
        databaseClient = DatabaseClient.create(connectionFactory);
        createSchema().block();
    }

    @AfterEach
    void tearDown() {
        databaseClient.sql("TRUNCATE TABLE events RESTART IDENTITY CASCADE")
            .fetch().rowsUpdated().block();
    }
}
```

### Kafka and RabbitMQ with @ServiceConnection

```java
@TestConfiguration(proxyBeanMethods = false)
public class TestContainersConfiguration {

    @Bean
    @ServiceConnection
    public KafkaContainer kafkaContainer() {
        return new KafkaContainer(DockerImageName.parse("confluentinc/cp-kafka:7.5.0"))
            .withReuse(true);
    }

    @Bean
    @ServiceConnection
    public RabbitMQContainer rabbitMQContainer() {
        return new RabbitMQContainer(DockerImageName.parse("rabbitmq:3.12-management-alpine"))
            .withReuse(true);
    }
}
```

### @DynamicPropertySource for Firefly-specific properties

```java
@DynamicPropertySource
static void kafkaProperties(DynamicPropertyRegistry registry) {
    registry.add("firefly.eda.publishers.kafka.default.enabled", () -> "true");
    registry.add("firefly.eda.publishers.kafka.default.bootstrap-servers", kafka::getBootstrapServers);
    registry.add("firefly.eda.publishers.kafka.default.default-topic", () -> "test-events");
}
```

Enable reusable containers in `src/test/resources/testcontainers.properties`:

```properties
testcontainers.reuse.enable=true
```

## Repository Tests

### H2 in-memory R2DBC for fast tests

```java
connectionFactory = new H2ConnectionFactory(
    H2ConnectionConfiguration.builder()
        .url("mem:testdb;DB_CLOSE_DELAY=-1").username("sa").build());
databaseClient = DatabaseClient.create(connectionFactory);
entityTemplate = new R2dbcEntityTemplate(connectionFactory);
transactionManager = new R2dbcTransactionManager(connectionFactory);
```

### Schema initialization

Create tables in `@BeforeEach` or place SQL in `src/test/resources/db/test-schema.sql`:

```java
private Mono<Void> createSchema() {
    return databaseClient.sql("""
        CREATE TABLE IF NOT EXISTS events (
            event_id UUID PRIMARY KEY,
            aggregate_id UUID NOT NULL,
            aggregate_type VARCHAR(255) NOT NULL,
            aggregate_version BIGINT NOT NULL,
            event_type VARCHAR(255) NOT NULL,
            event_data TEXT NOT NULL,
            CONSTRAINT unique_aggregate_version UNIQUE(aggregate_id, aggregate_version))
        """).fetch().rowsUpdated().then();
}
```

### Mocked R2dbcEntityTemplate for filter tests

```java
@ExtendWith(MockitoExtension.class)
class FilterUtilsTest {
    @Mock private R2dbcEntityTemplate entityTemplate;
    @Mock private ReactiveSelect<TestEntity> reactiveSelect;
    @Mock private TerminatingSelect<TestEntity> terminatingSelect;

    @BeforeEach
    void setUp() {
        when(entityTemplate.select(TestEntity.class)).thenReturn(reactiveSelect);
        when(reactiveSelect.matching(any(Query.class))).thenReturn(terminatingSelect);
    }

    @Test
    void testStringFiltering() {
        when(terminatingSelect.all()).thenReturn(Flux.just(entity));
        when(terminatingSelect.count()).thenReturn(Mono.just(1L));

        StepVerifier.create(filter.filter(request))
            .assertNext(response -> {
                assertThat(response.getContent()).hasSize(1);
                assertThat(response.getContent().get(0).getName()).isEqualTo("test name");
            })
            .verifyComplete();

        ArgumentCaptor<Query> queryCaptor = ArgumentCaptor.forClass(Query.class);
        verify(reactiveSelect, atLeastOnce()).matching(queryCaptor.capture());
        assertThat(queryCaptor.getValue().getCriteria().orElse(Criteria.empty()).toString())
            .contains("name LIKE '%test%'");
    }
}
```

## Service Tests

### CQRS command/query handler testing

Register handlers manually, mock infrastructure, verify with StepVerifier:

```java
@BeforeEach
void setUp() {
    ApplicationContext ctx = mock(ApplicationContext.class);
    CommandHandlerRegistry registry = new CommandHandlerRegistry(ctx);
    CommandValidationService validation = new CommandValidationService(new AutoValidationProcessor(null));
    CommandMetricsService metrics = new CommandMetricsService(new SimpleMeterRegistry());

    commandBus = new DefaultCommandBus(registry, validation,
        new AuthorizationService(TestAuthorizationProperties.createDefault(), Optional.empty()),
        metrics, new CorrelationContext());
    ((DefaultCommandBus) commandBus).registerHandler(new CreateAccountHandler());
}

@Test
void testCreateAccountCommand() {
    StepVerifier.create(commandBus.send(new CreateAccountCommand("CUST-123", "SAVINGS", new BigDecimal("1000"))))
        .expectNextMatches(r -> ((AccountCreatedResult) r).getAccountNumber().startsWith("ACC-"))
        .verifyComplete();
}
```

### Mocking reactive dependencies with Mockito

Return `Mono.just()`, `Mono.empty()`, or `Mono.error()` from mocks. Use `thenAnswer` for stateful behavior:

```java
@ExtendWith(MockitoExtension.class)
class ResiliencePatternTest {
    @Mock private EventPublisher delegatePublisher;

    @Test
    void shouldRetryOnFailureAndSucceedEventually() {
        AtomicInteger attempts = new AtomicInteger(0);
        when(delegatePublisher.publish(eq(event), eq("topic"), eq(headers)))
            .thenAnswer(inv -> attempts.incrementAndGet() < 3
                ? Mono.error(new RuntimeException("Temporary failure"))
                : Mono.empty());

        StepVerifier.create(Mono.fromSupplier(() -> delegatePublisher.publish(event, "topic", headers))
            .flatMap(m -> m).transformDeferred(RetryOperator.of(retry)))
            .verifyComplete();
        verify(delegatePublisher, times(3)).publish(event, "topic", headers);
    }
}
```

## Controller Tests

### MockServerWebExchange (unit-level, no Spring context)

```java
@Test
void handleBusinessException() {
    var exception = new BusinessException(HttpStatus.BAD_REQUEST, "ERROR_CODE", "Error message");
    var exchange = MockServerWebExchange.from(MockServerHttpRequest.get("/api/test").build());

    StepVerifier.create(exceptionHandler.handle(exchange, exception)).verifyComplete();
    assertEquals(HttpStatus.BAD_REQUEST, exchange.getResponse().getStatusCode());
    assertEquals(MediaType.APPLICATION_JSON, exchange.getResponse().getHeaders().getContentType());
}

@Test
void shouldAllowFirstRequestWithIdempotencyKey() {
    var request = MockServerHttpRequest.post("/test")
        .header("X-Idempotency-Key", "key-1").contentType(MediaType.APPLICATION_JSON).build();
    var exchange = MockServerWebExchange.from(request);
    var filterChain = mock(WebFilterChain.class);
    when(filterChain.filter(any())).thenReturn(Mono.empty());

    StepVerifier.create(filter.filter(exchange, filterChain)).verifyComplete();
    verify(filterChain, times(1)).filter(any());
}
```

### WebTestClient (full integration)

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@ActiveProfiles("test")
class AccountControllerIntegrationTest {
    @Autowired private WebTestClient webTestClient;

    @Test
    void shouldCreateAccount() {
        webTestClient.post().uri("/api/accounts")
            .contentType(MediaType.APPLICATION_JSON)
            .bodyValue(new CreateAccountRequest("CUST-123", "SAVINGS", new BigDecimal("1000")))
            .exchange()
            .expectStatus().isCreated()
            .expectBody()
            .jsonPath("$.accountNumber").isNotEmpty()
            .jsonPath("$.customerId").isEqualTo("CUST-123");
    }

    @Test
    void shouldReturnNotFound() {
        webTestClient.get().uri("/api/accounts/{id}", UUID.randomUUID())
            .exchange()
            .expectStatus().isNotFound()
            .expectBody()
            .jsonPath("$.code").isEqualTo("RESOURCE_NOT_FOUND");
    }
}
```

## WireMock

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@ActiveProfiles("test")
class ExternalServiceIntegrationTest {
    private static WireMockServer wireMockServer;

    @DynamicPropertySource
    static void props(DynamicPropertyRegistry registry) {
        registry.add("external-service.base-url", () -> "http://localhost:" + wireMockServer.port());
    }

    @BeforeAll
    static void startWireMock() {
        wireMockServer = new WireMockServer(WireMockConfiguration.wireMockConfig().dynamicPort());
        wireMockServer.start();
    }

    @AfterAll static void stopWireMock() { wireMockServer.stop(); }
    @BeforeEach void resetStubs() { wireMockServer.resetAll(); }

    @Test
    void shouldFetchCustomer() {
        wireMockServer.stubFor(get(urlEqualTo("/api/customers/123"))
            .willReturn(aResponse().withStatus(200)
                .withHeader("Content-Type", "application/json")
                .withBody("{\"id\":\"123\",\"name\":\"John Doe\"}")));

        StepVerifier.create(customerClient.getCustomer("123"))
            .assertNext(c -> assertThat(c.getName()).isEqualTo("John Doe"))
            .verifyComplete();
        wireMockServer.verify(getRequestedFor(urlEqualTo("/api/customers/123")));
    }

    @Test
    void shouldHandleTimeout() {
        wireMockServer.stubFor(get(urlEqualTo("/api/customers/123"))
            .willReturn(aResponse().withStatus(200).withFixedDelay(5000)));
        StepVerifier.create(customerClient.getCustomer("123"))
            .expectError(OperationTimeoutException.class).verify(Duration.ofSeconds(10));
    }

    @Test
    void shouldHandle5xx() {
        wireMockServer.stubFor(get(urlEqualTo("/api/customers/123"))
            .willReturn(aResponse().withStatus(503)));
        StepVerifier.create(customerClient.getCustomer("123"))
            .expectError(ThirdPartyServiceException.class).verify();
    }
}
```

## Reactive Context

### ExecutionContext (tenant, user, feature flags)

The framework propagates `ExecutionContext` through the CQRS pipeline. Tests construct context explicitly:

```java
@Test
void testCommandWithExecutionContext() {
    ExecutionContext context = ExecutionContext.builder()
        .withUserId("user-456").withTenantId("premium-tenant")
        .withSource("mobile-app")
        .withFeatureFlag("premium-features", true)
        .withFeatureFlag("auto-approve", true)
        .withProperty("priority", "HIGH").build();

    StepVerifier.create(commandBus.send(command, context))
        .assertNext(r -> {
            assertThat(r.getTenantId()).isEqualTo("premium-tenant");
            assertThat(r.getStatus()).isEqualTo("ACTIVE");
        }).verifyComplete();
}

@Test
void testMissingRequiredContext() {
    ExecutionContext context = ExecutionContext.builder().withUserId("user-456").build(); // no tenantId
    StepVerifier.create(handler.doHandle(command, context))
        .expectError(IllegalArgumentException.class).verify();
}

@Test
void testFeatureFlagBehavior() {
    ExecutionContext context = ExecutionContext.builder()
        .withUserId("user-456").withTenantId("basic-tenant")
        .withFeatureFlag("premium-features", false).build();
    StepVerifier.create(handler.doHandle(command, context))
        .assertNext(r -> assertThat(r.getStatus()).isEqualTo("PENDING_APPROVAL"))
        .verifyComplete();
}
```

### Authorization testing

```java
@Test
void shouldFailWhenAuthorizationFails() {
    StepVerifier.create(commandBus.send(new BankTransferCommand("ACC-FORBIDDEN", "ACC-456", amount)))
        .expectError(AuthorizationException.class).verify();
}

@Test
void shouldFailWithForbiddenUser() {
    ExecutionContext ctx = ExecutionContext.builder()
        .withUserId("USER-FORBIDDEN").withTenantId("TENANT-456").build();
    StepVerifier.create(commandBus.send(command, ctx))
        .expectError(AuthorizationException.class).verify();
}
```

## Test Configuration

### Test application

```java
@SpringBootConfiguration
@EnableAutoConfiguration
public class TestApplication { }
```

### Base integration test class

```java
@SpringBootTest(classes = TestApplication.class)
@ActiveProfiles("test")
public abstract class BaseIntegrationTest {
    protected static final Duration TEST_TIMEOUT = Duration.ofSeconds(30);

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("firefly.eda.enabled", () -> "true");
        registry.add("spring.main.allow-bean-definition-overriding", () -> "true");
    }
}
```

### application-test.yml

```yaml
spring:
  main:
    banner-mode: off
  r2dbc:
    pool: { enabled: true, initial-size: 2, max-size: 5 }
  flyway:
    enabled: false
firefly:
  cqrs: { enabled: true }
  eda: { enabled: true }
logging:
  level:
    org.fireflyframework: DEBUG
    org.springframework: WARN
    io.r2dbc: DEBUG
    org.testcontainers: INFO
```

### Test data builders

Use static factory methods and builders in a shared `testconfig` package. Pattern: `TestEventModels.SimpleTestEvent.create("message")` using `UUID.randomUUID()` and `Instant.now()` for IDs and timestamps. Provide a `builder()` method for events with many optional fields (tenantId, premium, etc.).

## Anti-patterns

**Do not call `.block()` in test assertions.** Use `StepVerifier`. `.block()` is only acceptable in `@BeforeEach`/`@AfterEach` for setup/teardown (schema creation, data cleanup).

**Do not use `Thread.sleep()`.** Use Awaitility for async side effects:

```java
await().atMost(Duration.ofSeconds(10)).pollInterval(Duration.ofMillis(100))
    .untilAsserted(() -> assertThat(consumedMessages).isNotEmpty());
```

**Do not over-mock the reactive chain.** Mock at boundaries (repository, external client), not in the middle of `flatMap` chains. If mocking gets complex, test at a higher level.

**Do not test framework plumbing.** Test business behavior, not that `@Autowired` works.

**Do not share mutable state across tests.** Construct a fresh `StepVerifier` per test. Do not reuse `Mono`/`Flux` instances.

**Do not skip `verifyComplete()` or `verify()`.** Every `StepVerifier` must terminate. Without it, the reactive chain is never subscribed.

**Do not ignore test isolation.** Truncate tables in `@AfterEach`. Use unique Kafka consumer group IDs (`"test-group-" + UUID.randomUUID()`). Clear caches between runs.
