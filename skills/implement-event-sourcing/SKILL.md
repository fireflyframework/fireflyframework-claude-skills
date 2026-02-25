---
name: implement-event-sourcing
description: Use when implementing event-sourced aggregates in a Firefly Framework service — covers AggregateRoot, reactive EventStore over R2DBC, optimistic concurrency, snapshots, transactional outbox, projections, event upcasting, and multi-tenancy
---

# Implement Event Sourcing

## When to Use

Use this skill when the domain requires a full audit trail of state changes, temporal queries (time travel), or complex event-driven projections. Event sourcing stores every state change as an immutable event, making the event log the single source of truth.

- **AggregateRoot** enforces business invariants and produces events via `applyChange(Event)`.
- **EventStore** persists events reactively over R2DBC with optimistic concurrency control.
- **SnapshotStore** caches aggregate state at periodic versions to avoid replaying thousands of events.
- **Transactional Outbox** guarantees at-least-once delivery of events to external systems (Kafka, RabbitMQ).
- **ProjectionService** builds eventually-consistent read models from the event stream.
- **EventUpcaster** handles event schema evolution when event structures change over time.
- **TenantContext** provides tenant isolation in multi-tenant deployments through Reactor context propagation.

All APIs are reactive (`Mono`/`Flux` from Project Reactor). Never call `.block()` in production code.

## Event Sourcing Flow

### Step 1 -- Define Domain Events

Implement `org.fireflyframework.eventsourcing.domain.Event` or extend `org.fireflyframework.eventsourcing.domain.AbstractDomainEvent`. Annotate with `@DomainEvent("aggregate.action")` to set the event type identifier. The `@DomainEvent` annotation also serves as `@JsonTypeName` for polymorphic serialization.

```java
package com.example.account.events;

import org.fireflyframework.eventsourcing.annotation.DomainEvent;
import org.fireflyframework.eventsourcing.domain.AbstractDomainEvent;
import lombok.AllArgsConstructor;
import lombok.Getter;
import lombok.NoArgsConstructor;
import lombok.experimental.SuperBuilder;

import java.math.BigDecimal;
import java.util.UUID;

@DomainEvent("account.opened")
@SuperBuilder
@Getter
@NoArgsConstructor
@AllArgsConstructor
public class AccountOpenedEvent extends AbstractDomainEvent {
    private String accountNumber;
    private String accountType;
    private UUID customerId;
    private BigDecimal initialDeposit;
    private String currency;
}
```

`AbstractDomainEvent` provides `aggregateId`, `eventTimestamp`, `metadata`, and `eventVersion` out of the box. The builder supports fluent metadata via `correlationId(...)`, `causationId(...)`, `userId(...)`, and `source(...)`.

Follow the same pattern for all events: `@DomainEvent("money.deposited")`, `@DomainEvent("money.withdrawn")`, etc. Each event class extends `AbstractDomainEvent` with `@SuperBuilder @Getter @NoArgsConstructor @AllArgsConstructor`.

**Event naming conventions**: Use dot notation in past tense (`account.opened`, `money.withdrawn`). The value must be stable -- changing it breaks deserialization of historical events. Java Records also work: `@DomainEvent("account.opened") public record AccountOpenedEvent(UUID aggregateId, String accountNumber) implements Event {}`.

### Step 2 -- Implement the AggregateRoot

Extend `org.fireflyframework.eventsourcing.aggregate.AggregateRoot`. The aggregate has two responsibilities: (1) validate business rules in public command methods, and (2) update state in private `on(EventType)` handler methods. Never modify state directly -- always go through `applyChange(Event)`.

