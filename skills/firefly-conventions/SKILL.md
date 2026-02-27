---
name: firefly-conventions
description: Use when writing any code in a Firefly Framework project — applies naming conventions, package structure, reactive patterns, error handling, and build conventions that all fireflyframework modules follow
---

# Firefly Framework Conventions

## 1. Package Structure

Base package: `org.fireflyframework`. Every module follows `org.fireflyframework.{module}`.

| Module | Base Package | Example Sub-packages |
|---|---|---|
| kernel | `org.fireflyframework.kernel` | `exception` |
| web | `org.fireflyframework.web` | `error.handler`, `error.converter`, `error.exceptions`, `error.models`, `cors.config`, `idempotency`, `logging`, `openapi` |
| cqrs | `org.fireflyframework.cqrs` | `command`, `query`, `annotations`, `config`, `context`, `event`, `fluent`, `validation`, `authorization`, `cache`, `tracing` |
| starter-core | `org.fireflyframework.core` | `actuator.config`, `config`, `logging`, `web.client`, `web.resilience`, `messaging.config` |
| starter-domain | `org.fireflyframework.domain` | `config` |
| starter-application | `org.fireflyframework.common.application` | `config`, `controller`, `context`, `security`, `security.annotation`, `metadata`, `resolver`, `spi.dto`, `service`, `health` |
| starter-data | `org.fireflyframework.data` | `config`, `controller`, `model`, `service`, `enrichment`, `mapper`, `persistence`, `quality`, `transform` |
| utils | `org.fireflyframework.utils` | (PDF, templating utilities) |
| observability | `org.fireflyframework.observability` | `metrics` |
| client | `org.fireflyframework.client` | `exception` |
| cache | `org.fireflyframework.cache` | (unified caching) |
| validators | `org.fireflyframework.validators` | (common validators) |
| eda | `org.fireflyframework.eda` | (event-driven architecture) |
| webhooks | `org.fireflyframework.webhooks` | `core.mappers`, `core.domain`, `interfaces.dto` |
| callbacks | `org.fireflyframework.callbacks` | `core.mapper`, `interfaces` |
| rules | `org.fireflyframework.rules` | `core.mappers`, `interfaces.dtos`, `sdk.model` |

**Sub-package conventions within modules:**

| Sub-package | Purpose |
|---|---|
| `config` | Spring `@Configuration` and `@AutoConfiguration` classes, `*Properties` classes |
| `annotation` / `annotations` | Custom annotations (e.g., `@CommandHandlerComponent`, `@Secure`) |
| `exception` / `exceptions` | Exception classes specific to the module |
| `domain` | Domain entities and value objects |
| `port` | Port interfaces (hexagonal architecture) |
| `adapter` | Adapter implementations |
| `service` | Service layer classes |
| `health` | Spring Boot `HealthIndicator` implementations |
| `metrics` | Micrometer metrics support |
| `controller` | Reactive REST controllers |
| `dto` / `dtos` | Data Transfer Objects |
| `mapper` / `mappers` | MapStruct mapper classes |
| `model` / `models` | Response/request models |
| `converter` | Exception converters or type converters |

## 2. Reactive Rules

All Firefly modules are **fully reactive** using Spring WebFlux and Project Reactor.

- **Return types**: Always `Mono<T>` for single values or `Flux<T>` for streams. Never return plain objects from handlers or service methods.
- **No blocking**: Never call `.block()`, `.blockFirst()`, `.blockLast()`, or `Thread.sleep()` in production code. Use `Mono.defer()`, `Mono.fromCallable()`, or `flatMap()` chains.
- **WebFlux only**: The framework uses `spring-boot-starter-webflux`. Spring MVC (`spring-webmvc`, `spring-boot-starter-web`) is explicitly excluded in test configurations.
- **WebClient**: Use `WebClient` (not `RestTemplate`) for HTTP calls. The framework provides `ResilientWebClient` and `WebClientTemplate` in `org.fireflyframework.core.web`.
- **Testing**: Use `reactor-test` (`StepVerifier`) for reactive assertions. Use `WebTestClient` for controller tests.
- **Reactor context**: Use `Mono.deferContextual()` for propagating tenant/user context in reactive chains.

```java
// CORRECT: Reactive handler
@Override
protected Mono<AccountResult> doHandle(CreateAccountCommand command) {
    return accountService.create(command)
        .flatMap(this::publishEvent);
}

// WRONG: Blocking call
@Override
protected Mono<AccountResult> doHandle(CreateAccountCommand command) {
    Account account = accountService.create(command).block(); // NEVER DO THIS
    return Mono.just(new AccountResult(account));
}
```

