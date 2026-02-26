---
name: create-microservice
description: Use when creating a new Firefly Framework microservice — scaffolds the complete multi-module Maven project with interfaces, models, core, web, and sdk modules following framework conventions
---

# Create Microservice

## 1. Checklist -- Gather Before Scaffolding

Ask the user for the following before generating any files:

| Question | Example | Notes |
|----------|---------|-------|
| Service name (kebab-case artifactId) | `core-common-customer-mgmt` | Becomes the root artifactId and directory name |
| Tier | `core`, `domain`, `experience`, `application`, `data` | Determines starter dependency and module layout |
| Group ID | `org.fireflyframework` | Default: `org.fireflyframework` |
| Base Java package | `org.fireflyframework.core.common.customer.mgmt` | Derived from groupId + sanitized artifactId |
| Initial entity name (PascalCase) | `Customer` | Used to generate sample DTO, entity, service, controller |
| Server port | `8080` | Default for application.yaml |
| Database name (core tier only) | `core_common_customer_mgmt` | Derived from artifactId with hyphens replaced by underscores |

## 2. Directory Structure

### Core Tier (5 modules: interfaces, models, core, sdk, web)

```
{artifactId}/
  pom.xml                          # Parent POM (packaging: pom)
  .gitignore
  README.md
  LICENSE
  {artifactId}-interfaces/
    pom.xml
    src/main/java/{package}/interfaces/
      dtos/
        {Entity}DTO.java
      enums/
        {Entity}Status.java
  {artifactId}-models/
    pom.xml
    src/main/java/{package}/models/
      entities/
        {Entity}Entity.java
      repositories/
        BaseRepository.java
        {Entity}Repository.java
    src/main/resources/db/migration/
      V1__Initial.sql
  {artifactId}-core/
    pom.xml
    src/main/java/{package}/core/
      services/
        {Entity}Service.java
        impl/
          {Entity}ServiceImpl.java
      mappers/
        {Entity}Mapper.java
    src/test/java/{package}/core/services/impl/
      {Entity}ServiceImplTest.java
  {artifactId}-sdk/
    pom.xml
    src/main/resources/api-spec/
      openapi.yml
  {artifactId}-web/
    pom.xml
    src/main/java/{package}/web/
      Application.java
      controllers/
        {Entity}Controller.java
    src/main/resources/
      application.yaml
```

### Domain Tier (5 modules: interfaces, infra, core, sdk, web)

The domain tier replaces `models` with `infra` and adds CQRS/Saga packages:

```
{artifactId}/
  pom.xml
  {artifactId}-interfaces/
    src/main/java/{package}/interfaces/
      dtos/
      commands/
        {Entity}Command.java
      queries/
        {Entity}Query.java
  {artifactId}-infra/
    src/main/java/{package}/infra/
      clients/
        ClientFactory.java
        DownstreamProperties.java
  {artifactId}-core/
    src/main/java/{package}/core/
      handlers/
        {Entity}Handler.java
      services/
        {Entity}DomainService.java
        impl/
          {Entity}DomainServiceImpl.java
      workflows/
        Create{Entity}Saga.java
  {artifactId}-sdk/
    src/main/resources/api-spec/openapi.yml
  {artifactId}-web/
    src/main/java/{package}/web/
      Application.java
      controller/
        {Entity}Controller.java
    src/main/resources/application.yaml
```

### Experience Tier (5 modules: interfaces, infra, core, sdk, web)

Experience-tier services (`exp-*`) use the same 5-module structure as domain services: `interfaces`, `infra`, `core`, `sdk`, `web`. The `-infra` module contains client factories for **domain** service SDKs (never core SDKs directly). The `-core` module contains journey orchestration and response shaping. Uses `fireflyframework-starter-application`.

```
{artifactId}/
  pom.xml
  {artifactId}-interfaces/
    src/main/java/{package}/interfaces/
      dtos/                    # Journey-specific request/response DTOs
  {artifactId}-infra/
    src/main/java/{package}/infra/
      clients/
        ClientFactory.java     # Domain SDK client factories
        DownstreamProperties.java
  {artifactId}-core/
    src/main/java/{package}/core/
      services/
        {Journey}Service.java
        impl/
          {Journey}ServiceImpl.java
      mappers/
  {artifactId}-sdk/
    src/main/resources/api-spec/openapi.yml
  {artifactId}-web/
    src/main/java/{package}/web/
      Application.java
      controller/
        {Journey}Controller.java
    src/main/resources/application.yaml
```