```java
package com.example.account;

import org.fireflyframework.eventsourcing.aggregate.AggregateRoot;
import com.example.account.events.*;
import lombok.Getter;

import java.math.BigDecimal;
import java.time.Instant;
import java.util.UUID;

@Getter
public class AccountLedger extends AggregateRoot {

    private String accountNumber;
    private String accountType;
    private UUID customerId;
    private BigDecimal balance;
    private String currency;
    private boolean frozen;
    private boolean closed;

    // Constructor for loading from event store (reconstruction)
    public AccountLedger(UUID id) {
        super(id, "AccountLedger");
        this.balance = BigDecimal.ZERO;
    }

    // Constructor for creating a new aggregate (command)
    public AccountLedger(UUID id, String accountNumber, String accountType,
                         UUID customerId, BigDecimal initialDeposit, String currency) {
        super(id, "AccountLedger");

        // Validate business rules BEFORE generating events
        if (initialDeposit.compareTo(BigDecimal.ZERO) < 0) {
            throw new IllegalArgumentException("Initial deposit cannot be negative");
        }

        // Generate the creation event
        applyChange(AccountOpenedEvent.builder()
                .aggregateId(id)
                .accountNumber(accountNumber)
                .accountType(accountType)
                .customerId(customerId)
                .initialDeposit(initialDeposit)
                .currency(currency)
                .build());
    }

    // Command method -- validates then generates events
    public void deposit(BigDecimal amount, String source, String reference, String userId) {
        if (closed) {
            throw new IllegalStateException("Cannot deposit to closed account");
        }
        if (amount.compareTo(BigDecimal.ZERO) <= 0) {
            throw new IllegalArgumentException("Deposit amount must be positive");
        }

        applyChange(MoneyDepositedEvent.builder()
                .aggregateId(getId())
                .amount(amount)
                .source(source)
                .reference(reference)
                .depositedBy(userId)
                .build());
    }

    public void withdraw(BigDecimal amount, String destination, String reference, String userId) {
        if (closed) {
            throw new IllegalStateException("Cannot withdraw from closed account");
        }
        if (frozen) {
            throw new IllegalStateException("Cannot withdraw from frozen account");
        }
        if (amount.compareTo(BigDecimal.ZERO) <= 0) {
            throw new IllegalArgumentException("Withdrawal amount must be positive");
        }
        if (balance.compareTo(amount) < 0) {
            throw new IllegalStateException("Insufficient funds");
        }

        applyChange(MoneyWithdrawnEvent.builder()
                .aggregateId(getId())
                .amount(amount)
                .destination(destination)
                .reference(reference)
                .withdrawnBy(userId)
                .build());
    }

    // Event handlers -- update state, NO validation
    private void on(AccountOpenedEvent event) {
        this.accountNumber = event.getAccountNumber();
        this.accountType = event.getAccountType();
        this.customerId = event.getCustomerId();
        this.balance = event.getInitialDeposit();
        this.currency = event.getCurrency();
        this.frozen = false;
        this.closed = false;
    }

    private void on(MoneyDepositedEvent event) {
        this.balance = this.balance.add(event.getAmount());
    }

    private void on(MoneyWithdrawnEvent event) {
        this.balance = this.balance.subtract(event.getAmount());
    }
}
```

**Key rules**:
- The `AggregateRoot(UUID id, String aggregateType)` constructor requires a non-null ID and non-empty type.
- `applyChange(Event)` adds the event to `uncommittedEvents`, invokes the matching `on(EventType)` handler via reflection, and increments the version.
- `loadFromHistory(List<StoredEventEnvelope>)` replays historical events without adding them to uncommitted events.
- `markEventsAsCommitted()` clears uncommitted events after persistence succeeds.
- Event handlers are found by reflection: a method named `on` taking a single event-type parameter (can be private).

### Step 3 -- Persist Events via the EventStore

Inject `org.fireflyframework.eventsourcing.store.EventStore` (auto-configured as `R2dbcEventStore`). The store provides reactive `Mono`/`Flux` operations for appending and loading events.

