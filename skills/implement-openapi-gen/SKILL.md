---
name: implement-openapi-gen
description: >
  Use when generating or configuring OpenAPI specs or SDK generation in a Firefly Framework
  service. Covers springdoc configuration, @EnableOpenApiGen, the openapi-generator-maven-plugin
  for SDK generation, AutoMockMissingBeansConfig, and generated package conventions.
---

# OpenAPI Spec Generation and SDK Generation in Firefly Framework

This skill covers the full pipeline: annotating controllers with Swagger/OpenAPI annotations,
generating the OpenAPI spec at build time via springdoc, and producing a Java SDK from that spec
via the `openapi-generator-maven-plugin`.

---

## 1. Architecture Overview

Firefly uses a **code-first** approach. Controllers annotated with `@Tag`, `@Operation`, and
`@ApiResponse` are the single source of truth. The OpenAPI spec is **never hand-written** -- it
is derived automatically from controller metadata during a Maven build.

### Build Flow (Reactor Order)

```
mvn clean install

[1] -interfaces  -> compile -> install        (DTOs, enums)
[2] -models      -> compile -> install        (JPA/R2DBC entities)
[3] -core        -> compile -> install        (services, mappers)
[4] -web         -> compile -> test ->
                    pre-integration-test  (spring-boot:start with OpenApiGenApplication)
                    integration-test      (springdoc-openapi:generate -> target/openapi/openapi.yml)
                    post-integration-test (spring-boot:stop)
                    -> install
[5] -sdk         -> openapi-generator:generate (reads -web/target/openapi/openapi.yml)
                    -> compile -> install
```

A single `mvn install` generates the OpenAPI spec and the SDK with zero manual intervention.

---

## 2. Framework Classes (from fireflyframework-web)

### 2.1 @EnableOpenApiGen

**File:** `org.fireflyframework.web.openapi.EnableOpenApiGen`

A meta-annotation that configures a minimal Spring Boot application for OpenAPI spec generation.
It combines `@SpringBootConfiguration`, `@EnableWebFlux`, and all required auto-configurations
for Springdoc and WebFlux into a single annotation.

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Inherited
@SpringBootConfiguration
@EnableWebFlux
@ImportAutoConfiguration({
    AutoMockMissingBeansConfig.class,
    SpringDocConfiguration.class,
    SpringDocConfigProperties.class,
    SpringDocWebFluxConfiguration.class,
    ReactiveWebServerFactoryAutoConfiguration.class,
    HttpHandlerAutoConfiguration.class,
    WebFluxAutoConfiguration.class,
    JacksonAutoConfiguration.class,
    SpringApplicationAdminJmxAutoConfiguration.class
})
public @interface EnableOpenApiGen {
}
```

Key points:
- Lives in `fireflyframework-web` (shared across all 60+ microservices)
- Replaces the need for `@SpringBootApplication` with its heavy auto-configuration
- Imports `AutoMockMissingBeansConfig` so controller dependencies are automatically mocked
- Imports Springdoc classes directly so the spec endpoint (`/v3/api-docs`) is available

### 2.2 AutoMockMissingBeansConfig

**File:** `org.fireflyframework.web.openapi.AutoMockMissingBeansConfig`

A `BeanDefinitionRegistryPostProcessor` activated only under the `openapi-gen` Spring profile.
It scans all `@RestController` beans, finds their unsatisfied dependencies (both `@Autowired`
field injection and constructor injection), and registers Mockito mocks for each missing bean.

```java
@Configuration
@Profile("openapi-gen")
public class AutoMockMissingBeansConfig implements BeanDefinitionRegistryPostProcessor {
    // Scans @RestController beans for missing dependencies
    // Registers Mockito.mock() singletons for each missing type
    // Supports both @Autowired field injection and constructor injection
}
```

This class is what makes it possible to start the application for spec generation without
requiring PostgreSQL, Kafka, Redis, or any other infrastructure.

### 2.3 OpenAPIAutoConfiguration

**File:** `org.fireflyframework.web.openapi.OpenAPIAutoConfiguration`

Auto-configured bean that creates the `OpenAPI` object with metadata from application properties.
Registered in `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`.

```java
@AutoConfiguration
public class OpenAPIAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean
    public OpenAPI customOpenAPI() {
        // Populates info (title, version, description, contact, license)
        // Adds server URLs based on active Spring profile (local, dev, staging, prod)
    }
}
```

Properties it reads:

| Property | Default | Description |
|----------|---------|-------------|
| `spring.application.name` | `Service` | Used as API title |
| `spring.application.version` | `1.0.0` | API version |
| `spring.application.description` | Derived from name | API description |
| `spring.application.team.name` | `Development Team` | Contact name |
| `spring.application.team.email` | `developers@getfirefly.io` | Contact email |
| `spring.application.team.url` | `https://getfirefly.io` | Contact URL |
| `spring.application.license.name` | `Apache 2.0` | License name |
| `spring.application.license.url` | Apache URL | License URL |
| `openapi.servers.enabled` | `true` | Whether to add server entries |