### Application Tier (single module)

Application-tier services (`app-*`) are single-module projects for infrastructure concerns (authentication, gateway, config).

### Library (single module, no Spring Boot app)

Library projects produce a JAR with Spring Boot auto-configuration.

## 3. Parent POM

The root `pom.xml` declares the parent, modules, and dependency management. Every service inherits from `fireflyframework-parent` and imports the `fireflyframework-bom`.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
                             http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>org.fireflyframework</groupId>
        <artifactId>fireflyframework-parent</artifactId>
        <version>${PARENT_VERSION}</version>
        <relativePath/>
    </parent>

    <groupId>${GROUP_ID}</groupId>
    <artifactId>${ARTIFACT_ID}</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <packaging>pom</packaging>

    <name>${PROJECT_NAME}</name>
    <description>${DESCRIPTION}</description>

    <!-- Core tier modules -->
    <modules>
        <module>${ARTIFACT_ID}-interfaces</module>
        <module>${ARTIFACT_ID}-models</module>
        <module>${ARTIFACT_ID}-core</module>
        <module>${ARTIFACT_ID}-sdk</module>
        <module>${ARTIFACT_ID}-web</module>
    </modules>

    <!-- Domain tier: replace models with infra -->
    <!--
    <modules>
        <module>${ARTIFACT_ID}-interfaces</module>
        <module>${ARTIFACT_ID}-infra</module>
        <module>${ARTIFACT_ID}-core</module>
        <module>${ARTIFACT_ID}-sdk</module>
        <module>${ARTIFACT_ID}-web</module>
    </modules>
    -->

    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.fireflyframework</groupId>
                <artifactId>fireflyframework-bom</artifactId>
                <version>${PARENT_VERSION}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
            <!-- Internal module versions -->
            <dependency>
                <groupId>${GROUP_ID}</groupId>
                <artifactId>${ARTIFACT_ID}-interfaces</artifactId>
                <version>${project.version}</version>
            </dependency>
            <dependency>
                <groupId>${GROUP_ID}</groupId>
                <artifactId>${ARTIFACT_ID}-models</artifactId>
                <version>${project.version}</version>
            </dependency>
            <dependency>
                <groupId>${GROUP_ID}</groupId>
                <artifactId>${ARTIFACT_ID}-core</artifactId>
                <version>${project.version}</version>
            </dependency>
            <dependency>
                <groupId>${GROUP_ID}</groupId>
                <artifactId>${ARTIFACT_ID}-web</artifactId>
                <version>${project.version}</version>
            </dependency>
            <dependency>
                <groupId>${GROUP_ID}</groupId>
                <artifactId>${ARTIFACT_ID}-sdk</artifactId>
                <version>${project.version}</version>
            </dependency>
        </dependencies>
    </dependencyManagement>
</project>
```

Replace `${PARENT_VERSION}` with the current framework CalVer (e.g. `26.02.03`). The parent version can be found in `~/.flywork/config.yaml` under `parent_version`, or by checking the latest `fireflyframework-parent` release.

## 4. Module: interfaces

**Purpose:** Shared DTOs, enums, validation annotations, and API contracts. This module has ZERO business logic. Other services depend on this module to consume the service's API types.

**POM -- core tier:**

```xml
<parent>
    <groupId>${GROUP_ID}</groupId>
    <artifactId>${ARTIFACT_ID}</artifactId>
    <version>${VERSION}</version>
</parent>

<artifactId>${ARTIFACT_ID}-interfaces</artifactId>