```java
package com.example.account;

import org.fireflyframework.eventsourcing.annotation.EventSourcingTransactional;
import org.fireflyframework.eventsourcing.domain.EventStream;
import org.fireflyframework.eventsourcing.store.EventStore;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;
import reactor.core.publisher.Mono;

import java.math.BigDecimal;
import java.util.UUID;

@Service
@RequiredArgsConstructor
public class AccountLedgerService {

    private final EventStore eventStore;

    // Create a new aggregate
    @EventSourcingTransactional
    public Mono<AccountLedger> openAccount(String accountNumber, String accountType,
                                           UUID customerId, BigDecimal initialDeposit,
                                           String currency) {
        UUID accountId = UUID.randomUUID();
        return Mono.fromCallable(() -> new AccountLedger(
                    accountId, accountNumber, accountType,
                    customerId, initialDeposit, currency))
                .flatMap(account -> eventStore.appendEvents(
                        accountId,
                        "AccountLedger",
                        account.getUncommittedEvents(),
                        0L  // expectedVersion: -1 means new, 0 means first version
                    )
                    .doOnSuccess(stream -> account.markEventsAsCommitted())
                    .thenReturn(account));
    }

    // Load, modify, and save
    @EventSourcingTransactional(retryOnConcurrencyConflict = true, maxRetries = 3)
    public Mono<AccountLedger> deposit(UUID accountId, BigDecimal amount,
                                       String source, String reference, String userId) {
        return loadAccount(accountId)
                .doOnNext(account -> account.deposit(amount, source, reference, userId))
                .flatMap(this::saveAccount);
    }

    private Mono<AccountLedger> loadAccount(UUID accountId) {
        return eventStore.loadEventStream(accountId, "AccountLedger")
                .map(stream -> {
                    AccountLedger account = new AccountLedger(accountId);
                    account.loadFromHistory(stream.getEvents());
                    return account;
                });
    }

    private Mono<AccountLedger> saveAccount(AccountLedger account) {
        return eventStore.appendEvents(
                    account.getId(),
                    "AccountLedger",
                    account.getUncommittedEvents(),
                    account.getCurrentVersion() - account.getUncommittedEventCount()
                )
                .doOnSuccess(stream -> account.markEventsAsCommitted())
                .thenReturn(account);
    }
}
```

**EventStore API** (all reactive `Mono`/`Flux`):
- `appendEvents(aggregateId, aggregateType, events, expectedVersion)` -- atomic append. Throws `ConcurrencyException` on version mismatch. Overload accepts `Map<String, Object> metadata`.
- `loadEventStream(aggregateId, aggregateType)` -- load all events. Overloads accept `fromVersion` and `fromVersion, toVersion`.
- `streamAllEvents()` / `streamAllEvents(fromSequence)` -- global ordering for projections.
- `streamEventsByType(List<String>)`, `streamEventsByAggregateType(List<String>)`, `streamEventsByTimeRange(from, to)` -- filtered streams.
- `getCurrentGlobalSequence()` -- latest global sequence number.

### Step 4 -- Use @EventSourcingTransactional

The `@EventSourcingTransactional` annotation (`org.fireflyframework.eventsourcing.annotation.EventSourcingTransactional`) wraps the method in a reactive transaction with event sourcing guarantees: atomic event persistence, automatic event publishing via the transactional outbox, and optional concurrency conflict retry.

```java
// Basic -- atomic save + auto-publish
@EventSourcingTransactional
public Mono<AccountLedger> openAccount(...) { ... }

// With concurrency retry -- reloads and re-executes on ConcurrencyException
@EventSourcingTransactional(retryOnConcurrencyConflict = true, maxRetries = 3, retryDelay = 100)
public Mono<AccountLedger> withdraw(...) { ... }

// Read-only -- optimized, no event publishing
@EventSourcingTransactional(readOnly = true, publishEvents = false)
public Mono<AccountLedger> getAccount(...) { ... }
```

**Annotation attributes**:
- `publishEvents` (default `true`) -- publish events to external systems after commit.
- `retryOnConcurrencyConflict` (default `false`) -- auto-retry on `ConcurrencyException`.
- `maxRetries` (default `3`) -- maximum retry attempts.
- `retryDelay` (default `100`) -- base delay in milliseconds with exponential backoff.
- `propagation` -- `REQUIRED` (default), `REQUIRES_NEW`, `MANDATORY`, `NEVER`, `SUPPORTS`, `NOT_SUPPORTED`.
- `isolation` -- `DEFAULT`, `READ_UNCOMMITTED`, `READ_COMMITTED`, `REPEATABLE_READ`, `SERIALIZABLE`.
- `timeout` -- transaction timeout in seconds (-1 for no timeout).
- `readOnly` -- optimize for read-only operations.

## Optimistic Concurrency Control

The `R2dbcEventStore` enforces optimistic concurrency by checking the aggregate version before appending events. Each event gets a sequential `aggregate_version` within its stream, and the database has a unique constraint on `(aggregate_id, aggregate_version)`.

When the expected version does not match the current version, a `ConcurrencyException` is thrown:

```java
import org.fireflyframework.eventsourcing.store.ConcurrencyException;

// Manual retry pattern
public Mono<AccountLedger> withdrawWithRetry(UUID accountId, BigDecimal amount,
                                              String dest, String ref, String userId) {
    return Mono.defer(() -> loadAccount(accountId)
            .doOnNext(account -> account.withdraw(amount, dest, ref, userId))
            .flatMap(this::saveAccount))
        .retry(3, throwable -> throwable instanceof ConcurrencyException);
}
```

`ConcurrencyException` exposes `getAggregateId()`, `getAggregateType()`, `getExpectedVersion()`, and `getActualVersion()` for diagnostics.

## Snapshots

### Step 5 -- Define a Snapshot

Extend `org.fireflyframework.eventsourcing.snapshot.AbstractSnapshot` and include all aggregate state fields needed for reconstruction.

```java
package com.example.account.snapshot;

import org.fireflyframework.eventsourcing.snapshot.AbstractSnapshot;
import lombok.Getter;

import java.math.BigDecimal;
import java.time.Instant;
import java.util.UUID;

@Getter
public class AccountLedgerSnapshot extends AbstractSnapshot {

    private final String accountNumber;
    private final String accountType;
    private final UUID customerId;
    private final BigDecimal balance;
    private final String currency;
    private final boolean frozen;
    private final boolean closed;

    public AccountLedgerSnapshot(UUID aggregateId, long version, Instant createdAt,
                                 String accountNumber, String accountType, UUID customerId,
                                 BigDecimal balance, String currency,
                                 boolean frozen, boolean closed) {
        super(aggregateId, version, createdAt);
        this.accountNumber = accountNumber;
        this.accountType = accountType;
        this.customerId = customerId;
        this.balance = balance;
        this.currency = currency;
        this.frozen = frozen;
        this.closed = closed;
    }

    @Override
    public String getSnapshotType() {
        return "AccountLedger";
    }
}
```

`AbstractSnapshot` provides `getAggregateId()`, `getVersion()`, `getCreatedAt()`, `getReason()`, `getSizeBytes()`, and utility methods like `isOlderThan(int days)` and `isNewerThan(long version)`.

### Step 6 -- Add Snapshot Support to the Aggregate

Add a factory method and a creation method to the aggregate:

```java
// In AccountLedger.java

public AccountLedgerSnapshot createSnapshot() {
    return new AccountLedgerSnapshot(
            getId(), getCurrentVersion(), Instant.now(),
            accountNumber, accountType, customerId,
            balance, currency, frozen, closed);
}

public static AccountLedger fromSnapshot(AccountLedgerSnapshot snapshot) {
    AccountLedger account = new AccountLedger(snapshot.getAggregateId());
    account.accountNumber = snapshot.getAccountNumber();
    account.accountType = snapshot.getAccountType();
    account.customerId = snapshot.getCustomerId();
    account.balance = snapshot.getBalance();
    account.currency = snapshot.getCurrency();
    account.frozen = snapshot.isFrozen();
    account.closed = snapshot.isClosed();
    account.setCurrentVersion(snapshot.getVersion());
    return account;
}
```

### Step 7 -- Load with Snapshot Optimization

Update the service to try snapshot-first loading. Inject `SnapshotStore` alongside `EventStore`:

```java
private final EventStore eventStore;
private final SnapshotStore snapshotStore;

private Mono<AccountLedger> loadAccount(UUID accountId) {
    return snapshotStore.loadLatestSnapshot(accountId, "AccountLedger")
            .cast(AccountLedgerSnapshot.class)
            .flatMap(snapshot -> loadFromSnapshot(accountId, snapshot))
            .switchIfEmpty(loadFromEvents(accountId));
}

private Mono<AccountLedger> loadFromSnapshot(UUID accountId, AccountLedgerSnapshot snapshot) {
    // Load only events AFTER the snapshot version
    return eventStore.loadEventStream(accountId, "AccountLedger", snapshot.getVersion())
            .map(stream -> {
                AccountLedger account = AccountLedger.fromSnapshot(snapshot);
                account.loadFromHistory(stream.getEvents());
                return account;
            });
}

private Mono<AccountLedger> loadFromEvents(UUID accountId) {
    return eventStore.loadEventStream(accountId, "AccountLedger")
            .map(stream -> {
                AccountLedger account = new AccountLedger(accountId);
                account.loadFromHistory(stream.getEvents());
                return account;
            });
}
```