### 2.4 Idempotency OpenAPI Customizers

When `fireflyframework-cache` and `springdoc` are on the classpath, two customizers are
auto-configured by `IdempotencyOpenAPIAutoConfiguration`:

- **`IdempotencyOpenAPICustomizer`** -- adds the `X-Idempotency-Key` header parameter to all
  operations in the spec (implements `OpenApiCustomizer`)
- **`IdempotencyOperationCustomizer`** -- marks operations annotated with `@DisableIdempotency`
  via the `x-disable-idempotency` extension so the header is excluded for them (implements
  `OperationCustomizer`)

### 2.5 application-openapi-gen.yaml

**File:** `fireflyframework-web/src/main/resources/application-openapi-gen.yaml`

This Spring profile configuration is inherited by all services that depend on `fireflyframework-web`:

```yaml
spring:
  main:
    allow-bean-definition-overriding: true
springdoc:
  api-docs:
    enabled: true
    version: openapi_3_0
```

---

## 3. Step-by-Step: Setting Up a New Service

### Step 1: Add Dependencies to the -web Module POM

```xml
<!-- In the -web module pom.xml -->
<dependencies>
    <!-- Firefly Web (brings in springdoc, OpenAPI auto-config, AutoMock) -->
    <dependency>
        <groupId>org.fireflyframework</groupId>
        <artifactId>fireflyframework-web</artifactId>
    </dependency>

    <!-- OpenAPI / Springdoc (also brought transitively, but explicit is clearer) -->
    <dependency>
        <groupId>org.springdoc</groupId>
        <artifactId>springdoc-openapi-starter-webflux-ui</artifactId>
    </dependency>
</dependencies>
```

### Step 2: Set Properties in the -web Module POM

```xml
<properties>
    <!-- Enable OpenAPI generation (profile inherited from firefly-parent) -->
    <openapi.gen.skip>false</openapi.gen.skip>
    <openapi.gen.mainClass>com.firefly.your.service.web.openapi.OpenApiGenApplication</openapi.gen.mainClass>
</properties>
```

### Step 3: Create OpenApiGenApplication in src/test/java

This class lives under `src/test/java` and is **never packaged** into the production artifact.
It is activated by the build profile with `useTestClasspath=true`.

```java
package com.firefly.your.service.web.openapi;

import org.fireflyframework.web.openapi.EnableOpenApiGen;
import org.springframework.boot.SpringApplication;
import org.springframework.context.annotation.ComponentScan;

@EnableOpenApiGen
@ComponentScan(basePackages = "com.firefly.your.service.web.controllers")
public class OpenApiGenApplication {
    public static void main(String[] args) {
        SpringApplication.run(OpenApiGenApplication.class, args);
    }
}
```

Key conventions:
- Package: `<service-base-package>.web.openapi`
- `@ComponentScan` MUST point only at the controllers package (not services, repos, etc.)
- No `@SpringBootApplication` -- use `@EnableOpenApiGen` instead
- No infrastructure exclusions needed -- `AutoMockMissingBeansConfig` handles everything

### Step 4: Annotate Controllers with OpenAPI Metadata

Use Swagger v3 annotations from `io.swagger.v3.oas.annotations`. Every controller needs `@Tag`
and every method needs `@Operation`:

```java
@Tag(name = "Accounts", description = "APIs for managing bank accounts within the system")
@RestController
@RequestMapping("/api/v1/accounts")
public class AccountController {

    @Operation(summary = "Create Account",
               description = "Create a new bank account with the specified details.")
    @PostMapping(consumes = MediaType.APPLICATION_JSON_VALUE, produces = MediaType.APPLICATION_JSON_VALUE)
    public Mono<ResponseEntity<AccountDTO>> createAccount(
            @Parameter(description = "Data for the new account", required = true,
                    schema = @Schema(implementation = AccountDTO.class))
            @RequestBody AccountDTO accountDTO) { ... }

    @Operation(summary = "Get Account by ID",
               description = "Retrieve an existing bank account by its unique identifier.")
    @GetMapping(value = "/{accountId}", produces = MediaType.APPLICATION_JSON_VALUE)
    public Mono<ResponseEntity<AccountDTO>> getAccount(
            @Parameter(description = "Unique identifier of the account", required = true)
            @PathVariable("accountId") UUID accountId) { ... }
}
```