## 3. Exception Hierarchy

The framework defines a three-level kernel exception hierarchy in `org.fireflyframework.kernel.exception`:

```
RuntimeException
  └── FireflyException                          # Base for ALL framework errors
        ├── errorCode: String
        ├── context: Map<String, Object>
        ├── FireflyInfrastructureException      # Database, cache, messaging, network failures
        └── FireflySecurityException            # Authentication & authorization errors
```

**Web-layer exceptions** in `org.fireflyframework.web.error.exceptions` extend `FireflyException` via `BusinessException`:

```
FireflyException
  └── BusinessException                         # Base web exception (status + code + metadata)
        ├── ResourceNotFoundException           # 404
        ├── ValidationException                 # 400 (with field-level errors)
        ├── InvalidRequestException             # 400
        ├── UnauthorizedException               # 401
        ├── ForbiddenException                  # 403
        ├── AuthorizationException              # 403
        ├── ConflictException                   # 409
        ├── GoneException                       # 410
        ├── PayloadTooLargeException            # 413
        ├── UnsupportedMediaTypeException       # 415
        ├── MethodNotAllowedException           # 405
        ├── PreconditionFailedException         # 412
        ├── LockedResourceException             # 423
        ├── RateLimitException                  # 429
        ├── QuotaExceededException              # 429
        ├── ServiceException                    # 500
        ├── DataIntegrityException              # 500 (constraint violations)
        ├── ConcurrencyException                # 409
        ├── OperationTimeoutException           # 504
        ├── NotImplementedException             # 501
        ├── BadGatewayException                 # 502
        ├── ServiceUnavailableException         # 503
        ├── GatewayTimeoutException             # 504
        ├── ThirdPartyServiceException          # 502
        ├── DegradedServiceException            # 503
        ├── CircuitBreakerException             # 503
        ├── BulkheadException                   # 503
        └── RetryExhaustedException             # 503
```

**Usage pattern** -- always throw framework exceptions, never raw ones:

```java
// CORRECT
throw ResourceNotFoundException.forResource("Account", accountId);
throw new BusinessException(HttpStatus.BAD_REQUEST, "INVALID_AMOUNT", "Amount must be positive");

// WRONG
throw new RuntimeException("Account not found");
throw new ResponseStatusException(HttpStatus.NOT_FOUND, "not found");
```

## 4. DTO and Mapping Conventions

### DTO naming

- Suffix all DTOs with `DTO`: `ContractInfoDTO`, `WebhookEventDTO`, `RuleDefinitionDTO`, `AuditTrailDTO`
- Place DTOs in `.dto` or `.dtos` or `.interfaces.dto` sub-packages
- Use Lombok `@Data @Builder @NoArgsConstructor @AllArgsConstructor` on every DTO
- OpenAPI-generated DTOs land in `${openapi.model.package}` which resolves to `org.fireflyframework.{artifact}.interfaces.dto`

### MapStruct mappers

- Use `@Mapper(componentModel = MappingConstants.ComponentModel.SPRING)` for Spring integration
- Prefer `abstract class` over `interface` when custom mapping methods are needed (allows `@Autowired` injection)
- Place mappers in `.mapper` or `.mappers` sub-packages
- Name: `{Entity}Mapper` -- e.g., `WebhookEventMapper`, `RuleDefinitionMapper`, `CallbackConfigurationMapper`
- Method naming: `toDomainEvent()`, `toDto()`, `toDomain()`, `toEntity()`

```java
@Mapper(componentModel = MappingConstants.ComponentModel.SPRING)
public abstract class WebhookEventMapper {
    @Autowired
    protected ObjectMapper objectMapper;

    public abstract WebhookReceivedEvent toDomainEvent(WebhookEventDTO dto);
    public abstract WebhookEventDTO toDto(WebhookReceivedEvent event);
}
```

### Lombok usage

- `@Data @Builder @NoArgsConstructor @AllArgsConstructor` on DTOs and model classes
- `@Getter` on exception classes (see `BusinessException`)
- `@Slf4j` on all classes that need logging -- never instantiate a logger manually
- `@RequiredArgsConstructor` on classes with `final` dependencies (replaces hand-written constructor)
- Lombok is `<scope>provided</scope>` or `<optional>true</optional>` -- never a transitive dependency
- Annotation processor order: Lombok -> MapStruct -> lombok-mapstruct-binding -> spring-boot-configuration-processor