### Automatic Snapshot Trigger

The `SnapshotTrigger` class creates snapshots automatically every N events:

```java
// Auto-configured via properties; to use programmatically:
SnapshotTrigger trigger = new SnapshotTrigger(snapshotStore, 50);

// Fire-and-forget (async, failures logged but not propagated)
trigger.maybeTrigger(account.createSnapshot());

// Reactive variant (chains into pipeline)
trigger.maybeTriggerReactive(account.createSnapshot())
    .then(/* continue pipeline */);
```

**SnapshotStore API** (all reactive):
- `saveSnapshot(Snapshot)` -- save or replace a snapshot.
- `loadLatestSnapshot(UUID aggregateId, String snapshotType)` -- load the most recent snapshot.
- `loadSnapshotAtOrBeforeVersion(UUID, String, long maxVersion)` -- load snapshot at or before a version.
- `deleteAllSnapshots(UUID, String)` -- remove all snapshots for an aggregate.
- `keepLatestSnapshots(UUID, String, int keepCount)` -- retain only the N most recent snapshots.
- `deleteSnapshotsOlderThan(Instant)` / `deleteSnapshotsOlderThan(int days)` -- cleanup old snapshots.

## Transactional Outbox

The outbox pattern ensures events are published reliably to external systems. Events are written to the outbox table in the same database transaction as the event store write. A background processor polls and publishes them.

The `R2dbcEventStore` automatically saves events to the outbox when `EventOutboxService` is on the classpath and configured. No additional code is needed in the aggregate service -- the `@EventSourcingTransactional` annotation handles it.

### Outbox Processing

`EventOutboxProcessor` runs scheduled tasks to poll and publish pending outbox entries:

- **Pending entries**: polled every 5 seconds.
- **Retry entries**: polled every 30 seconds (exponential backoff).
- **Cleanup**: completed entries older than 7 days removed every hour.
- **Statistics**: logged every 5 minutes.

Outbox entry statuses: `PENDING` -> `PROCESSING` -> `COMPLETED` or `FAILED`. After `maxRetries` failures, entries enter the dead letter queue.

**EventOutboxService API**:
- `saveToOutbox(StoredEventEnvelope)` -- save to outbox within the current transaction.
- `saveToOutbox(StoredEventEnvelope, int priority, int maxRetries)` -- save with custom priority (1=highest, 10=lowest) and retry limit.
- `processPendingEntries(int batchSize)` -- process a batch of pending entries.
- `processRetryEntries(int batchSize)` -- process entries ready for retry.
- `getDeadLetterEntries()` -- retrieve permanently failed entries.
- `cleanupCompletedEntries(int olderThanDays)` -- purge old completed entries.
- `getStatistics()` -- returns `OutboxStatistics(pendingCount, processingCount, completedCount, failedCount, deadLetterCount)`.

### Event Publishing Integration

`EventSourcingPublisher` bridges the event sourcing module to the EDA infrastructure (`fireflyframework-eda`). It publishes domain events to Kafka, RabbitMQ, or Spring Application Events depending on configuration.

Destination routing: `{prefix}.{eventType}` by default (e.g., `events.account.opened`). Custom mappings can be defined in configuration.

## Projections

### Step 8 -- Build a Read Model Projection

Extend `org.fireflyframework.eventsourcing.projection.ProjectionService<T>` to build eventually-consistent read models from the event stream.