Add `@ApiResponses`/`@ApiResponse` with `@Content`/`@Schema` for detailed response docs.
See the Annotation Reference in Section 5 below.

### Step 5: Configure Application Properties for OpenAPI Metadata

In your service's `application.yml`:

```yaml
spring:
  application:
    name: core-banking-accounts
    version: 1.0.0
    description: Banking Accounts Core Application
    team:
      name: Firefly Software Solutions Inc
      email: dev@getfirefly.io
      url: https://getfirefly.io
```

### Step 6: Build to Generate the Spec

```bash
# Full build -- generates spec and SDK
mvn clean install

# Skip spec generation for quick rebuilds
mvn install -Dopenapi.gen.skip=true
```

The generated spec file will be at:
```
<service>-web/target/openapi/openapi.yml
```

---

## 4. SDK Generation with openapi-generator-maven-plugin

### 4.1 Create the -sdk Module

Every service that needs an SDK gets a `<service>-sdk` Maven module. Add it to the parent POM
`<modules>` list:

```xml
<modules>
    <module>core-banking-accounts-interfaces</module>
    <module>core-banking-accounts-models</module>
    <module>core-banking-accounts-core</module>
    <module>core-banking-accounts-web</module>
    <module>core-banking-accounts-sdk</module>
</modules>
```

### 4.2 SDK Module POM -- openapi-generator-maven-plugin Configuration

The SDK POM declares dependencies on `swagger-annotations`, Jackson, `jakarta.annotation-api`,
`javax.annotation-api`, `jackson-databind-nullable`, and OkHttp. The critical section is the
plugin configuration:

```xml
<properties>
    <openapi-generator.version>7.0.1</openapi-generator.version>
    <generated-sources-dir>${project.build.directory}/generated-sources</generated-sources-dir>
</properties>

<dependencies>
    <!-- Web module (provided scope -- only for reactor ordering) -->
    <dependency>
        <groupId>com.firefly</groupId>
        <artifactId>core-banking-accounts-web</artifactId>
        <scope>provided</scope>
    </dependency>
    <!-- Plus: swagger-annotations, jackson-databind, jackson-datatype-jsr310,
         jackson-databind-nullable, jackson-dataformat-yaml, jakarta.annotation-api,
         javax.annotation-api, okhttp, logging-interceptor -->
</dependencies>

<build>
    <plugins>
        <plugin>
            <groupId>org.openapitools</groupId>
            <artifactId>openapi-generator-maven-plugin</artifactId>
            <version>${openapi-generator.version}</version>
            <executions>
                <execution>
                    <id>core-banking-accounts-sdk</id>
                    <goals><goal>generate</goal></goals>
                    <configuration>
                        <inputSpec>${project.parent.basedir}/${project.parent.artifactId}-web/target/openapi/openapi.yml</inputSpec>
                        <generatorName>java</generatorName>
                        <library>webclient</library>
                        <output>${generated-sources-dir}</output>
                        <apiPackage>com.firefly.core.banking.accounts.sdk.api</apiPackage>
                        <modelPackage>com.firefly.core.banking.accounts.sdk.model</modelPackage>
                        <invokerPackage>com.firefly.core.banking.accounts.sdk.invoker</invokerPackage>
                        <configOptions>
                            <java20>true</java20>
                            <useTags>true</useTags>
                            <dateLibrary>java8-localdatetime</dateLibrary>
                            <sourceFolder>src/gen/java/main</sourceFolder>
                            <library>webclient</library>
                            <reactive>true</reactive>
                            <returnResponse>true</returnResponse>
                        </configOptions>
                    </configuration>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

### 4.3 Key Plugin Configuration Explained

| Setting | Value | Why |
|---------|-------|-----|
| `inputSpec` | `...-web/target/openapi/openapi.yml` | Reads the spec generated during the -web module build |
| `generatorName` | `java` | Generates a Java client SDK |
| `library` | `webclient` | Uses Spring WebClient (reactive HTTP client) |
| `reactive` | `true` | Returns `Mono`/`Flux` from API methods |
| `returnResponse` | `true` | Returns `ResponseEntity` wrappers for status code access |
| `useTags` | `true` | Groups API classes by `@Tag` name instead of path |
| `dateLibrary` | `java8-localdatetime` | Uses `LocalDateTime` instead of `OffsetDateTime` |
| `java20` | `true` | Targets Java 20+ features |
| `sourceFolder` | `src/gen/java/main` | Output subfolder for generated sources |

### 4.4 Package Naming Convention

The generated SDK follows this pattern:

```
com.firefly.<domain>.<subdomain>.<service>.sdk.api      -- API client classes
com.firefly.<domain>.<subdomain>.<service>.sdk.model    -- DTO model classes
com.firefly.<domain>.<subdomain>.<service>.sdk.invoker  -- HTTP invoker/config
```

Real examples:

| Service | apiPackage | modelPackage | invokerPackage |
|---------|-----------|--------------|----------------|
| core-banking-accounts | `com.firefly.core.banking.accounts.sdk.api` | `com.firefly.core.banking.accounts.sdk.model` | `com.firefly.core.banking.accounts.sdk.invoker` |
| core-banking-payments | `com.firefly.core.banking.payments.sdk.api` | `com.firefly.core.banking.payments.sdk.model` | `com.firefly.core.banking.payments.sdk.invoker` |
| core-lending-loan-origination | `com.firefly.core.lending.origination.sdk.api` | `com.firefly.core.lending.origination.sdk.model` | `com.firefly.core.lending.origination.sdk.invoker` |

### 4.5 inputSpec Path Patterns

There are two patterns depending on whether the spec is committed to Git or generated at build time:

```xml
<!-- Pattern A: Static spec committed in the SDK module (legacy) -->
<inputSpec>${project.basedir}/src/main/resources/api-spec/openapi.yml</inputSpec>