```java
// CORRECT -- @Slf4j generates the `log` field at compile time
@Slf4j
@Service
public class PaymentService {
    public Mono<PaymentResult> process(PaymentRequest request) {
        log.info("Processing payment: paymentId={}", request.getPaymentId());
        // ...
    }
}

// WRONG -- manual logger declaration
@Service
public class PaymentService {
    private static final Logger log = LoggerFactory.getLogger(PaymentService.class);
}
```

## 5. POM Conventions

### Inheritance chain

```
fireflyframework-parent (groupId: org.fireflyframework, packaging: pom)
  └── Every module's <parent>
        fireflyframework-bom (standalone, imports all module versions)
```

All modules set `<parent>` to `fireflyframework-parent`:

```xml
<parent>
    <groupId>org.fireflyframework</groupId>
    <artifactId>fireflyframework-parent</artifactId>
    <version>26.02.06</version>
    <relativePath/>
</parent>
```

Consumer projects import the BOM in `<dependencyManagement>`:

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.fireflyframework</groupId>
            <artifactId>fireflyframework-bom</artifactId>
            <version>26.02.06</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

### Annotation processors (defined in parent)

The parent POM configures `maven-compiler-plugin` with these annotation processors in order:

1. `lombok` (v1.18.42)
2. `mapstruct-processor` (v1.6.3)
3. `lombok-mapstruct-binding` (v0.2.0)
4. `spring-boot-configuration-processor`

Compiler flags: `-parameters -proc:full`

### Key managed versions (from parent properties)

| Dependency | Version |
|---|---|
| Java | 25 (profile `java21` available) |
| Spring Boot | 3.5.10 |
| Spring Cloud | 2025.0.1 |
| MapStruct | 1.6.3 |
| Lombok | 1.18.42 |
| SpringDoc OpenAPI | 2.8.15 |
| PostgreSQL | 42.7.9 |
| Flyway | 11.20.3 |
| Testcontainers | 1.21.4 |
| Resilience4j | 2.3.0 |
| gRPC | 1.79.0 |
| OpenTelemetry | 1.59.0 |
| AWS SDK v2 | 2.41.24 |

## 6. Build and Versioning

### CalVer format

All modules use **CalVer**: `YY.MM.patch` -- e.g., `26.02.06` means year 2026, month 02, patch 06.

### JDK requirements

- Default: Java 25 (`<java.version>25</java.version>`)
- Minimum enforced: JDK 21+ (`maven-enforcer-plugin` requires `[21,)`)
- Profile `java21` switches to Java 21 compilation: `mvn -Pjava21`

### Maven profiles

| Profile | Purpose |
|---|---|
| `java21` | Compile with Java 21 compatibility |
| `release` | Attach sources + javadoc JARs |
| `maven-central` | GPG sign + publish to Maven Central via `central-publishing-maven-plugin` + flatten POM |

### Library JARs vs. Boot JARs

- Library modules (kernel, web, cqrs, etc.): Standard JAR (`<packaging>jar</packaging>`)
- Starter modules with `spring-boot-maven-plugin`: Use `<classifier>exec</classifier>` to produce both a library JAR and an executable JAR

## 7. Error Handling

### RFC 7807 compliance

The framework provides full RFC 7807 (Problem Details for HTTP APIs) support via:

- `org.fireflyframework.web.error.models.ProblemDetail` -- RFC 7807 model with `type`, `title`, `status`, `detail`, `instance`, and `extensions`
- `org.fireflyframework.web.error.models.ErrorResponse` -- Enhanced response model with tracing, categories, severity, retry info
- `ProblemDetail.fromErrorResponse(errorResponse)` converts between the two

### GlobalExceptionHandler

`org.fireflyframework.web.error.handler.GlobalExceptionHandler` implements `ErrorWebExceptionHandler` (order -2) and provides:

- Automatic exception-to-response mapping via `ExceptionConverterService`
- Distributed tracing (OpenTelemetry trace/span IDs injected into error responses)
- PII masking via `PiiMaskingService`
- Error categorization: `VALIDATION`, `BUSINESS`, `TECHNICAL`, `SECURITY`, `EXTERNAL`, `RESOURCE`, `RATE_LIMIT`, `CIRCUIT_BREAKER`
- Severity levels: `LOW`, `MEDIUM`, `HIGH`, `CRITICAL`
- Retryable detection + `Retry-After` header
- Content negotiation for JSON/Problem+JSON
- Security headers: `X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`, `X-XSS-Protection: 1; mode=block`
- Environment-aware: Stack traces and debug info only in non-production profiles

### Exception converters

Implement `ExceptionConverter<T extends Throwable>` and register as `@Component`. Built-in converters:

| Converter | Handles |
|---|---|
| `DataAccessExceptionConverter` | Spring `DataAccessException` (integrity, timeout, transient) |
| `R2dbcExceptionConverter` | R2DBC exceptions |
| `SecurityExceptionConverter` | Spring Security exceptions |
| `ValidationExceptionConverter` | Jakarta validation exceptions |
| `WebFluxExceptionConverter` | WebFlux-specific exceptions |
| `JsonExceptionConverter` | Jackson JSON processing errors |
| `NetworkExceptionConverter` | Network/connectivity errors |
| `Resilience4jExceptionConverter` | Circuit breaker, bulkhead, rate limit |
| `JpaExceptionConverter` | JPA/Hibernate exceptions |
| `OptimisticLockingFailureExceptionConverter` | Optimistic locking conflicts |
| `HttpClientErrorExceptionConverter` | HTTP 4xx from upstream services |
| `HttpServerErrorExceptionConverter` | HTTP 5xx from upstream services |
| `ExternalServiceExceptionConverter` | Third-party service failures |

### Creating custom converters

```java
@Component
public class MyCustomExceptionConverter implements ExceptionConverter<MyLibraryException> {
    @Override
    public Class<MyLibraryException> getExceptionType() { return MyLibraryException.class; }

    @Override
    public BusinessException convert(MyLibraryException ex) {
        return new ServiceException("MY_ERROR", ex.getMessage());
    }
}
```

## 8. Tier Model

The framework defines 5 tiers, each with a dedicated starter:

| Tier | Starter | Base Package | Purpose |
|---|---|---|---|
| Core | `fireflyframework-starter-core` | `org.fireflyframework.core` | Actuator, WebClient, logging, tracing, service registry, cloud config. Infrastructure layer shared by all tiers. |
| Domain | `fireflyframework-starter-domain` | `org.fireflyframework.domain` | DDD + reactive: CQRS (command/query bus), SAGA orchestration, service client, EDA, validators. For domain microservices. |
| Experience | `fireflyframework-starter-application` | `org.fireflyframework.common.application` | BFF per user journey (`exp-*`): aggregates domain SDKs into journey-specific APIs for frontends. Security, session management, config/context resolvers. Does NOT access core services directly. |
| Application | `fireflyframework-starter-application` | `org.fireflyframework.common.application` | Infrastructure edge services (`app-*`): authentication, gateways, backoffice. Not for user-journey BFFs — use Experience tier instead. |
| Data | `fireflyframework-starter-data` | `org.fireflyframework.data` | Data enrichment, ETL job orchestration, data quality, lineage tracking, transformation pipelines. For data microservices. |

**Dependency flow**: Core is the foundation. Domain, Experience, Application, and Data each include Core transitively. Domain includes CQRS, EDA, orchestration, client. Experience and Application include security, session, metadata. Data includes enrichment, quality, jobs. Experience services consume only domain SDKs; Application services may consume domain SDKs.

**Dependency direction**: `channel (web/mobile) → exp-* → domain-* → core-*` and `app-* → domain-* → core-*`. Never invert.

### Starter dependencies (from BOM)

```xml
<!-- Domain microservice -->
<dependency>
    <groupId>org.fireflyframework</groupId>
    <artifactId>fireflyframework-starter-domain</artifactId>
</dependency>

<!-- Experience (exp-*) or Application (app-*) microservice -->
<dependency>
    <groupId>org.fireflyframework</groupId>
    <artifactId>fireflyframework-starter-application</artifactId>
</dependency>

<!-- Data microservice -->
<dependency>
    <groupId>org.fireflyframework</groupId>
    <artifactId>fireflyframework-starter-data</artifactId>
</dependency>
```

## 9. Configuration

### application.yml conventions

```yaml
server:
  address: ${SERVER_ADDRESS:localhost}
  port: ${SERVER_PORT:8080}
  shutdown: graceful

firefly:
  cache:
    enabled: true
    default-cache-type: CAFFEINE
    metrics-enabled: true
    health-enabled: true
    caffeine:
      cache-name: application-layer
      key-prefix: "firefly:application"
      maximum-size: 1000
      expire-after-write: PT1H
      record-stats: true

management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,caches
  endpoint:
    health:
      show-details: when-authorized
```

### Profile naming

- `dev` -- local development
- `pre` -- pre-production / staging
- `pro` / `prod` / `production` -- production (the `GlobalExceptionHandler` checks for `prod` and `production` to suppress debug info)

### Key config prefixes

