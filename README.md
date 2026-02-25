# Firefly Framework — Claude Code Skills

[![License](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)

Claude Code skills for accelerating development with **Firefly Framework** — reactive microservices, CQRS, Sagas, EDA, and more.

> These skills target experienced developers working with the [Firefly Framework](https://github.com/fireflyframework) ecosystem (Spring Boot 3.5, Project Reactor, R2DBC, Java 21+).

---

## Table of Contents

- [Installation](#installation)
- [How Skills Work](#how-skills-work)
  - [Skills (Context-Sensitive)](#skills-context-sensitive)
  - [Commands (Slash-Invoked)](#commands-slash-invoked)
  - [Agents (Specialized Reviewers)](#agents-specialized-reviewers)
- [Skills Catalog (18)](#skills-catalog-18)
  - [Scaffolding & Conventions](#scaffolding--conventions)
  - [Core Framework Patterns](#core-framework-patterns)
  - [Infrastructure & Integration](#infrastructure--integration)
  - [Testing](#testing)
  - [Debugging](#debugging)
- [Commands Catalog (4)](#commands-catalog-4)
- [Agents Catalog (2)](#agents-catalog-2)
- [Usage Guide](#usage-guide)
  - [Skill Activation & Chaining](#skill-activation--chaining)
  - [Using Commands Effectively](#using-commands-effectively)
  - [Working with Agents](#working-with-agents)
  - [Combining Skills, Commands, and Agents](#combining-skills-commands-and-agents)
- [Practical Examples](#practical-examples)
  - [Example 1: Creating a New Domain-Tier Microservice with CQRS + Saga](#example-1-creating-a-new-domain-tier-microservice-with-cqrs--saga)
  - [Example 2: Adding Event Sourcing to an Existing Service](#example-2-adding-event-sourcing-to-an-existing-service)
  - [Example 3: Implementing a Service Client with Caching and Resilience](#example-3-implementing-a-service-client-with-caching-and-resilience)
  - [Example 4: Full TDD Workflow for a CQRS Handler](#example-4-full-tdd-workflow-for-a-cqrs-handler)
  - [Example 5: Debugging a Failing Reactive Chain](#example-5-debugging-a-failing-reactive-chain)
- [Repository Structure](#repository-structure)
- [For Banking Platform Developers](#for-banking-platform-developers)
- [Links](#links)
- [License](#license)

---

## Installation

```bash
# Add the marketplace
/plugin marketplace add fireflyframework/fireflyframework-claude-skills

# Install the plugin
/plugin install fireflyframework-claude-skills@fireflyframework-skills
```

After installation, skills activate automatically based on context. Commands are invoked with `/command-name`.

---

## How Skills Work

This plugin provides three types of Claude Code extensions. Understanding when and how each type activates is key to getting the most out of them.

### Skills (Context-Sensitive)

Skills activate **automatically** when Claude Code detects you're working on a relevant task. You don't need to invoke them — just describe what you want to do.

**How they activate:** Claude Code reads your prompt and the surrounding code context. When it recognizes a pattern that matches a skill (e.g., "implement a command handler", "add caching to this query"), it loads the skill and uses its guidance to produce framework-correct code.

**What they contain:** Each skill is a detailed reference document with real API classes, annotations, configuration properties, code patterns, and anti-patterns from the actual Firefly Framework source code. They are not generic tutorials — they reference `org.fireflyframework.*` classes directly.

**When multiple skills apply:** Skills can chain. For example, asking "create a saga that publishes events" will activate both `implement-saga` and `implement-eda`. Claude Code merges their guidance to produce code that correctly uses both the orchestration and event-driven modules.

### Commands (Slash-Invoked)

Commands are **explicitly invoked** with a slash prefix and generate complete code scaffolds.

**How they work:** Type `/command-name <args>` in your Claude Code prompt. The command generates a full file tree with correct POM inheritance, package structure, annotations, and boilerplate — ready for you to fill in business logic.

**When to use them:** Use commands at the start of new work — a new service, a new handler, a new saga. They ensure the foundational structure is correct so you can focus on implementation.

### Agents (Specialized Reviewers)

Agents are **invoked by Claude Code** when it determines your task benefits from specialized review or architectural guidance. You can also request them explicitly (e.g., "review this code with the Firefly reviewer").

**How they work:** Agents have deep domain knowledge and structured review templates. They analyze your code or architecture proposal against Firefly Framework conventions and produce severity-tiered findings.

**When to use them:** After implementing a feature (code review), before starting a complex feature (architecture advice), or when you're unsure about tier placement, pattern selection, or integration strategy.

---

## Skills Catalog (18)

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

## Commands Catalog (4)

Commands generate code when invoked with a slash command:

| Command | Usage | What it generates |
|---------|-------|-------------------|
| **`/create-service`** | `/create-service <name> <tier>` | Complete multi-module Maven project with POM, application class, config, and initial migration. Tier: `core`, `domain`, `application`, `data`, `library`. |
| **`/create-handler`** | `/create-handler <command\|query> <name>` | CQRS handler triplet: Command/Query class + Handler class + Result DTO with validation and authorization. |
| **`/create-saga`** | `/create-saga <name>` | Saga class with `@Saga`/`@SagaStep` annotations, compensation handlers, `@StepEvent` constants, and lifecycle callbacks. |
| **`/create-migration`** | `/create-migration <name>` | Flyway SQL migration file with timestamp naming (`V{yyyyMMddHHmmss}__{name}.sql`) and common SQL patterns. |

---

## Agents Catalog (2)

| Agent | Role | When to invoke |
|-------|------|----------------|
| **`firefly-code-reviewer`** | Validates Firefly conventions: reactive correctness, package structure, naming, RFC 7807 error handling, CQRS patterns, hexagonal architecture, EDA, saga, and data access rules. Findings are severity-tiered (BLOCKER/WARNING/INFO). | After implementing a feature or before creating a PR. Ask: *"Review this code with the Firefly reviewer"* or *"Check this handler follows Firefly conventions"*. |
| **`firefly-architect`** | Architecture advisor: tier selection (core/domain/application/data), pattern selection (CQRS/Saga/TCC/Event Sourcing), module design, integration patterns, data strategy, resilience planning, and testing strategy. | Before starting a new feature or service. Ask: *"Help me design the architecture for X"* or *"Which tier should this new service go in?"*. |

---

## Usage Guide

### Skill Activation & Chaining

Skills activate based on the **intent** in your prompt. Here are the key patterns:

| What you say | Skills that activate |
|--------------|---------------------|
| "Create a command handler for X" | `implement-cqrs` |
| "Add a saga for the registration flow" | `implement-saga` |
| "Publish an event when the order completes" | `implement-eda` |
| "Create an event-sourced aggregate for Account" | `implement-event-sourcing` |
| "Add a Keycloak adapter" | `implement-hexagonal-adapter` |
| "Create an R2DBC entity for transactions" | `implement-reactive-r2dbc` |
| "Call the customer service via REST" | `implement-service-client` |
| "Cache the product query results" | `implement-cache-strategy` + `implement-cqrs` |
| "Add circuit breaker to the payment call" | `implement-resilience` |
| "Add metrics to this handler" | `implement-observability` + `implement-cqrs` |
| "Generate the OpenAPI spec for this service" | `implement-openapi-gen` |
| "Write tests for this saga" | `testing-sagas` |
| "My reactive chain is throwing a weird error" | `debugging-reactive-chains` |

**Chaining** happens naturally. When you say "Create a saga that calls the customer service and publishes an event on completion", Claude Code loads `implement-saga` + `implement-service-client` + `implement-eda` and produces code that correctly uses all three patterns.

### Using Commands Effectively

Commands are your starting point for new artifacts. Use them **before** adding business logic:

```
# 1. Scaffold a new core-tier service
/create-service inventory core

# 2. Generate a command handler inside that service
/create-handler command CreateInventoryItem

# 3. Generate a saga for a multi-step operation
/create-saga RegisterInventoryItem

# 4. Create the database migration
/create-migration create_inventory_items_table
```

After each command generates the scaffold, you fill in the business logic. The skills (e.g., `implement-cqrs`, `implement-saga`) guide you through the implementation details.

### Working with Agents

**Code Reviewer** — invoke after writing code to catch convention violations before they reach PR review:

```
> Review my changes to the InventoryCommandHandler with the Firefly reviewer

firefly-code-reviewer activates and checks:
  - Reactive chain correctness (no blocking calls)
  - Package structure (org.fireflyframework.{module})
  - DTO naming conventions (suffix: DTO, Command, Query)
  - Error handling (RFC 7807 ProblemDetail)
  - CQRS patterns (handler returns Mono<R>)
  - Data access rules (R2DBC patterns)
```

**Architect** — invoke before starting design to get tier/pattern guidance:

```
> I need to build an inventory management system. Help me design the architecture.

firefly-architect activates and provides:
  - Tier recommendation (core vs domain vs app)
  - Pattern selection (CQRS, Event Sourcing, Saga)
  - Module decomposition
  - Integration strategy (SDK sync vs Kafka async)
  - Data architecture
  - Testing strategy
```

### Combining Skills, Commands, and Agents

The typical workflow for a new feature:

```
1. ASK THE ARCHITECT    → "Where should inventory management go?"
                          Architect recommends: core-tier service, CQRS + Event Sourcing

2. SCAFFOLD             → /create-service inventory core
                          Generates multi-module Maven project

3. CREATE ENTITIES      → "Create R2DBC entities for inventory items"
                          implement-reactive-r2dbc activates

4. ADD HANDLERS         → /create-handler command CreateInventoryItem
                          /create-handler query GetInventoryItem
                          implement-cqrs activates for business logic

5. ADD EVENTS           → "Publish InventoryItemCreated event via Kafka"
                          implement-eda activates

6. ADD RESILIENCE       → "Add circuit breaker and retry to the supplier API call"
                          implement-resilience activates

7. WRITE TESTS          → "Write tests for the CreateInventoryItem handler"
                          testing-cqrs-handlers activates

8. REVIEW               → "Review all my changes with the Firefly reviewer"
                          firefly-code-reviewer scans for convention violations
```

---

## Practical Examples

### Example 1: Creating a New Domain-Tier Microservice with CQRS + Saga

**Scenario:** You need to build `domain-order-fulfillment`, a domain-tier service that orchestrates order placement across `core-inventory`, `core-payments`, and `core-shipping`.

**Step 1: Ask the architect for guidance**

```
> I need a service that orchestrates order fulfillment across inventory,
> payments, and shipping. Help me design the architecture.
```

The `firefly-architect` agent will recommend a domain-tier service with a saga, since it orchestrates 3 core services without owning data.

**Step 2: Scaffold the service**

```
/create-service order-fulfillment domain
```

This generates the complete Maven project:

```
domain-order-fulfillment/
  domain-order-fulfillment-interfaces/   # Commands, Queries, DTOs
  domain-order-fulfillment-infra/        # SDK clients, config
  domain-order-fulfillment-core/         # Saga, handlers
  domain-order-fulfillment-web/          # REST controllers
  domain-order-fulfillment-sdk/          # Client SDK for consumers
  pom.xml                               # Parent POM with fireflyframework-parent
```

**Step 3: Create the saga**

```
/create-saga FulfillOrder
```

This generates:

```java
@Saga(id = "fulfill-order")
public class FulfillOrderSaga {

    @SagaStep(id = "reserveInventory", compensate = "releaseInventory")
    @StepEvent(type = "inventory.reserved")
    public Mono<UUID> reserveInventory(ReserveInventoryCommand cmd, SagaContext ctx) { ... }

    public Mono<Void> releaseInventory(UUID reservationId, SagaContext ctx) { ... }

    @SagaStep(id = "processPayment", compensate = "refundPayment", dependsOn = "reserveInventory")
    @StepEvent(type = "payment.processed")
    public Mono<UUID> processPayment(ProcessPaymentCommand cmd, SagaContext ctx) { ... }

    // ... more steps
}
```

**Step 4: Add SDK clients for downstream core services**

```
> Add SDK clients for core-inventory, core-payments, and core-shipping
```

The `implement-service-client` skill activates and generates `@ConfigurationProperties` + `ClientFactory` beans in the `-infra` module, following the 4-step SDK integration pattern.

**Step 5: Add resilience to external calls**

```
> Add circuit breaker and retry to all SDK calls in the saga
```

The `implement-resilience` skill wraps each SDK call with `Mono.transformDeferred(CircuitBreakerOperator.of(...))` and configures Resilience4j beans.

**Step 6: Write saga tests**

```
> Write tests for the FulfillOrderSaga including compensation
```

The `testing-sagas` skill generates tests covering happy path, per-step compensation, and the full rollback scenario using mocked SDK clients.

**Step 7: Review**

```
> Review this implementation with the Firefly code reviewer
```

---

### Example 2: Adding Event Sourcing to an Existing Service

**Scenario:** You're evolving `core-wallet` from a CRUD model to event sourcing for full auditability.

**Step 1: Design the aggregate**

```
> I want to event-source the Wallet entity. Help me design the aggregate root
> with WalletCreated, FundsDeposited, FundsWithdrawn, and WalletFrozen events.
```

The `implement-event-sourcing` skill activates and produces:

```java
public class WalletAggregate extends AggregateRoot<UUID> {

    private BigDecimal balance;
    private WalletStatus status;

    // Command methods
    public Mono<Void> deposit(BigDecimal amount) {
        return apply(new FundsDeposited(getId(), amount, Instant.now()));
    }

    // Event handlers (state reconstruction)
    @EventHandler
    public void on(FundsDeposited event) {
        this.balance = this.balance.add(event.getAmount());
    }
}
```

**Step 2: Add projections for read models**

```
> Create a read-side projection that maintains the current wallet balance
> in R2DBC for fast queries
```

Both `implement-event-sourcing` (projection pattern) and `implement-reactive-r2dbc` (R2DBC entity) activate together.

**Step 3: Add snapshots for performance**

```
> Configure snapshotting every 100 events for the WalletAggregate
```

**Step 4: Create the migration**

```
/create-migration create_wallet_event_store
```

---

### Example 3: Implementing a Service Client with Caching and Resilience

**Scenario:** Your service needs to call `core-product-catalog` frequently, and you want to cache results with circuit breaker protection.

```
> I need to call the ProductCatalogService via REST. Cache the results for
> 5 minutes with Caffeine (L1) and Redis (L2). Add circuit breaker protection.
```

Three skills activate simultaneously:

- **`implement-service-client`** — generates the `RestClientBuilder` setup with OAuth2 token propagation
- **`implement-cache-strategy`** — configures multi-tier L1 (Caffeine, 5min TTL) + L2 (Redis, 15min TTL) caching on the query handler
- **`implement-resilience`** — wraps the REST call with circuit breaker (50% failure threshold) and retry (3 attempts, exponential backoff)

The resulting code uses `@InvalidateCacheOn(commands = {UpdateProductCommand.class})` to automatically evict the cache when products change.

---

### Example 4: Full TDD Workflow for a CQRS Handler

**Scenario:** You're implementing a `TransferFundsCommand` handler and want to follow TDD.

**Step 1: Generate the handler scaffold**

```
/create-handler command TransferFunds
```

**Step 2: Write the test first**

```
> Write a test for the TransferFundsCommandHandler that verifies:
> 1. Source account is debited
> 2. Destination account is credited
> 3. A FundsTransferred event is published
> 4. Unauthorized users get a 403
```

The `testing-cqrs-handlers` skill generates:

```java
@Test
void shouldTransferFunds() {
    // Arrange
    var cmd = new TransferFundsCommand(sourceId, destId, amount);
    var ctx = ExecutionContext.builder().userId(userId).roles(Set.of("TRANSFER")).build();

    // Act
    StepVerifier.create(handler.handle(cmd, ctx))
        .assertNext(result -> {
            assertThat(result.getSourceBalance()).isEqualTo(expectedSourceBalance);
            assertThat(result.getDestBalance()).isEqualTo(expectedDestBalance);
        })
        .verifyComplete();

    // Assert event published
    verify(eventPublisher).publish(argThat(event ->
        event instanceof FundsTransferred
        && ((FundsTransferred) event).getAmount().equals(amount)));
}

@Test
void shouldRejectUnauthorizedTransfer() {
    var ctx = ExecutionContext.builder().userId(userId).roles(Set.of("READ_ONLY")).build();

    StepVerifier.create(handler.handle(cmd, ctx))
        .expectError(AuthorizationException.class)
        .verify();
}
```

**Step 3: Implement the handler to pass the tests**

```
> Now implement the TransferFundsCommandHandler to pass these tests
```

The `implement-cqrs` skill guides the implementation with proper `@CommandHandlerComponent`, `@Authorize`, `@Validate`, and `ExecutionContext` usage.

---

### Example 5: Debugging a Failing Reactive Chain

**Scenario:** Your saga step is throwing `java.lang.IllegalStateException: Context is empty` and you don't know where.

```
> My RegisterOrderSaga step processPayment is failing with
> "Context is empty" — help me debug this reactive chain
```

The `debugging-reactive-chains` skill activates and guides you through:

1. **Adding checkpoints** — `Mono.checkpoint("processPayment-step")` to pinpoint the exact operator in the chain
2. **Enabling ReactorDebugAgent** — `ReactorDebugAgent.init()` in test setup for full assembly tracing
3. **Checking ExecutionContext propagation** — the most common cause: `contextWrite()` missing from a `.flatMap()` boundary
4. **Saga-specific patterns** — verifying `SagaContext` variables are set in upstream steps

---

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

The banking plugin adds 20 skills, 3 commands, and 2 agents covering core banking, lending, payments, customer management, PSD2/Open Banking, and enterprise integrations. It builds on top of the framework skills — install both for full coverage.

## Links

- [Firefly Framework GitHub Organization](https://github.com/fireflyframework)
- [Firefly Banking Platform Skills](https://github.com/firefly-oss/firefly-oss-claude-skills)
- [Apache 2.0 License](LICENSE)

## License

Copyright 2026 Firefly Software Solutions Inc

Licensed under the Apache License, Version 2.0. See [LICENSE](LICENSE) for details.