<dependencies>
    <dependency>
        <groupId>org.fireflyframework</groupId>
        <artifactId>fireflyframework-utils</artifactId>
    </dependency>
    <dependency>
        <groupId>org.fireflyframework</groupId>
        <artifactId>fireflyframework-validators</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-validation</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-webflux</artifactId>
    </dependency>
    <dependency>
        <groupId>io.swagger.core.v3</groupId>
        <artifactId>swagger-annotations</artifactId>
    </dependency>
    <dependency>
        <groupId>org.mapstruct</groupId>
        <artifactId>mapstruct</artifactId>
    </dependency>
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <scope>provided</scope>
    </dependency>
    <dependency>
        <groupId>com.fasterxml.jackson.core</groupId>
        <artifactId>jackson-databind</artifactId>
    </dependency>
    <dependency>
        <groupId>com.fasterxml.jackson.datatype</groupId>
        <artifactId>jackson-datatype-jsr310</artifactId>
    </dependency>
</dependencies>
```

**Domain tier interfaces** additionally include `fireflyframework-orchestration` and `jakarta.validation-api`, and add `commands/` and `queries/` packages alongside `dtos/`.

**Example DTO:**

```java
package {basePackage}.interfaces.dtos;

import com.fasterxml.jackson.annotation.JsonProperty;
import jakarta.validation.constraints.NotNull;
import jakarta.validation.constraints.Size;
import lombok.*;

import java.time.LocalDateTime;
import java.util.UUID;

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class {Entity}DTO {

    @JsonProperty(access = JsonProperty.Access.READ_ONLY)
    private UUID id;

    @NotNull(message = "Name is required")
    @Size(max = 255, message = "Name must not exceed 255 characters")
    private String name;

    private String description;
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
}
```

**Example enum:**

```java
package {basePackage}.interfaces.enums;

public enum {Entity}Status {
    ACTIVE,
    INACTIVE,
    PENDING
}
```

## 5. Module: models (Core Tier Only)

**Purpose:** R2DBC entities, reactive repositories, and Flyway SQL migrations. This module owns the database schema. Domain-tier services do NOT have a models module (they consume core services via SDKs).

**POM:**

```xml
<artifactId>${ARTIFACT_ID}-models</artifactId>

<dependencies>
    <dependency>
        <groupId>${GROUP_ID}</groupId>
        <artifactId>${ARTIFACT_ID}-interfaces</artifactId>
    </dependency>
    <dependency>
        <groupId>org.fireflyframework</groupId>
        <artifactId>fireflyframework-r2dbc</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-r2dbc</artifactId>
    </dependency>
    <dependency>
        <groupId>org.postgresql</groupId>
        <artifactId>r2dbc-postgresql</artifactId>
    </dependency>
    <dependency>
        <groupId>org.postgresql</groupId>
        <artifactId>postgresql</artifactId>
    </dependency>
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <scope>provided</scope>
    </dependency>
</dependencies>