| Prefix | Module |
|---|---|
| `firefly.cache.*` | fireflyframework-cache |
| `firefly.cqrs.*` | fireflyframework-cqrs |
| `firefly.error.*` | fireflyframework-web error handling |
| `firefly.cors.*` | fireflyframework-web CORS |
| `firefly.idempotency.*` | fireflyframework-web idempotency |
| `firefly.pii-masking.*` | fireflyframework-web PII masking |
| `firefly.logging.*` | fireflyframework-starter-core |
| `firefly.web-client.*` | fireflyframework-starter-core |
| `firefly.service-registry.*` | fireflyframework-starter-core |
| `firefly.cloud-config.*` | fireflyframework-starter-core |
| `firefly.application.*` | fireflyframework-starter-application (used by both `exp-*` and `app-*` services) |
| `firefly.data.enrichment.*` | fireflyframework-starter-data |
| `firefly.data.jobs.*` | fireflyframework-starter-data |

### OpenAPI properties (from parent)

```xml
<openapi.base.package>${base.package}.${project.artifactId}</openapi.base.package>
<openapi.model.package>${openapi.base.package}.interfaces.dto</openapi.model.package>
<openapi.api.package>${openapi.base.package}.interfaces.api</openapi.api.package>
```

Generated DTO classes land in `org.fireflyframework.{artifactId}.interfaces.dto`.

## 10. CQRS Conventions

### Commands and Queries

- Commands implement `Command<R>` -- represent intent to change state, return `Mono<R>`
- Queries implement `Query<R>` -- represent read requests, return `Mono<R>`, support caching
- Both support built-in validation (`Mono<ValidationResult> validate()`) and authorization (`Mono<AuthorizationResult> authorize()`)

### Handlers

- Command handlers extend `CommandHandler<C, R>` -- implement `protected abstract Mono<R> doHandle(C command)`
- Query handlers extend `QueryHandler<Q, R>` -- implement `protected abstract Mono<R> doHandle(Q query)`
- Both auto-detect generic types -- no need to override `getCommandType()` or `getQueryType()`
- Both provide lifecycle hooks: `preProcess()`, `postProcess()`, `onSuccess()`, `onError()`, `mapError()`

### Annotations

| Annotation | Package | Purpose |
|---|---|---|
| `@CommandHandlerComponent` | `o.f.cqrs.annotations` | Marks a command handler as a Spring component with timeout, retries, metrics, tracing, validation config |
| `@QueryHandlerComponent` | `o.f.cqrs.annotations` | Marks a query handler with caching (TTL, key fields), metrics, tracing config |
| `@PublishDomainEvent` | `o.f.cqrs.event.annotation` | Auto-publishes command result to EDA destination after success |
| `@InvalidateCacheOn` | `o.f.cqrs.cache.annotation` | Auto-invalidates query cache when specified event types arrive via EDA |
| `@CustomAuthorization` | `o.f.cqrs.authorization.annotation` | Custom authorization rules for commands/queries |
| `@Secure` | `o.f.common.application.security.annotation` | Declarative endpoint security: roles, permissions, SpEL expressions |
| `@RequireContext` | `o.f.common.application.security.annotation` | Requires specific execution context values |
| `@FireflyApplication` | `o.f.common.application.metadata` | Microservice metadata: name, domain, team, owners, dependencies |
| `@EnableOpenApiGen` | `o.f.web.openapi` | Meta-annotation for OpenAPI spec generation in test sources |
| `@DisableIdempotency` | `o.f.web.idempotency.annotation` | Disables idempotency filter for specific endpoints |

## 11. Web-Layer Exception Availability

`NotImplementedException` and other exceptions from `org.fireflyframework.web.error.exceptions` belong to the `fireflyframework-web` module. If a `-core` module needs to throw these exceptions, it must explicitly declare the dependency:

```xml
<!-- In {service}-core/pom.xml -->
<dependency>
    <groupId>org.fireflyframework</groupId>
    <artifactId>fireflyframework-web</artifactId>
</dependency>
```

**Common mistake:** Using `NotImplementedException` in a `-core` module without adding `fireflyframework-web` as a dependency. The `-core` module typically depends on `fireflyframework-starter-core` or `fireflyframework-starter-domain`, neither of which transitively includes `fireflyframework-web`.

## 12. Orchestration Enums

The `StepStatus` enum in `org.fireflyframework.orchestration.core.model` has these values:

```
PENDING, RUNNING, DONE, FAILED, SKIPPED, TIMED_OUT, RETRYING
```

**Common mistake:** Using `StepStatus.COMPLETED` -- this value does not exist. The correct value is `StepStatus.DONE`.

## 13. Controller Conventions

### @Valid on Request Bodies

