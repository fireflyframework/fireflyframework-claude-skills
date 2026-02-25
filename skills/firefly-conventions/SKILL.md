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
- `@Slf4j` on all service classes, handlers, and configuration classes
- Lombok is `<scope>provided</scope>` or `<optional>true</optional>` -- never a transitive dependency
- Annotation processor order: Lombok -> MapStruct -> lombok-mapstruct-binding -> spring-boot-configuration-processor

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

The framework defines 4 tiers, each with a dedicated starter:

| Tier | Starter | Base Package | Purpose |
|---|---|---|---|
| Core | `fireflyframework-starter-core` | `org.fireflyframework.core` | Actuator, WebClient, logging, tracing, service registry, cloud config. Infrastructure layer shared by all tiers. |
| Domain | `fireflyframework-starter-domain` | `org.fireflyframework.domain` | DDD + reactive: CQRS (command/query bus), SAGA orchestration, service client, EDA, validators. For domain microservices. |
| Application | `fireflyframework-starter-application` | `org.fireflyframework.common.application` | API gateway/BFF layer: security (`@Secure`), `@FireflyApplication` metadata, session management, config/context resolvers, abstract controllers. |
| Data | `fireflyframework-starter-data` | `org.fireflyframework.data` | Data enrichment, ETL job orchestration, data quality, lineage tracking, transformation pipelines. For data microservices. |

**Dependency flow**: Core is the foundation. Domain, Application, and Data each include Core transitively. Domain includes CQRS, EDA, orchestration, client. Application includes security, session, metadata. Data includes enrichment, quality, jobs.

### Starter dependencies (from BOM)

```xml
<!-- Domain microservice -->
<dependency>
    <groupId>org.fireflyframework</groupId>
    <artifactId>fireflyframework-starter-domain</artifactId>
</dependency>

<!-- Application / BFF microservice -->
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
| `firefly.application.*` | fireflyframework-starter-application |
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
| Profiles | `dev`, `pre`, `pro` | `local`, `staging`, `production` (unless also needed for compatibility) |
| Versioning | CalVer `YY.MM.patch` (e.g., `26.02.06`) | SemVer |
| Parent POM | Inherit from `fireflyframework-parent` | Define your own plugin/dependency management |
| Dependency versions | Import `fireflyframework-bom` | Hardcode framework module versions |
| Java version | JDK 25 default, JDK 21 minimum | JDK < 21 |
| Web framework | Spring WebFlux | Spring MVC |
| CQRS handlers | Extend `CommandHandler<C,R>` / `QueryHandler<Q,R>` | Implement interfaces directly |
| Handler registration | `@CommandHandlerComponent` / `@QueryHandlerComponent` | Manual `CommandBus.register()` |
| Logging | `@Slf4j` (Lombok) | `Logger.getLogger()` |
| Cache keys | Prefix with `firefly:cache:{name}:` | Unprefixed keys |
| Error responses | RFC 7807 `ProblemDetail` / `ErrorResponse` | Custom error JSON shapes |
| Security | `@Secure(roles = {...})` | Inline `SecurityContextHolder` checks |