```java
package com.example.account.projection;

import org.fireflyframework.eventsourcing.domain.StoredEventEnvelope;
import org.fireflyframework.eventsourcing.projection.ProjectionService;
import org.fireflyframework.eventsourcing.store.EventStore;
import com.example.account.events.*;
import io.micrometer.core.instrument.MeterRegistry;
import org.springframework.r2dbc.core.DatabaseClient;
import org.springframework.stereotype.Service;
import reactor.core.publisher.Mono;
import java.time.Instant;

@Service
public class AccountLedgerProjectionService extends ProjectionService<AccountLedgerReadModel> {

    private final AccountLedgerRepository repository;
    private final DatabaseClient databaseClient;
    private final EventStore eventStore;

    public AccountLedgerProjectionService(AccountLedgerRepository repository,
                                          DatabaseClient databaseClient,
                                          EventStore eventStore,
                                          MeterRegistry meterRegistry) {
        super(meterRegistry);
        this.repository = repository;
        this.databaseClient = databaseClient;
        this.eventStore = eventStore;
    }

    @Override public String getProjectionName() { return "account-ledger-projection"; }

    @Override
    public Mono<Void> handleEvent(StoredEventEnvelope envelope) {
        return Mono.defer(() -> {
            if (envelope.getEvent() instanceof AccountOpenedEvent event) {
                return repository.save(AccountLedgerReadModel.builder()
                        .accountId(event.getAggregateId())
                        .accountNumber(event.getAccountNumber())
                        .balance(event.getInitialDeposit())
                        .status("ACTIVE")
                        .version(envelope.getAggregateVersion())
                        .build()).then();
            } else if (envelope.getEvent() instanceof MoneyDepositedEvent event) {
                return repository.findById(event.getAggregateId())
                        .flatMap(rm -> {
                            rm.setBalance(rm.getBalance().add(event.getAmount()));
                            rm.setVersion(envelope.getAggregateVersion());
                            return repository.save(rm);
                        }).then();
            }
            return Mono.empty();
        });
    }

    @Override
    public Mono<Long> getCurrentPosition() {
        return databaseClient.sql(
                "SELECT position FROM projection_positions WHERE projection_name = :name")
                .bind("name", getProjectionName())
                .map(row -> row.get("position", Long.class)).one().defaultIfEmpty(0L);
    }

    @Override
    public Mono<Void> updatePosition(long position) {
        return databaseClient.sql("""
                INSERT INTO projection_positions (projection_name, position, last_updated)
                VALUES (:name, :position, :lastUpdated)
                ON CONFLICT (projection_name)
                DO UPDATE SET position = :position, last_updated = :lastUpdated""")
                .bind("name", getProjectionName())
                .bind("position", position)
                .bind("lastUpdated", Instant.now()).then();
    }

    @Override
    protected Mono<Void> clearProjectionData() {
        return databaseClient.sql("DELETE FROM account_ledger_read_model")
                .fetch().rowsUpdated().then();
    }

    @Override
    protected Mono<Long> getLatestGlobalSequenceFromEventStore() {
        return eventStore.getCurrentGlobalSequence();
    }
}
```

**ProjectionService abstract methods** (you implement): `handleEvent(StoredEventEnvelope)`, `getCurrentPosition()`, `updatePosition(long)`, `getProjectionName()`, `clearProjectionData()`, `getLatestGlobalSequenceFromEventStore()`.

**Provided methods**: `processBatch(Flux)` -- batch processing with atomic position update. `processIndividually(Flux)` -- per-event position update. `resetProjection()` -- clear data and reset to zero. `getHealth(long)` -- returns `ProjectionHealth` with lag and status. `getMaxAllowedLag()` -- override for custom threshold (default 1000).

## Event Upcasting

When event schemas evolve, implement `org.fireflyframework.eventsourcing.upcasting.EventUpcaster` to transform old event versions to new ones at read time (events in the store are never modified).

```java
package com.example.account.upcasting;

import org.fireflyframework.eventsourcing.domain.Event;
import org.fireflyframework.eventsourcing.upcasting.EventUpcaster;
import com.example.account.events.AccountOpenedEvent;

public class AccountOpenedV1ToV2Upcaster implements EventUpcaster {

    @Override
    public boolean canUpcast(String eventType, int eventVersion) {
        return "account.opened".equals(eventType) && eventVersion == 1;
    }

    @Override
    public Event upcast(Event event) {
        // Transform V1 event to V2 -- add default currency
        AccountOpenedEvent v1 = (AccountOpenedEvent) event;
        return AccountOpenedEvent.builder()
                .aggregateId(v1.getAggregateId())
                .accountNumber(v1.getAccountNumber())
                .accountType(v1.getAccountType())
                .customerId(v1.getCustomerId())
                .initialDeposit(v1.getInitialDeposit())
                .currency("USD")  // New field with default
                .build();
    }

    @Override
    public int getTargetVersion() {
        return 2;
    }

    @Override
    public int getPriority() {
        return 10; // Higher priority runs first
    }
}
```

Register upcasters as Spring beans. The `EventUpcastingService` auto-discovers them, sorts by priority (descending), and applies them in chain: if a V1->V2 upcaster and a V2->V3 upcaster both exist, an event at V1 will be upcasted through both to reach V3.