All `@RequestBody` parameters in controllers MUST have the `@Valid` annotation for Jakarta Bean Validation:

```java
// CORRECT
@PostMapping
public Mono<ResponseEntity<AccountDTO>> create(
        @Valid @RequestBody AccountDTO dto) { ... }

// WRONG -- missing @Valid
@PostMapping
public Mono<ResponseEntity<AccountDTO>> create(
        @RequestBody AccountDTO dto) { ... }
```

### scanBasePackages

When using `@SpringBootApplication(scanBasePackages = {...})`, the framework web package is `org.fireflyframework.web`, not `com.firefly.common.web`:

```java
// CORRECT
@SpringBootApplication(scanBasePackages = {
    "com.firefly.core.customer",
    "org.fireflyframework.web"
})

// WRONG
@SpringBootApplication(scanBasePackages = {
    "com.firefly.core.customer",
    "com.firefly.common.web"  // Package does not exist
})
```

### springdoc.packages-to-scan

The controllers sub-package is plural (`controllers`, not `controller`):

```yaml
# CORRECT
springdoc:
  packages-to-scan: com.firefly.core.customer.web.controllers

# WRONG
springdoc:
  packages-to-scan: com.firefly.core.customer.web.controller
```

## 14. Configuration Properties

### @ConfigurationProperties without @Configuration

When the application class has `@ConfigurationPropertiesScan`, properties classes only need `@ConfigurationProperties` -- do NOT add `@Configuration`:

```java
// CORRECT -- discovered by @ConfigurationPropertiesScan
@ConfigurationProperties(prefix = "api-configuration.customer-mgmt")
public class CustomerMgmtProperties {
    private String basePath;
    // getters and setters
}

// WRONG -- @Configuration is unnecessary and causes bean conflicts
@Configuration
@ConfigurationProperties(prefix = "api-configuration.customer-mgmt")
public class CustomerMgmtProperties { ... }
```

### Actuator Health Details

Health endpoint details should be `when-authorized` (not `always`) in all environments:

```yaml
management:
  endpoint:
    health:
      show-details: when-authorized  # NOT "always"
```

### Profile Names

Standard profiles are `dev`, `pre`, and `pro`. Never use `testing`, `staging`, or `local`:

```yaml
# CORRECT
spring.config.activate.on-profile: pre

# WRONG
spring.config.activate.on-profile: testing
```

## 15. Code Hygiene

### Extract hardcoded strings into named constants

Repeated string literals used as identifiers (step IDs, signal names, event types, query names, status values, configuration keys) must be declared as `static final String` constants. This applies to workflow/saga definitions, service implementations, controllers, and tests.

**Where to place constants:**
- **Workflow/Saga classes**: Own their step IDs, signal names, variable names, phase labels, and status values as `public static final` fields. These form the contract that service classes and tests reference.
- **Service classes**: Reference constants from the workflow/saga instead of declaring their own copies. Remove duplicate `private static final` fields.
- **Controllers**: Declare `private static final` fields for response map keys and response status strings specific to the REST API (e.g., `KEY_STATUS`, `STATUS_INITIATED`).
- **Tests**: Use `static import` of the constants from the owning class.

```java
// CORRECT -- constants in the workflow class (single source of truth)
@Workflow(id = IndividualOnboardingWorkflow.WORKFLOW_ID)
public class IndividualOnboardingWorkflow {

    public static final String WORKFLOW_ID = "individual-onboarding";
    public static final String STEP_REGISTER_PARTY = "register-party";
    public static final String SIGNAL_PERSONAL_DATA = "personal-data-submitted";
    public static final String PHASE_AWAITING_DOCS = "AWAITING_DOCUMENTS";

    @WorkflowStep(id = STEP_REGISTER_PARTY)
    public Mono<UUID> registerParty(@Input Command cmd) { ... }
}

// CORRECT -- service references the workflow's constants
public class OnboardingServiceImpl {
    return signalService.signal(cid, IndividualOnboardingWorkflow.SIGNAL_PERSONAL_DATA, command);
}

// CORRECT -- test uses static import
import static com.example.workflows.IndividualOnboardingWorkflow.*;
assertThat(status.getCurrentPhase()).isEqualTo(PHASE_AWAITING_DOCS);

// WRONG -- raw string literals scattered across files
@WorkflowStep(id = "register-party")
return signalService.signal(cid, "personal-data-submitted", command);
assertThat(status.getCurrentPhase()).isEqualTo("AWAITING_DOCUMENTS");
```

Java `static final String` fields initialized with string literals are compile-time constant expressions, so they can be used in annotation values (`@WorkflowStep(id = CONSTANT)`, `@WaitForSignal(CONSTANT)`, `@FromStep(CONSTANT)`).

