# Firefly Framework — Claude Code Skills

[![License](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)

Claude Code skills for accelerating development with **Firefly Framework** — reactive microservices, CQRS, Sagas, EDA, and more.

> These skills target experienced developers working with the [Firefly Framework](https://github.com/fireflyframework) ecosystem (Spring Boot 3.5, Project Reactor, R2DBC, Java 21+).

## Installation

```bash
# Add the marketplace
/plugin marketplace add fireflyframework/fireflyframework-claude-skills

# Install the plugin
/plugin install fireflyframework-claude-skills@fireflyframework-skills
```

After installation, skills activate automatically based on context. Commands are invoked with `/command-name`.

---

## Skills (18)

### Scaffolding & Conventions

| Skill | When to use |
|-------|-------------|
| **`firefly-conventions`** | Writing any code in a Firefly project. Quick reference for package structure (`org.fireflyframework.{module}`), reactive patterns, DTO naming, CalVer, error handling, and build conventions. |
| **`create-microservice`** | Creating a new microservice. Generates the complete multi-module Maven project (interfaces, models, core, web, sdk) with POM inheritance, Spring Boot config, and Flyway migration. |

### Core Framework Patterns

| Skill | When to use |
|-------|-------------|
| **`implement-cqrs`** | Implementing command or query handlers. Covers `Command<R>`/`Query<R>`, `@CommandHandlerComponent`/`@QueryHandlerComponent`, authorization, validation, caching, metrics, and `ExecutionContext`. |
| **`implement-saga`** | Implementing distributed transactions. Covers `@Saga`/`@SagaStep`, 5 compensation policies, DAG execution, `@Workflow`, `@Tcc`, pluggable persistence. |
| **`implement-eda`** | Implementing event-driven architecture. Covers `EventPublisher`/`@EventListener` over Kafka/RabbitMQ, JSON/Avro/Protobuf serialization, DLQ, circuit breakers, event filtering. |
| **`implement-event-sourcing`** | Implementing event-sourced aggregates. Covers `AggregateRoot`, reactive `EventStore`, optimistic concurrency, snapshots, transactional outbox, projections, event upcasting. |
| **`implement-hexagonal-adapter`** | Creating adapters for existing ports. Port/adapter pattern with real examples from IDP, ECM, and Notifications modules. Includes NoOp fallbacks and adapter testing. |

### Infrastructure & Integration

| Skill | When to use |
|-------|-------------|
| **`implement-reactive-r2dbc`** | Creating R2DBC entities, repositories, or Flyway migrations. Covers entity annotations, `FilterUtils`, `PaginationUtils`, connection pooling, auditing, and MapStruct mappers. |
| **`implement-service-client`** | Consuming external services. Covers `RestClientBuilder`, `GrpcClientBuilder`, `SoapClientBuilder`, `GraphQLClientHelper`, `WebSocketClientHelper` with OAuth2, circuit breakers, rate limiting, and deduplication. |
| **`implement-cache-strategy`** | Adding caching. Covers `FireflyCacheManager` with Caffeine/Redis/Hazelcast, multi-tier (L1+L2) strategies, CQRS query caching with `@InvalidateCacheOn`, SPI extension. |
| **`implement-resilience`** | Adding resilience patterns. Covers `ResilientEventPublisher`, `CircuitBreakerManager`, retry, rate limiting, bulkhead, load shedding, and adaptive timeout with Reactor decorators. |
| **`implement-observability`** | Adding metrics, tracing, or health checks. Covers `FireflyMetricsSupport`, `FireflyTracingSupport`, `FireflyHealthIndicator`, structured logging, and reactive context propagation. |
| **`implement-openapi-gen`** | Generating OpenAPI specs or SDKs. Covers `@EnableOpenApiGen`, springdoc profile, `openapi-generator-maven-plugin` configuration, and generated package conventions. |
| **`implement-rule-engine`** | Implementing business rules. Covers YAML DSL, `ASTRulesDSLParser`, `ASTRulesEvaluationEngine`, batch evaluation, audit trails, and Python compilation. |

### Testing

| Skill | When to use |
|-------|-------------|
| **`testing-reactive-services`** | Writing tests for reactive services. Covers `StepVerifier`, Testcontainers (PostgreSQL, Kafka, Redis), WireMock, `WebTestClient`, and `ExecutionContext` testing. |
| **`testing-cqrs-handlers`** | Testing CQRS handlers specifically. Covers handler unit tests, authorization/validation testing, event emission verification, and metrics assertions. |
| **`testing-sagas`** | Testing sagas and compensation. Covers happy path, compensation per step, all 5 compensation policies, persistence verification, TCC, and `ExpandEach`. |

### Debugging

| Skill | When to use |
|-------|-------------|
| **`debugging-reactive-chains`** | Debugging reactive chain errors. Covers `log()`, `checkpoint()`, `ReactorDebugAgent`, and Firefly-specific error patterns for CQRS, EDA, and Saga modules. |

---

## Commands (4)

Commands generate code when invoked with a slash command:

| Command | Usage | What it generates |
|---------|-------|-------------------|
| **`/create-service`** | `/create-service <name> <tier>` | Complete multi-module Maven project with POM, application class, config, and initial migration. Tier: `core`, `domain`, `application`, `data`, `library`. |
| **`/create-handler`** | `/create-handler <command\|query> <name>` | CQRS handler triplet: Command/Query class + Handler class + Result DTO with validation and authorization. |
| **`/create-saga`** | `/create-saga <name>` | Saga class with `@Saga`/`@SagaStep` annotations, compensation handlers, `@StepEvent` constants, and lifecycle callbacks. |
| **`/create-migration`** | `/create-migration <name>` | Flyway SQL migration file with timestamp naming (`V{yyyyMMddHHmmss}__{name}.sql`) and common SQL patterns. |

---

## Agents (2)

Agents provide specialized review and architecture guidance:

| Agent | Role |
|-------|------|
| **`firefly-code-reviewer`** | Code review validating Firefly conventions: reactive correctness, package structure, naming, RFC 7807 error handling, CQRS patterns, hexagonal architecture, EDA, saga, and data access rules. Findings are severity-tiered (BLOCKER/WARNING/INFO). |
| **`firefly-architect`** | Architecture advisor: tier selection (core/domain/application/data), pattern selection (CQRS/Saga/TCC/Event Sourcing), module design, integration patterns, data strategy, resilience planning, and testing strategy. |

---

## How Skills Work

Skills are context-sensitive — Claude Code automatically loads the relevant skill based on what you're doing:

- **Writing a CQRS handler?** `implement-cqrs` activates and guides you through the real `CommandBus`/`QueryBus` API.
- **Creating a new microservice?** `create-microservice` provides the exact multi-module Maven structure with correct POM inheritance.
- **Debugging a reactive chain?** `debugging-reactive-chains` shows Firefly-specific error patterns and debugging techniques.

All skills reference **real API classes, annotations, and configuration properties** from the Firefly Framework repositories — no invented or generic examples.

## Repository Structure

```
fireflyframework-claude-skills/
  .claude-plugin/
    plugin.json              # Plugin metadata
    marketplace.json         # Self-referencing marketplace
  skills/
    firefly-conventions/SKILL.md
    create-microservice/SKILL.md
    implement-cqrs/SKILL.md
    implement-saga/SKILL.md
    implement-eda/SKILL.md
    implement-event-sourcing/SKILL.md
    implement-hexagonal-adapter/SKILL.md
    implement-reactive-r2dbc/SKILL.md
    implement-service-client/SKILL.md
    implement-cache-strategy/SKILL.md
    implement-resilience/SKILL.md
    implement-observability/SKILL.md
    implement-openapi-gen/SKILL.md
    implement-rule-engine/SKILL.md
    testing-reactive-services/SKILL.md
    testing-cqrs-handlers/SKILL.md
    testing-sagas/SKILL.md
    debugging-reactive-chains/SKILL.md
  commands/
    create-service/SKILL.md
    create-handler/SKILL.md
    create-saga/SKILL.md
    create-migration/SKILL.md
  agents/
    firefly-code-reviewer/SKILL.md
    firefly-architect/SKILL.md
```

## For Banking Platform Developers

If you work on the [Firefly Banking Platform](https://github.com/firefly-oss), install the companion plugin for banking-specific skills:

```bash
/plugin marketplace add firefly-oss/firefly-oss-claude-skills
/plugin install firefly-oss-claude-skills@firefly-oss-skills
```

## Links

- [Firefly Framework GitHub Organization](https://github.com/fireflyframework)
- [Apache 2.0 License](LICENSE)

## License

Copyright 2026 Firefly Software Solutions Inc

Licensed under the Apache License, Version 2.0. See [LICENSE](LICENSE) for details.