<!-- Pattern B: Generated spec from -web build output (preferred) -->
<inputSpec>${project.parent.basedir}/${project.parent.artifactId}-web/target/openapi/openapi.yml</inputSpec>
```

Always prefer Pattern B. The spec should never be committed to Git.

---

## 5. OpenAPI Annotation Reference

### Controller-Level: @Tag

```java
@Tag(name = "Payment Orders", description = "APIs for managing payment orders")
@RestController
@RequestMapping("/api/v1/payment-orders")
public class PaymentOrderManagerController { ... }
```

### Method-Level: @Operation

```java
@Operation(
    summary = "Create Payment Order",   // short one-line summary
    description = "Create a new payment order."  // detailed description
)
```

### Response Documentation: @ApiResponses / @ApiResponse

```java
@ApiResponses(value = {
    @ApiResponse(responseCode = "201", description = "Payment order created successfully",
        content = @Content(mediaType = MediaType.APPLICATION_JSON_VALUE,
            schema = @Schema(implementation = PaymentOrderDTO.class))),
    @ApiResponse(responseCode = "400", description = "Invalid payment order data provided",
        content = @Content)
})
```

### Parameter Documentation: @Parameter

```java
@Parameter(description = "Unique identifier of the account", required = true)
@PathVariable("accountId") UUID accountId
```

```java
@Parameter(description = "Data for the new account", required = true,
        schema = @Schema(implementation = AccountDTO.class))
@RequestBody AccountDTO accountDTO
```

### Complex Query Parameters: @ParameterObject

```java
import org.springdoc.core.annotations.ParameterObject;

@GetMapping(produces = MediaType.APPLICATION_JSON_VALUE)
public Mono<ResponseEntity<PaginationResponse<PaymentOrderDTO>>> getAllPaymentOrders(
        @ParameterObject @ModelAttribute FilterRequest<PaymentOrderDTO> filterRequest
) { ... }
```

### Idempotency Opt-Out: @DisableIdempotency

```java
import org.fireflyframework.web.idempotency.annotation.DisableIdempotency;

