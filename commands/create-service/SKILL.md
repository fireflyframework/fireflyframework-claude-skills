---
name: create-service
description: "Creates a new Firefly Framework microservice with multi-module Maven structure. Usage: /create-service <name> <tier>"
---

# Create Service

Generate a complete Firefly Framework microservice project. Refer to the `create-microservice` skill for full scaffolding details when you need additional context beyond what is described here.

## Parameters

| Parameter | Required | Description | Example |
|-----------|----------|-------------|---------|
| `name` | Yes | Service name in kebab-case (becomes the artifactId) | `core-common-customer-mgmt` |
| `tier` | Yes | One of: `core`, `domain`, `application`, `data`, `library` | `core` |

If either parameter is missing, prompt the user before generating anything.

## Derived Values

Compute these from the parameters:

- **groupId**: `org.fireflyframework`
- **basePackage**: `org.fireflyframework.{name with hyphens replaced by dots}` (e.g., `core-common-customer-mgmt` -> `org.fireflyframework.core.common.customer.mgmt`)
- **packagePath**: basePackage with dots replaced by `/`
- **dbName** (core tier only): name with hyphens replaced by underscores (e.g., `core_common_customer_mgmt`)
- **parentVersion**: `26.02.06` (CalVer -- update if user specifies a different version)

## Generation Instructions

### Core Tier (5 modules)

Generate this directory tree:

```
{name}/
  pom.xml
  {name}-interfaces/
    pom.xml
    src/main/java/{packagePath}/interfaces/
      dtos/
      enums/
  {name}-models/
    pom.xml
    src/main/java/{packagePath}/models/
      entities/
      repositories/
        BaseRepository.java
    src/main/resources/db/migration/
      V1__Initial.sql
  {name}-core/
    pom.xml
    src/main/java/{packagePath}/core/
      services/
        impl/
      mappers/
    src/test/java/{packagePath}/core/services/impl/
  {name}-sdk/
    pom.xml
    src/main/resources/api-spec/
      openapi.yml
  {name}-web/
    pom.xml
    src/main/java/{packagePath}/web/
      Application.java
      controllers/
    src/main/resources/
      application.yaml
```

### Domain Tier (5 modules)

Replace `{name}-models` with `{name}-infra`. The infra module contains client factories and downstream properties instead of entities and repositories. The core module adds `handlers/` and `workflows/` packages. No Flyway migration. No `@EnableR2dbcRepositories`.

### Application / Data / Library Tier

Generate a single-module project (no sub-modules). Application tier includes Spring Security. Library tier has no Spring Boot main class.

## Root pom.xml Template

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
        <version>{parentVersion}</version>
        <relativePath/>
    </parent>

    <groupId>org.fireflyframework</groupId>
    <artifactId>{name}</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <packaging>pom</packaging>

    <modules>
        <module>{name}-interfaces</module>
        <module>{name}-models</module>   <!-- or {name}-infra for domain tier -->
        <module>{name}-core</module>
        <module>{name}-sdk</module>
        <module>{name}-web</module>
    </modules>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.fireflyframework</groupId>
                <artifactId>fireflyframework-bom</artifactId>
                <version>{parentVersion}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
            <!-- Internal module versions -->
            <dependency>
                <groupId>org.fireflyframework</groupId>
                <artifactId>{name}-interfaces</artifactId>
                <version>${project.version}</version>
            </dependency>
            <dependency>
                <groupId>org.fireflyframework</groupId>
                <artifactId>{name}-models</artifactId>
                <version>${project.version}</version>
            </dependency>
            <dependency>
                <groupId>org.fireflyframework</groupId>
                <artifactId>{name}-core</artifactId>
                <version>${project.version}</version>
            </dependency>
            <dependency>
                <groupId>org.fireflyframework</groupId>
                <artifactId>{name}-sdk</artifactId>
                <version>${project.version}</version>
            </dependency>
            <dependency>
                <groupId>org.fireflyframework</groupId>
                <artifactId>{name}-web</artifactId>
                <version>${project.version}</version>
            </dependency>
        </dependencies>
    </dependencyManagement>
</project>
```

## Application.java (Core Tier)

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

For domain tier, omit `@EnableR2dbcRepositories`.

## application.yaml (Core Tier)

```yaml
spring:
  application:
    name: {name}
  r2dbc:
    url: r2dbc:postgresql://${DB_HOST:localhost}:${DB_PORT:5432}/${DB_NAME:{dbName}}
    username: ${DB_USER:postgres}
    password: ${DB_PASS:postgres}
  flyway:
    url: jdbc:postgresql://${DB_HOST:localhost}:${DB_PORT:5432}/${DB_NAME:{dbName}}
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

For domain tier, remove the `r2dbc` and `flyway` sections and add `downstream.services` instead.

## V1__Initial.sql (Core Tier Only)

```sql
-- Initial schema for {name}
-- Add your table definitions here

-- Example:
-- CREATE TABLE example (
--     id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
--     name        VARCHAR(255) NOT NULL,
--     description TEXT,
--     created_at  TIMESTAMP NOT NULL DEFAULT now(),
--     updated_at  TIMESTAMP NOT NULL DEFAULT now()
-- );
```

## BaseRepository.java (Core Tier Only)

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

## After Generation

1. Confirm every module has a valid `pom.xml` with the correct parent reference.
2. Verify the package directories match the basePackage.
3. Tell the user to run `mvn clean install` to validate compilation.
4. Suggest next steps: add an entity, create a CQRS handler, or define a saga.