## Multi-Tenancy

Use `org.fireflyframework.eventsourcing.multitenancy.TenantContext` to propagate tenant information through the reactive pipeline via Reactor context:

```java
import org.fireflyframework.eventsourcing.multitenancy.TenantContext;

// Set tenant context on a reactive chain
return eventStore.appendEvents(accountId, "AccountLedger", events, expectedVersion)
    .contextWrite(TenantContext.withTenantId("tenant-123"));

// Read current tenant ID inside a reactive chain
TenantContext.getCurrentTenantId()
    .flatMap(tenantId -> eventStore.loadEventStream(accountId, "AccountLedger"));

// Check if tenant is set
TenantContext.hasTenantId()
    .filter(Boolean::booleanValue)
    .flatMap(hasIt -> /* proceed */);

// Clear tenant context
.contextWrite(TenantContext.clear());
```

The `R2dbcEventStore` automatically reads the tenant ID from the context and stores it in the `tenant_id` column of the events table when the multi-tenancy auto-configuration is active.

## Time Travel

Event sourcing enables querying aggregate state at any point in time:

```java
public Mono<AccountLedger> getAccountAtTime(UUID accountId, Instant pointInTime) {
    return eventStore.loadEventStream(accountId, "AccountLedger")
            .map(stream -> {
                AccountLedger account = new AccountLedger(accountId);
                List<StoredEventEnvelope> filtered = stream.getEvents().stream()
                        .filter(e -> !e.getCreatedAt().isAfter(pointInTime))
                        .toList();
                account.loadFromHistory(filtered);
                return account;
            });
}
```

## Configuration

All properties are under `firefly.eventsourcing` (class: `EventSourcingProperties`):

```yaml
firefly:
  eventsourcing:
    enabled: true
    store:
      type: r2dbc                    # r2dbc (default)
      batch-size: 100
      connection-timeout: 30s
      query-timeout: 30s
      max-events-per-load: 1000
    snapshot:
      enabled: true
      threshold: 50                  # Snapshot every N events
      keep-count: 3                  # Snapshots to keep per aggregate
      max-age: 30d
    publisher:
      enabled: true
      type: AUTO                     # AUTO, KAFKA, RABBITMQ, SPRING
      destination-prefix: events     # Topic prefix (events.account.opened)
      async: true
      destination-mappings:          # Custom routing per event type
        account.opened: account-events
      retry:
        enabled: true
        max-attempts: 3
        initial-delay: 1s
        backoff-multiplier: 2.0
    performance:
      metrics-enabled: true
      health-checks-enabled: true
      tracing-enabled: true
      circuit-breaker:
        enabled: false
```

## Database Schema

The `R2dbcEventStore` requires the following PostgreSQL tables (JSONB columns; use TEXT for other databases):

```sql
CREATE TABLE events (
    event_id UUID PRIMARY KEY,
    aggregate_id UUID NOT NULL,
    aggregate_type VARCHAR(255) NOT NULL,
    aggregate_version BIGINT NOT NULL,
    global_sequence BIGSERIAL UNIQUE,
    event_type VARCHAR(255) NOT NULL,
    event_data JSONB NOT NULL,
    metadata JSONB,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    correlation_id VARCHAR(255),
    tenant_id VARCHAR(255),
    created_by VARCHAR(255),
    UNIQUE(aggregate_id, aggregate_version)
);
CREATE INDEX idx_events_aggregate ON events(aggregate_id, aggregate_type);
CREATE INDEX idx_events_global_sequence ON events(global_sequence);
CREATE INDEX idx_events_type ON events(event_type);

CREATE TABLE snapshots (
    snapshot_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    aggregate_id UUID NOT NULL,
    snapshot_type VARCHAR(255) NOT NULL,
    version BIGINT NOT NULL,
    snapshot_data JSONB NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
CREATE INDEX idx_snapshots_aggregate ON snapshots(aggregate_id, snapshot_type);

CREATE TABLE event_outbox (
    outbox_id UUID PRIMARY KEY,
    aggregate_id UUID NOT NULL,
    aggregate_type VARCHAR(255) NOT NULL,
    event_type VARCHAR(255) NOT NULL,
    event_data JSONB NOT NULL,
    metadata JSONB,
    status VARCHAR(50) NOT NULL DEFAULT 'PENDING',
    priority INTEGER DEFAULT 5,
    retry_count INTEGER DEFAULT 0,
    max_retries INTEGER DEFAULT 3,
    last_error TEXT,
    next_retry_at TIMESTAMP WITH TIME ZONE,
    processed_at TIMESTAMP WITH TIME ZONE,
    partition_key VARCHAR(255),
    correlation_id VARCHAR(255),
    tenant_id VARCHAR(255),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE
);
CREATE INDEX idx_outbox_status ON event_outbox(status);
CREATE INDEX idx_outbox_retry ON event_outbox(status, next_retry_at);

CREATE TABLE projection_positions (
    projection_name VARCHAR(255) PRIMARY KEY,
    position BIGINT NOT NULL DEFAULT 0,
    last_updated TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
```