@PostMapping("/api/resource")
@DisableIdempotency
public Mono<ResponseEntity<Resource>> createResource(@RequestBody Resource resource) {
    // X-Idempotency-Key header will NOT appear in the OpenAPI spec for this operation
}
```

---

## 6. Generated Spec Notes

- Output format is **OpenAPI 3.0.1** (set via `springdoc.api-docs.version: openapi_3_0`)
- The `X-Idempotency-Key` header is automatically injected into every operation by
  `IdempotencyOpenAPICustomizer` unless the controller method has `@DisableIdempotency`
- Tags from `@Tag` annotations become top-level `tags` entries in the spec
- `operationId` is derived from the Java method name
- The spec is written to `<service>-web/target/openapi/openapi.yml`
- Optional springdoc properties for services: `springdoc.swagger-ui.enabled`,
  `springdoc.packages-to-scan`, `springdoc.swagger-ui.path`

---

## 7. Anti-Patterns

### DO NOT: Use @SpringBootApplication for OpenApiGenApplication

```java
// WRONG -- pulls in all auto-configuration, requires infrastructure
@SpringBootApplication
public class OpenApiGenApplication { ... }
```

```java
// CORRECT -- minimal config with @EnableOpenApiGen
@EnableOpenApiGen
@ComponentScan(basePackages = "com.firefly.your.service.web.controllers")
public class OpenApiGenApplication { ... }
```

### DO NOT: Scan Beyond the Controllers Package

```java
// WRONG -- scans services, repos, infrastructure classes
@ComponentScan(basePackages = "com.firefly.your.service")
```

```java
// CORRECT -- only controllers
@ComponentScan(basePackages = "com.firefly.your.service.web.controllers")
```

### DO NOT: Commit the Generated OpenAPI Spec to Git

The spec file at `src/main/resources/api-spec/openapi.yml` is a legacy pattern. The correct
approach is to generate it during the build. The `-sdk` module should read from:
```xml
<inputSpec>${project.parent.basedir}/${project.parent.artifactId}-web/target/openapi/openapi.yml</inputSpec>
```

### DO NOT: Place OpenApiGenApplication in src/main/java

It must live in `src/test/java` so it is never packaged into the production artifact. The Maven
build uses `useTestClasspath=true` to find it.

### DO NOT: Manually Configure Mock Beans for Services

```java
// WRONG -- service-specific mock configuration
@Bean
public AccountService accountService() {
    return Mockito.mock(AccountService.class);
}
```

`AutoMockMissingBeansConfig` handles this automatically for all `@RestController` dependencies
when the `openapi-gen` profile is active.

### DO NOT: Skip @Tag on Controllers

Without `@Tag`, the generated API classes in the SDK will have generic names. Always tag every
controller:

```java
@Tag(name = "Accounts", description = "APIs for managing bank accounts")
```

### DO NOT: Use okhttp Library for Reactive Services

Use `<library>webclient</library>` with `<reactive>true</reactive>`, not `okhttp-gson`.

### DO NOT: Forget the -web Dependency in the -sdk Module

The `-web` module must be declared as a `provided` scope dependency in the `-sdk` POM. This
ensures Maven reactor ordering so the generated spec exists when `-sdk` reads it.

### DO NOT: Use -Dmaven.test.skip=true When Building the -web Module

`OpenApiGenApplication` lives in `src/test/java`. Using `-Dmaven.test.skip=true` skips **test
compilation entirely**, which means the OpenAPI gen application class is not compiled and the
spec will not be generated. Use `-DskipTests` instead (compiles test sources but skips execution):

```bash
# WRONG -- skips test compilation, OpenApiGenApplication is not compiled
mvn verify -pl service-web -Dmaven.test.skip=true

# CORRECT -- compiles test sources (including OpenApiGenApplication) but skips test execution
mvn verify -pl service-web -DskipTests
```

### DO NOT: Forget the -core Dependency in the -web Module

When controllers inject services or reference commands from the `-core` module, the `-web`
module MUST declare a dependency on `-core`:

```xml
<!-- In {service}-web/pom.xml -->
<dependency>
    <groupId>com.firefly</groupId>
    <artifactId>{service}-core</artifactId>
</dependency>
```

Without this dependency, the `-web` module will fail to compile when controllers import service
interfaces, commands, or other types defined in `-core`. The `-interfaces` dependency alone is
not sufficient if controllers directly use `-core` types.

### DO NOT: Create Inverted Dependencies from -interfaces to -core

The `-interfaces` module contains DTOs and enums. It must NEVER depend on `-core`:

```
CORRECT dependency direction:
  -core → -interfaces (core imports DTOs from interfaces)
  -web  → -core       (web imports services from core)
  -web  → -interfaces (web imports DTOs from interfaces)

WRONG (inverted):
  -interfaces → -core  (creates circular dependency)