<build>
    <resources>
        <resource>
            <directory>src/main/resources</directory>
            <filtering>false</filtering>
            <includes>
                <include>**/*.sql</include>
            </includes>
        </resource>
    </resources>
</build>
```

**Example entity:**

```java
package {basePackage}.models.entities;

import lombok.*;
import org.springframework.data.annotation.Id;
import org.springframework.data.relational.core.mapping.Column;
import org.springframework.data.relational.core.mapping.Table;

import java.time.LocalDateTime;
import java.util.UUID;

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
@Table("{entity_snake_case}")
public class {Entity}Entity {

    @Id
    @Column("id")
    private UUID id;

    @Column("name")
    private String name;

    @Column("description")
    private String description;

    @Column("created_at")
    private LocalDateTime createdAt;

    @Column("updated_at")
    private LocalDateTime updatedAt;
}
```

**BaseRepository (always generated):**

```java
package {basePackage}.models.repositories;

import org.springframework.data.domain.Pageable;
import org.springframework.data.repository.NoRepositoryBean;
import org.springframework.data.repository.reactive.ReactiveCrudRepository;
import reactor.core.publisher.Flux;
import reactor.core.publisher.Mono;

@NoRepositoryBean
public interface BaseRepository<T, ID> extends ReactiveCrudRepository<T, ID> {
    Flux<T> findAllBy(Pageable pageable);
    Mono<Long> count();
}
```

**Entity repository:**

```java
@Repository
public interface {Entity}Repository extends BaseRepository<{Entity}Entity, UUID> {
}
```

**Flyway migration** at `src/main/resources/db/migration/V1__Initial.sql`:

```sql
CREATE TABLE {entity_snake_case} (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name        VARCHAR(255) NOT NULL,
    description TEXT,
    created_at  TIMESTAMP NOT NULL DEFAULT now(),
    updated_at  TIMESTAMP NOT NULL DEFAULT now()
);
```

Flyway naming convention: `V{number}__{description}.sql` (two underscores). Place all migrations in `src/main/resources/db/migration/`.

## 6. Module: core

**Purpose:** Business logic, service interfaces and implementations, MapStruct mappers. For domain tier, this also contains CQRS handlers and saga workflows.

**POM (core tier):**

```xml
<artifactId>${ARTIFACT_ID}-core</artifactId>

<dependencies>
    <dependency>
        <groupId>${GROUP_ID}</groupId>
        <artifactId>${ARTIFACT_ID}-interfaces</artifactId>
    </dependency>
    <dependency>
        <groupId>${GROUP_ID}</groupId>
        <artifactId>${ARTIFACT_ID}-models</artifactId>
    </dependency>
    <dependency>
        <groupId>org.fireflyframework</groupId>
        <artifactId>fireflyframework-core</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-webflux</artifactId>
    </dependency>
    <dependency>
        <groupId>org.mapstruct</groupId>
        <artifactId>mapstruct</artifactId>
    </dependency>
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <scope>provided</scope>
    </dependency>
</dependencies>

<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-compiler-plugin</artifactId>
            <configuration>
                <annotationProcessorPaths>
                    <path>
                        <groupId>org.projectlombok</groupId>
                        <artifactId>lombok</artifactId>
                        <version>${lombok.version}</version>
                    </path>
                    <path>
                        <groupId>org.mapstruct</groupId>
                        <artifactId>mapstruct-processor</artifactId>
                        <version>${mapstruct.version}</version>
                    </path>
                    <path>
                        <groupId>org.projectlombok</groupId>
                        <artifactId>lombok-mapstruct-binding</artifactId>
                        <version>${lombok-mapstruct-binding.version}</version>
                    </path>
                </annotationProcessorPaths>
            </configuration>
        </plugin>
    </plugins>
</build>
```

**Domain tier core POM** replaces `fireflyframework-core` + `{artifactId}-models` with `fireflyframework-utils`, `fireflyframework-domain`, `fireflyframework-cqrs`, `fireflyframework-orchestration`, and `fireflyframework-eda`. It depends on `{artifactId}-interfaces` but NOT on a models module.

**Example service interface (core tier):**

```java
package {basePackage}.core.services;

import {basePackage}.interfaces.dtos.{Entity}DTO;
import reactor.core.publisher.Mono;
import java.util.UUID;

public interface {Entity}Service {
    Mono<{Entity}DTO> create({Entity}DTO dto);
    Mono<{Entity}DTO> getById(UUID id);
    Mono<{Entity}DTO> update(UUID id, {Entity}DTO dto);
    Mono<Void> delete(UUID id);
}
```

**Example MapStruct mapper:**

```java
package {basePackage}.core.mappers;

import {basePackage}.models.entities.{Entity}Entity;
import {basePackage}.interfaces.dtos.{Entity}DTO;
import org.mapstruct.Mapper;
import org.mapstruct.MappingTarget;
import org.mapstruct.NullValuePropertyMappingStrategy;

@Mapper(componentModel = "spring",
        nullValuePropertyMappingStrategy = NullValuePropertyMappingStrategy.IGNORE)
public interface {Entity}Mapper {
    {Entity}DTO toDto({Entity}Entity entity);
    {Entity}Entity toEntity({Entity}DTO dto);
    void updateEntity({Entity}DTO dto, @MappingTarget {Entity}Entity entity);
}
```

## 7. Module: web

**Purpose:** Spring Boot application entry point, REST controllers, application configuration, and OpenAPI documentation. This is the only module that produces a runnable JAR.

**POM:**

```xml
<artifactId>${ARTIFACT_ID}-web</artifactId>

<dependencies>
    <dependency>
        <groupId>${GROUP_ID}</groupId>
        <artifactId>${ARTIFACT_ID}-core</artifactId>
    </dependency>
    <dependency>
        <groupId>${GROUP_ID}</groupId>
        <artifactId>${ARTIFACT_ID}-interfaces</artifactId>
    </dependency>
    <dependency>
        <groupId>org.fireflyframework</groupId>
        <artifactId>fireflyframework-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-webflux</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-validation</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springdoc</groupId>
        <artifactId>springdoc-openapi-starter-webflux-ui</artifactId>
    </dependency>
    <dependency>
        <groupId>io.micrometer</groupId>
        <artifactId>micrometer-registry-prometheus</artifactId>
    </dependency>
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <scope>provided</scope>
    </dependency>
</dependencies>

<build>
    <finalName>${project.parent.artifactId}</finalName>
    <resources>
        <resource>
            <directory>src/main/resources</directory>
            <filtering>true</filtering>
            <includes>
                <include>**/*.yml</include>
                <include>**/*.yaml</include>
                <include>**/*.properties</include>
                <include>**/*.xml</include>
            </includes>
        </resource>
        <resource>
            <directory>src/main/resources</directory>
            <filtering>false</filtering>
            <excludes>
                <exclude>**/*.yml</exclude>
                <exclude>**/*.yaml</exclude>
                <exclude>**/*.properties</exclude>
                <exclude>**/*.xml</exclude>
            </excludes>
        </resource>
    </resources>
    <plugins>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
            <configuration>
                <excludes>
                    <exclude>
                        <groupId>org.projectlombok</groupId>
                        <artifactId>lombok</artifactId>
                    </exclude>
                </excludes>
                <layers>
                    <enabled>true</enabled>
                </layers>
            </configuration>
            <executions>
                <execution>
                    <goals>
                        <goal>repackage</goal>
                        <goal>build-info</goal>
                    </goals>
                </execution>
            </executions>
        </plugin>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-resources-plugin</artifactId>
            <configuration>
                <delimiters>
                    <delimiter>@</delimiter>
                </delimiters>
                <useDefaultDelimiters>false</useDefaultDelimiters>
            </configuration>
        </plugin>
    </plugins>
</build>
```

**Spring Boot main class (core tier):**

```java
package {basePackage}.web;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.data.r2dbc.repository.config.EnableR2dbcRepositories;

@SpringBootApplication(scanBasePackages = "{basePackage}")
@EnableR2dbcRepositories(basePackages = "{basePackage}.models.repositories")
public class Application {

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

**Domain tier main class** (no `@EnableR2dbcRepositories` since domain services do not own a database):

```java
package {basePackage}.web;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication(scanBasePackages = "{basePackage}")
public class Application {

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

**application.yaml (core tier):**

```yaml
spring:
  application:
    name: ${ARTIFACT_ID}
  r2dbc:
    url: r2dbc:postgresql://${DB_HOST:localhost}:${DB_PORT:5432}/${DB_NAME:my_db}
    username: ${DB_USER:postgres}
    password: ${DB_PASS:postgres}
  flyway:
    url: jdbc:postgresql://${DB_HOST:localhost}:${DB_PORT:5432}/${DB_NAME:my_db}
    user: ${DB_USER:postgres}
    password: ${DB_PASS:postgres}
    locations: classpath:db/migration

server:
  port: ${SERVER_PORT:8080}

management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  endpoint:
    health:
      show-details: always

logging:
  level:
    root: INFO
    {basePackage}: DEBUG
    org.fireflyframework: DEBUG
```

**application.yaml (domain tier):** No database section; adds downstream service URLs instead:

```yaml
spring:
  application:
    name: ${ARTIFACT_ID}

server:
  port: ${SERVER_PORT:8080}

downstream:
  services:
    example-service: ${DOWNSTREAM_EXAMPLE_URL:http://localhost:8081}

management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  endpoint:
    health:
      show-details: always

logging:
  level:
    root: INFO
    {basePackage}: DEBUG
    org.fireflyframework: DEBUG
```

**Example controller:**

```java
package {basePackage}.web.controllers;

import {basePackage}.core.services.{Entity}Service;
import {basePackage}.interfaces.dtos.{Entity}DTO;
import lombok.RequiredArgsConstructor;
import org.springframework.http.HttpStatus;
import org.springframework.web.bind.annotation.*;
import reactor.core.publisher.Mono;
import java.util.UUID;

@RestController
@RequestMapping("/api/v1/{entities}")
@RequiredArgsConstructor
public class {Entity}Controller {

    private final {Entity}Service {entity}Service;

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public Mono<{Entity}DTO> create(@RequestBody {Entity}DTO dto) {
        return {entity}Service.create(dto);
    }

    @GetMapping("/{id}")
    public Mono<{Entity}DTO> getById(@PathVariable UUID id) {
        return {entity}Service.getById(id);
    }

    @PutMapping("/{id}")
    public Mono<{Entity}DTO> update(@PathVariable UUID id, @RequestBody {Entity}DTO dto) {
        return {entity}Service.update(id, dto);
    }

    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public Mono<Void> delete(@PathVariable UUID id) {
        return {entity}Service.delete(id);
    }
}
```

## 8. Module: sdk

**Purpose:** Auto-generated client SDK from the OpenAPI specification. Other services depend on this module to call the service's REST API programmatically. The SDK is generated at build time using `openapi-generator-maven-plugin` with the `webclient` library (reactive).

**POM (key sections):**

```xml
<artifactId>${ARTIFACT_ID}-sdk</artifactId>

<properties>
    <openapi-generator.version>7.0.1</openapi-generator.version>
    <generated-sources-dir>${project.build.directory}/generated-sources</generated-sources-dir>
</properties>

<dependencies>
    <dependency>
        <groupId>${GROUP_ID}</groupId>
        <artifactId>${ARTIFACT_ID}-web</artifactId>
        <scope>provided</scope>
    </dependency>
    <dependency>
        <groupId>io.swagger.core.v3</groupId>
        <artifactId>swagger-annotations</artifactId>
    </dependency>
    <dependency>
        <groupId>com.fasterxml.jackson.core</groupId>
        <artifactId>jackson-databind</artifactId>
    </dependency>
    <dependency>
        <groupId>com.squareup.okhttp3</groupId>
        <artifactId>okhttp</artifactId>
    </dependency>
    <dependency>
        <groupId>com.squareup.okhttp3</groupId>
        <artifactId>logging-interceptor</artifactId>
    </dependency>
    <dependency>
        <groupId>org.openapitools</groupId>
        <artifactId>jackson-databind-nullable</artifactId>
    </dependency>
</dependencies>

<build>
    <plugins>
        <plugin>
            <groupId>org.openapitools</groupId>
            <artifactId>openapi-generator-maven-plugin</artifactId>
            <version>${openapi-generator.version}</version>
            <executions>
                <execution>
                    <id>${ARTIFACT_ID}-sdk</id>
                    <goals>
                        <goal>generate</goal>
                    </goals>
                    <configuration>
                        <inputSpec>${project.basedir}/src/main/resources/api-spec/openapi.yml</inputSpec>
                        <generatorName>java</generatorName>
                        <library>webclient</library>
                        <output>${generated-sources-dir}</output>
                        <apiPackage>{basePackage}.sdk.api</apiPackage>
                        <modelPackage>{basePackage}.sdk.model</modelPackage>
                        <invokerPackage>{basePackage}.sdk.invoker</invokerPackage>
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

**OpenAPI spec** at `src/main/resources/api-spec/openapi.yml`:

```yaml
openapi: 3.0.1
info:
  title: ${ARTIFACT_ID}
  description: ${DESCRIPTION}
  contact:
    name: Development Team
    email: dev@example.com
  version: ${VERSION}
servers:
  - url: /
    description: Local Development Environment
tags:
  - name: {Entity}
    description: {Entity} API endpoints
paths:
  /api/v1/{entities}:
    get:
      tags:
        - {Entity}
      summary: List {entities}
      operationId: list{Entities}
      responses:
        '200':
          description: Successfully retrieved {entities}
          content:
            application/json:
              schema:
                type: array
                items:
                  $ref: '#/components/schemas/{Entity}DTO'
    post:
      tags:
        - {Entity}
      summary: Create a {entity}
      operationId: create{Entity}
      requestBody:
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/{Entity}DTO'
        required: true
      responses:
        '201':
          description: {Entity} created
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/{Entity}DTO'
components:
  schemas:
    {Entity}DTO:
      type: object
      properties:
        id:
          type: string
          format: uuid
        name:
          type: string
        description:
          type: string
        createdAt:
          type: string
          format: date-time
```

## 9. Tier-Specific Configuration

| Aspect | Core | Domain | Experience | Application | Data |
|--------|------|--------|------------|-------------|------|
| **Starter** | `fireflyframework-starter-core` | `fireflyframework-starter-domain` | `fireflyframework-starter-application` | `fireflyframework-starter-application` | `fireflyframework-starter-data` |
| **Multi-module** | Yes (5 modules) | Yes (5 modules) | Yes (5 modules) | No (single module) | No (single module) |
| **Modules** | interfaces, models, core, sdk, web | interfaces, infra, core, sdk, web | interfaces, infra, core, sdk, web | single JAR | single JAR |
| **Database (R2DBC)** | Yes -- owns its schema | No -- calls core services via SDK | No | No |
| **Flyway migrations** | Yes (`models` module) | No | No | No |
| **CQRS** | Optional (via starter) | Yes -- `fireflyframework-cqrs` | Optional | Yes -- `fireflyframework-cqrs` |
| **Sagas/Orchestration** | Optional | Yes -- `fireflyframework-orchestration` | Optional | Optional |
| **EDA** | Yes -- `fireflyframework-eda` | Yes -- `fireflyframework-eda` | Yes -- `fireflyframework-eda` | Yes -- `fireflyframework-eda` |
| **Service client** | Optional | Yes -- `fireflyframework-client` | Yes -- `fireflyframework-client` | Yes -- `fireflyframework-client` |
| **Spring Security** | No | No (via starter) | Yes -- `spring-boot-starter-security` | No |
| **Cache** | Optional (Caffeine) | No | Yes -- `fireflyframework-cache` | Yes -- `fireflyframework-cache` |
| **Resilience4j** | Yes | No (via Spring Retry) | Yes | Yes (full suite) |
| **Spring Cloud** | Optional (Config, Eureka, Consul) | No | Optional (Gateway) | No |
| **`@EnableR2dbcRepositories`** | Yes (in Application.java) | No | No | No |
| **`application.yaml` DB section** | Yes (R2DBC + Flyway) | No | No | No |
| **`downstream.services` section** | No | Yes | Varies | Varies |
| **Typical responsibility** | CRUD, data access, schema | Orchestration, business rules | API gateway, auth, context | Data processing, reporting |

### Which tier to use

- **Core**: The service owns a database table and provides CRUD operations. Examples: `core-common-customer-mgmt`, `core-banking-ledger`.
- **Domain**: The service orchestrates multiple core services, implements business workflows (sagas), and applies domain rules. No direct database access. Examples: `domain-customer-people`, `domain-lending-loan-origination`.
- **Experience**: The service is a BFF organized by user journey. Consumes domain SDKs only (never core directly). Aggregates and shapes responses for frontends. Examples: `exp-onboarding`, `exp-lending`, `exp-payments`.
- **Application**: The service provides infrastructure edge concerns: authentication, gateway, configuration. Not for user-journey BFFs. Examples: `app-authenticator`, `app-gateway`, `app-backoffice`.
- **Data**: The service processes data, runs reports, or performs ETL. Examples: `core-data-*` services.

## 10. Post-Scaffolding Verification

After generating all files, verify the project compiles:

```bash
cd {artifactId}
mvn clean install     # for core/domain (multi-module)
mvn clean package     # for application/library (single-module)
```

**Expected outcomes:**

1. All modules compile without errors
2. MapStruct generates mapper implementations (`target/generated-sources/`)
3. For core tier: Flyway migration is found at `{artifactId}-models/src/main/resources/db/migration/`
4. For SDK module: OpenAPI generator produces client code under `target/generated-sources/src/gen/java/main/`
5. Spring Boot repackage produces runnable JAR in `{artifactId}-web/target/`
6. Unit tests (if any) pass

**Next steps after verification:**

1. Run `flywork doctor` to validate the environment
2. Start a local PostgreSQL (core tier) or verify downstream services are accessible (domain tier)
3. Run the application: `java -jar {artifactId}-web/target/{artifactId}.jar`
4. Open Swagger UI: `http://localhost:{port}/webjars/swagger-ui/index.html`
5. Add the new service's SDK to the organization BOM if other services will consume it