## Package Structure

```
com.example.myservice/
  aggregate/
    AccountLedger.java                     # extends AggregateRoot
  events/
    AccountOpenedEvent.java                # @DomainEvent + extends AbstractDomainEvent
    MoneyDepositedEvent.java
    MoneyWithdrawnEvent.java
    AccountFrozenEvent.java
    AccountClosedEvent.java
  snapshot/
    AccountLedgerSnapshot.java             # extends AbstractSnapshot
  projection/
    AccountLedgerProjectionService.java    # extends ProjectionService
    AccountLedgerReadModel.java            # read model entity
    AccountLedgerRepository.java           # R2DBC repository
  upcasting/
    AccountOpenedV1ToV2Upcaster.java       # implements EventUpcaster
  service/
    AccountLedgerService.java              # orchestrates load/save via EventStore
  exception/
    AccountClosedException.java
    InsufficientFundsException.java
```

## Common Mistakes

1. **Modifying aggregate state directly** -- Never set fields outside of `on(EventType)` handlers. All state changes must go through `applyChange(Event)` so they are captured in the event stream. Direct mutation means state diverges from events.

2. **Validating in event handlers** -- Event handlers (`on(...)` methods) must never throw exceptions or validate. They replay historical events which already passed validation. Put all validation in command methods (the public methods that call `applyChange`).

3. **Forgetting `markEventsAsCommitted()`** -- After `eventStore.appendEvents()` succeeds, call `account.markEventsAsCommitted()`. Without this, events accumulate in the uncommitted list and get re-persisted on the next save.

4. **Wrong `expectedVersion` calculation** -- When saving, the expected version is the version of the aggregate when it was loaded, not the current version after applying new events. Use `account.getCurrentVersion() - account.getUncommittedEventCount()`.

5. **Blocking in reactive chains** -- Never call `.block()`, `Thread.sleep()`, or synchronous I/O within reactive pipelines. The `EventStore`, `SnapshotStore`, and `ProjectionService` APIs are fully reactive. Use `flatMap`, `map`, `zipWith`, and other reactive operators.

6. **Changing `@DomainEvent` values** -- The string value in `@DomainEvent("account.opened")` is persisted in the database and used for deserialization. Changing it breaks loading of historical events. Use event upcasting instead.

7. **Fat aggregates** -- Keep aggregates focused on a single consistency boundary. If an aggregate handles too many concerns, split it into smaller aggregates communicating via eventual consistency and domain events.

8. **Loading aggregates inside aggregates** -- An aggregate must never load another aggregate. Cross-aggregate interactions should use sagas, process managers, or event-driven choreography.

9. **Skipping snapshots for high-volume aggregates** -- If an aggregate can accumulate hundreds or thousands of events, configure snapshots. Without them, loading the aggregate replays every event from the beginning, causing latency spikes.

10. **Not handling `ConcurrencyException`** -- In high-contention scenarios, concurrent writes to the same aggregate will cause version conflicts. Either use `@EventSourcingTransactional(retryOnConcurrencyConflict = true)` or implement explicit retry logic.

11. **Ignoring projection lag** -- Projections are eventually consistent. Override `getMaxAllowedLag()` in your `ProjectionService` subclass and monitor `ProjectionHealth` to detect when projections fall behind.

12. **Publishing events before commit** -- Events must be published only after the transaction commits. The transactional outbox pattern (automatic with `@EventSourcingTransactional`) handles this correctly. Never publish events directly from within the aggregate or the `appendEvents` call.