```

---

## 8. SDK Model DTO Patterns

Generated SDK model DTOs have specific patterns that differ from hand-written DTOs. Understanding
these patterns prevents compilation errors in consumer code.

### 8.1 Read-Only ID Fields

SDK model DTOs generated by `openapi-generator-maven-plugin` have **read-only ID fields**. The ID,
`dateCreated`, and `dateUpdated` fields are set only via the constructor -- there are no setters:

```java
// Generated SDK DTO (simplified)
public class KycVerificationDTO {
    private LocalDateTime dateCreated;  // read-only
    private LocalDateTime dateUpdated;  // read-only
    private UUID kycVerificationId;     // read-only

    // Constructor for read-only fields
    public KycVerificationDTO(LocalDateTime dateCreated, LocalDateTime dateUpdated,
                               UUID kycVerificationId) { ... }

    // Setter for mutable fields only
    public KycVerificationDTO verificationStatus(String verificationStatus) { ... }

    // NO setter for kycVerificationId, dateCreated, dateUpdated
}
```

**Implications for tests:** When mocking SDK responses, use the constructor to set the ID:

```java
// CORRECT -- set ID via constructor
KycVerificationDTO dto = new KycVerificationDTO(null, null, UUID.randomUUID());

// WRONG -- setKycVerificationId() does not exist
KycVerificationDTO dto = new KycVerificationDTO();
dto.setKycVerificationId(UUID.randomUUID());  // Compilation error
```

### 8.2 Fluent Setter Pattern

Mutable fields use a fluent setter pattern (returns `this`) rather than void setters:

```java
// Generated fluent setters
KycVerificationDTO dto = new KycVerificationDTO(null, null, caseId)
    .verificationStatus("VERIFIED")
    .riskScore(85)
    .riskLevel("LOW");
```

### 8.3 Inline Enum Types

Enum-typed fields generate **inner enum classes** within the DTO, not standalone enum types:

```java
// Generated inline enum
SendNotificationCommand command = new SendNotificationCommand()
    .notificationType(SendNotificationCommand.NotificationTypeEnum.WELCOME)
    .priority(SendNotificationCommand.PriorityEnum.NORMAL)
    .channels(List.of(SendNotificationCommand.ChannelsEnum.EMAIL));
```

### 8.4 Getter Naming for IDs

Generated getters use the **full field name** as defined in the OpenAPI spec. If the entity is
`KycVerification` and the ID field is `kycVerificationId`, the getter is `getKycVerificationId()`,
NOT `getId()` or `getDocumentId()`.

**Common mistakes:**
- Using `getId()` instead of `getKycVerificationId()`
- Using `getDocumentId()` instead of `getVerificationDocumentId()`
- Using `getCaseId()` instead of `getComplianceCaseId()`

Always check the generated SDK source to confirm the exact getter name.

---

## 9. Troubleshooting

### App Fails to Start During pre-integration-test

Check that the `@ComponentScan` in `OpenApiGenApplication` points only at the controllers
package. If a new infrastructure bean is being pulled in, it likely means the scan is too broad.

### Spec File Missing When -sdk Builds

The `-web` module must build before the `-sdk` module. Verify:
1. The `-web` module is listed before `-sdk` in the parent POM `<modules>` list
2. The `-sdk` POM has a `provided` dependency on the `-web` module

### Build Takes Too Long

Skip spec generation for quick local rebuilds:
```bash
mvn install -Dopenapi.gen.skip=true
```

### Port Conflicts in Parallel Builds

The `openapi-gen` profile should use `server.port=0` (random port) to avoid conflicts.

---

## 10. File Checklist for a New Service

| File | Location | Purpose |
|------|----------|---------|
| `OpenApiGenApplication.java` | `-web/src/test/java/<pkg>/web/openapi/` | Lightweight app for spec generation |
| `-web/pom.xml` properties | `-web/pom.xml` | Set `openapi.gen.skip` and `openapi.gen.mainClass` |
| `-sdk/pom.xml` | `-sdk/pom.xml` | Full SDK module with openapi-generator-maven-plugin |
| Controller annotations | `-web/src/main/java/<pkg>/web/controllers/` | `@Tag`, `@Operation`, `@ApiResponse` on all endpoints |
| `application.yml` | `-web/src/main/resources/` | `spring.application.name`, version, team info |

The `application-openapi-gen.yaml` profile and `AutoMockMissingBeansConfig` are inherited from
`fireflyframework-web` -- you do NOT create these per service.