### Remove dead code

Unused fields, methods, imports, and classes must be removed — not commented out, not annotated with `@Deprecated`, not left as "maybe useful later". Dead code increases cognitive load and hides real usage patterns.

**Systematic approach:**
1. After any refactoring or migration, grep for every field and method in the changed classes to verify they are still referenced.
2. Remove fields that are injected but never called (e.g., an SDK client declared in the constructor but never used in any method).
3. Remove command/DTO fields that are not read by any consumer (workflow, handler, or mapper).
4. Remove `@Bean` methods in factory classes when the produced bean is not injected anywhere.
5. Remove imports that become orphaned after field/method removal.

```java
// WRONG -- KybApi injected but never called
@Component
public class KycKybClientFactory {
    @Bean public KycApi kycApi() { return new KycApi(apiClient); }
    @Bean public KybApi kybApi() { return new KybApi(apiClient); } // unused
}

// CORRECT -- only beans that are actually consumed
@Component
public class KycKybClientFactory {
    @Bean public KycApi kycApi() { return new KycApi(apiClient); }
}
```

## 16. PII Logging Rules

Never log personally identifiable information (PII). Use resource identifiers instead:

```java
// WRONG -- logs PII
log.info("Processing customer: name={}, email={}", customer.getName(), customer.getEmail());
log.info("Payment for IBAN: {} amount: {}", iban, amount);

// CORRECT -- logs only resource identifiers
log.info("Processing customer: partyId={}", customer.getPartyId());
log.info("Payment initiated: paymentId={} consentId={}", paymentId, consentId);
```

PII includes: names, emails, phone numbers, addresses, IBANs, account numbers, SSNs, tax IDs, passport numbers, API keys, passwords, and card data.

## 17. Javadoc Documentation Requirements

All generated code must include Javadoc documentation:

### Classes and Interfaces

Every public class and interface requires a Javadoc comment explaining its purpose:

```java
/**
 * Service responsible for managing KYC verification lifecycle.
 * Orchestrates identity verification through external providers,
 * manages document collection, and records compliance decisions.
 */
@Service
@RequiredArgsConstructor
@Slf4j
public class KycServiceImpl implements KycService { ... }
```

### Public Methods

Every public method requires Javadoc with `@param`, `@return`, and `@throws` tags:

```java
/**
 * Initiates identity verification for the given case through the configured provider.
 *
 * @param caseId the compliance case identifier
 * @return a {@link SagaResult} indicating verification outcome
 * @throws ResourceNotFoundException if the case does not exist
 */
public Mono<SagaResult> verify(UUID caseId) { ... }
```

### When to Skip

- Private methods (unless complex logic warrants explanation)
- Simple getters/setters (Lombok-generated)
- Test methods (the test name should be self-explanatory)

## 18. README.md Documentation Standards

Every microservice MUST have a `README.md` at the project root with the following structure:

```markdown
# {Service Name}

{One-paragraph description of the service's purpose and role in the platform.}

## Table of Contents

- [Architecture](#architecture)
- [Module Structure](#module-structure)
- [API Reference](#api-reference)
- [Configuration](#configuration)
- [Getting Started](#getting-started)
- [Dependencies](#dependencies)

## Architecture

{Describe the service tier (Core/Domain/App), what business capability it provides,
and how it fits into the broader platform. Include a dependency diagram if it
orchestrates multiple services.}

## Module Structure

| Module | Purpose |
|--------|---------|
| `-interfaces` | DTOs, enums, API contracts |
| `-models` | R2DBC entities, repositories, Flyway migrations (core tier only) |
| `-infra` | SDK client factories, configuration properties (domain tier only) |
| `-core` | Service interfaces, implementations, mappers |
| `-web` | REST controllers, Spring Boot application, configuration |
| `-sdk` | Auto-generated OpenAPI client for consumers |

## API Reference

{List the main endpoints with HTTP method, path, and brief description.
Reference the Swagger UI URL for full documentation.}

## Configuration

{Document all required environment variables and configuration properties.
Include a table with variable name, description, and default value.}

## Getting Started

{Step-by-step instructions to build and run the service locally.
Include prerequisites, database setup, and Maven commands.}

## Dependencies

{List upstream services this service calls (via SDK) and downstream services
that consume this service's SDK.}
```

## 19. Cross-Layer Reflexive Property

When generating code for an upper-layer service (domain or app tier) that calls a lower-layer service (core tier), if the lower-layer method does not exist, you MUST create it. Never leave upper-layer methods returning static/mock data or empty results.

### Rule

If a domain service method calls `coreServiceApi.someMethod(...)` and that method does not exist in the core service's controller/service, then:

1. Create the endpoint in the core service's `-web` controller
2. Create the service interface method in the core service's `-core` module
3. Implement the service method with proper business logic
4. Rebuild the core service SDK so the domain service can call it

### Anti-Pattern

```java
// WRONG -- returning static data because the lower layer doesn't have the method
@Override
public Mono<SagaResult> verify(UUID caseId) {
    // TODO: call core service when endpoint is available
    return Mono.just(SagaResultHelper.success("verify"));
}
```

```java
// CORRECT -- calls the actual lower-layer service
@Override
public Mono<SagaResult> verify(UUID caseId) {
    return kycProviderPort.verifyIdentity(partyId, request)
        .flatMap(result -> kycVerificationApi.updateKycVerification(...))
        .map(updated -> SagaResultHelper.success("verify", "verification", updated.getId()));
}
```

### When the Lower Layer Doesn't Exist Yet

If you are building the upper layer first and the lower-layer endpoint does not exist:
1. Define a port interface in the upper layer's `-core` module
2. Create a stub adapter in the `-infra` module that throws `NotImplementedException`
3. Document the dependency clearly with a `// TODO: implement when {service} is available` comment
4. **Never** silently return empty or mock data from production service methods

## Quick Reference Table

| Topic | Do | Don't |
|---|---|---|
| Return types | `Mono<T>`, `Flux<T>` | Plain objects, `CompletableFuture` |
| Blocking | `flatMap()`, `map()`, `Mono.fromCallable()` | `.block()`, `.blockFirst()`, `Thread.sleep()` |
| HTTP client | `WebClient`, `ResilientWebClient` | `RestTemplate`, `HttpURLConnection` |
| Exceptions | `throw new ResourceNotFoundException(...)` | `throw new RuntimeException(...)` |
| Exception base | Extend `BusinessException` or `FireflyException` | Extend `RuntimeException` directly |
| DTOs | `@Data @Builder` + suffix `DTO` | Mutable POJOs, no suffix |
| Mappers | `@Mapper(componentModel = SPRING)` as abstract class | Manual mapping code, interface mappers with injections |
| Config classes | `@Configuration` + `*AutoConfiguration` suffix | `@Component` for config |
| Properties | `@ConfigurationProperties(prefix = "firefly.xxx")` | `@Value` for complex config |
| Profiles | `dev`, `pre`, `pro` | `local`, `staging`, `testing`, `production` |
| Versioning | CalVer `YY.MM.patch` (e.g., `26.02.06`) | SemVer |
| Parent POM | Inherit from `fireflyframework-parent` | Define your own plugin/dependency management |
| Dependency versions | Import `fireflyframework-bom` | Hardcode framework module versions |
| Java version | JDK 25 default, JDK 21 minimum | JDK < 21 |
| Web framework | Spring WebFlux | Spring MVC |
| CQRS handlers | Extend `CommandHandler<C,R>` / `QueryHandler<Q,R>` | Implement interfaces directly |
| Handler registration | `@CommandHandlerComponent` / `@QueryHandlerComponent` | Manual `CommandBus.register()` |
| Logging | `@Slf4j` (Lombok) with resource IDs only | `LoggerFactory.getLogger()`, logging PII (names, emails, IBANs) |
| String identifiers | `static final String` constants in the owning class | Raw string literals scattered across files |
| Dead code | Remove unused fields, methods, beans, imports after refactoring | Comment out, `@Deprecated` for "maybe later", unused `@Bean` methods |
| Cache keys | Prefix with `firefly:cache:{name}:` | Unprefixed keys |
| Error responses | RFC 7807 `ProblemDetail` / `ErrorResponse` | Custom error JSON shapes |
| Security | `@Secure(roles = {...})` | Inline `SecurityContextHolder` checks |
| Step status | `StepStatus.DONE` | `StepStatus.COMPLETED` (does not exist) |
| Controller validation | `@Valid @RequestBody` on all POST/PUT bodies | `@RequestBody` without `@Valid` |
| scanBasePackages | `org.fireflyframework.web` | `com.firefly.common.web` |
| Controller package | `web.controllers` (plural) | `web.controller` (singular) |
| Health details | `show-details: when-authorized` | `show-details: always` |
| Cross-layer methods | Implement full call chain to lower layers | Return static/mock data from service methods |
| Javadoc | Document all public classes and methods | Skip documentation entirely |
| README.md | Include in every microservice root | Leave services undocumented |
