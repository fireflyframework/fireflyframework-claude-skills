---
name: implement-eda
description: Use when implementing event-driven architecture in a Firefly Framework service — covers event publishing and consumption over Kafka/RabbitMQ with unified abstraction, JSON/Avro/Protobuf serialization, DLQ, circuit breakers, and event filtering
---

# Implement Event-Driven Architecture

## When to Use

Use this skill when you need to publish or consume events across services using Kafka, RabbitMQ, or in-memory Spring Events. The Firefly EDA module provides a unified abstraction layer so your application code stays independent of the underlying transport.

- **Publishing** -- Use `EventPublisher` (programmatic) or `@EventPublisher` / `@PublishResult` (declarative) to send events. The publisher auto-selects the transport based on classpath and configuration.
- **Consuming** -- Use `@EventListener` (declarative) or `EventConsumer` (programmatic) to receive events. Listeners are auto-discovered at startup and matched to incoming events by destination, event type, and consumer type.
- **CQRS Integration** -- Use `@PublishDomainEvent` on command handlers to automatically publish command results as domain events via the EDA module.

## Architecture Overview

The EDA module is organized around these core abstractions:

| Abstraction | Package | Purpose |
|---|---|---|
| `EventPublisher` | `org.fireflyframework.eda.publisher` | Unified reactive publishing interface |
| `EventConsumer` | `org.fireflyframework.eda.consumer` | Unified reactive consumption interface |
| `EventEnvelope` | `org.fireflyframework.eda.event` | Standard event wrapper with metadata, headers, and ack callbacks |
| `EventPublisherFactory` | `org.fireflyframework.eda.publisher` | Factory for obtaining publishers by type and connection |
| `EventListenerProcessor` | `org.fireflyframework.eda.listener` | Discovers and dispatches to `@EventListener` methods |
| `MessageSerializer` | `org.fireflyframework.eda.serialization` | Pluggable serialization (JSON, Avro, Protobuf) |
| `EventFilter` | `org.fireflyframework.eda.filter` | Composable event filtering |
| `ResilientEventPublisher` | `org.fireflyframework.eda.resilience` | Circuit breaker, retry, and rate limiting wrapper |

Transport selection priority when `PublisherType.AUTO` is used: KAFKA then RABBITMQ then APPLICATION_EVENT.

## Publishing Events

### Option A -- Programmatic Publishing with EventPublisherFactory

Inject `EventPublisherFactory` and obtain a publisher for a specific type or let the framework auto-select.

```java
package com.example.order.service;

import org.fireflyframework.eda.annotation.PublisherType;
import org.fireflyframework.eda.publisher.EventPublisher;
import org.fireflyframework.eda.publisher.EventPublisherFactory;
import reactor.core.publisher.Mono;

import java.util.Map;

@Service
public class OrderEventService {

    private final EventPublisherFactory publisherFactory;

    public OrderEventService(EventPublisherFactory publisherFactory) {
        this.publisherFactory = publisherFactory;
    }

    public Mono<Void> publishOrderCreated(OrderCreatedEvent event) {
        EventPublisher publisher = publisherFactory.getPublisher(PublisherType.KAFKA);
        return publisher.publish(event, "order-events", Map.of(
            "eventType", "order.created",
            "partition_key", event.getOrderId()
        ));
    }

    public Mono<Void> publishToAutoSelected(Object event, String destination) {
        // AUTO selects KAFKA -> RABBITMQ -> APPLICATION_EVENT based on availability
        EventPublisher publisher = publisherFactory.getPublisher(PublisherType.AUTO);
        return publisher.publish(event, destination);
    }
}
```

**Dynamic destination override** -- Use `getPublisherWithDestination()` to create a publisher whose default destination differs from configuration:

```java
EventPublisher tenantPublisher = publisherFactory.getPublisherWithDestination(
    PublisherType.KAFKA, "tenant-42-events"
);
tenantPublisher.publish(event, null); // publishes to "tenant-42-events"
```

**Named connections** -- Use `getPublisher(PublisherType.KAFKA, "analytics")` to select a specific named connection configured under `firefly.eda.publishers.kafka.analytics`.

### Option B -- Declarative Publishing with @EventPublisher

Annotate a method with `@EventPublisher` to automatically publish its parameters as events. The annotation is processed by `EventPublisherAspect`.

```java
package com.example.user.service;

import org.fireflyframework.eda.annotation.EventPublisher;
import org.fireflyframework.eda.annotation.PublisherType;
import reactor.core.publisher.Mono;

@Service
public class UserService {

    @EventPublisher(
        publisherType = PublisherType.KAFKA,
        destination = "user-commands",
        eventType = "user.create.command",
        parameterIndex = 0,
        timing = EventPublisher.PublishTiming.BEFORE
    )
    public Mono<User> createUser(CreateUserCommand command) {
        return userRepository.save(command.toUser());
    }
}
```

**Annotation attributes**:

| Attribute | Default | Description |
|---|---|---|
| `publisherType` | `AUTO` | Transport: `KAFKA`, `RABBITMQ`, `APPLICATION_EVENT`, `AUTO`, `NOOP` |
| `destination` | `""` | Topic, queue, or exchange. Supports SpEL: `"#{#param0.tenantId}-events"` |
| `eventType` | `""` | Routing key for consumer-side filtering |
| `connectionId` | `"default"` | Named connection ID |
| `parameterIndex` | `-1` | Which parameter to publish (`-1` = all as map) |
| `timing` | `BEFORE` | `BEFORE`, `AFTER`, or `BOTH` relative to method execution |
| `condition` | `""` | SpEL condition: `"#param0 != null"` |
| `key` | `""` | SpEL partition key: `"#param0.orderId"` |
| `headers` | `{}` | Custom headers: `{"priority=high", "source=web"}` |
| `async` | `true` | Whether to fire-and-forget or wait for publish confirmation |
| `timeoutMs` | `0` | Publish timeout (0 = use default) |
| `serializer` | `""` | Custom serializer bean name |

### Option C -- Declarative Result Publishing with @PublishResult

Annotate a method with `@PublishResult` to automatically publish its return value after successful execution.

```java
@PublishResult(
    publisherType = PublisherType.KAFKA,
    destination = "user-events",
    eventType = "user.created",
    key = "#result.userId"
)
public Mono<UserCreatedResult> createUser(CreateUserRequest request) {
    return userRepository.save(request.toUser())
        .map(user -> new UserCreatedResult(user.getId(), user.getEmail()));
}
```

Combine `@EventPublisher` and `@PublishResult` on the same method to publish parameters before execution and the result after:

```java
@EventPublisher(
    publisherType = PublisherType.KAFKA,
    destination = "user-commands",
    eventType = "user.create.requested",
    parameterIndex = 0
)
@PublishResult(
    publisherType = PublisherType.KAFKA,
    destination = "user-events",
    eventType = "user.created"
)
public Mono<User> createUser(CreateUserRequest request) {
    return userRepository.save(request.toUser());
}
```

`@PublishResult` supports the same attributes as `@EventPublisher` plus `publishOnError` (default `false`) which publishes the exception in the event payload when the method fails.

## Consuming Events

### Option A -- Declarative Consumption with @EventListener

Annotate methods with `org.fireflyframework.eda.annotation.EventListener` to receive events. The `EventListenerProcessor` discovers annotated methods at startup, registers them by destination and event type, and routes incoming events accordingly.

```java
package com.example.notification.listener;

import org.fireflyframework.eda.annotation.ErrorHandlingStrategy;
import org.fireflyframework.eda.annotation.EventListener;
import org.fireflyframework.eda.annotation.PublisherType;
import org.fireflyframework.eda.event.EventEnvelope;
import reactor.core.publisher.Mono;

@Component
public class NotificationEventListeners {

    @EventListener(
        destinations = {"user-events"},
        eventTypes = {"user.created", "user.updated"},
        consumerType = PublisherType.KAFKA,
        groupId = "notification-service",
        errorStrategy = ErrorHandlingStrategy.DEAD_LETTER,
        maxRetries = 3,
        retryDelayMs = 1000
    )
    public Mono<Void> handleUserEvents(EventEnvelope envelope) {
        Object payload = envelope.payload();
        String eventType = envelope.eventType();
        Map<String, Object> headers = envelope.headers();

        return notificationService.sendWelcomeEmail(payload)
            .then(envelope.acknowledge());
    }

    @EventListener(
        destinations = {"order-events"},
        consumerType = PublisherType.RABBITMQ,
        errorStrategy = ErrorHandlingStrategy.LOG_AND_RETRY,
        maxRetries = 5,
        retryDelayMs = 2000
    )
    public Mono<Void> handleOrderEvents(Object event, Map<String, Object> headers) {
        // Direct event object and headers injection
        return processOrder(event);
    }
}
```

**Annotation attributes**:

| Attribute | Default | Description |
|---|---|---|
| `destinations` | `{}` | Topics, queues, or channels to listen on (empty = all) |
| `eventTypes` | `{}` | Event types to handle (supports glob: `"user.*"`) |
| `consumerType` | `AUTO` | `KAFKA`, `RABBITMQ`, `AUTO` |
| `connectionId` | `"default"` | Named connection ID |
| `groupId` | `""` | Consumer group (Kafka) |
| `errorStrategy` | `LOG_AND_CONTINUE` | Error handling strategy (see below) |
| `maxRetries` | `3` | Retry attempts for `LOG_AND_RETRY` |
| `retryDelayMs` | `1000` | Delay between retries |
| `autoAck` | `true` | Auto-acknowledge on success |
| `condition` | `""` | SpEL filter: `"#envelope.headers['priority'] == 'high'"` |
| `priority` | `0` | Listener priority (higher = earlier execution) |
| `async` | `true` | Async processing on parallel scheduler |
| `timeoutMs` | `0` | Processing timeout (0 = 30s default) |

**Method signatures** -- The `EventListenerProcessor` supports these parameter patterns:

```java
// EventEnvelope (full access to metadata, headers, ack/reject)
public Mono<Void> handle(EventEnvelope envelope) { ... }

// Direct event object
public Mono<Void> handle(UserCreatedEvent event) { ... }

// Event object + headers map
public Mono<Void> handle(UserCreatedEvent event, Map<String, Object> headers) { ... }
```

### Option B -- Programmatic Consumption with EventConsumer

Inject an `EventConsumer` implementation for reactive stream-based consumption:

```java
@Service
public class StreamProcessingService {

    private final EventConsumer kafkaConsumer;

    public StreamProcessingService(KafkaEventConsumer kafkaConsumer) {
        this.kafkaConsumer = kafkaConsumer;
    }

    public Flux<EventEnvelope> processOrderStream() {
        return kafkaConsumer.consume("order-events", "payment-events")
            .filter(envelope -> "order.completed".equals(envelope.eventType()))
            .flatMap(this::enrichAndForward);
    }
}
```

The `EventConsumer` interface provides:
- `consume()` -- returns `Flux<EventEnvelope>` for all configured destinations
- `consume(String... destinations)` -- returns filtered `Flux<EventEnvelope>`
- `start()` / `stop()` -- lifecycle control returning `Mono<Void>`
- `isRunning()`, `isAvailable()`, `getHealth()` -- status checks

### Option C -- Dynamic Listener Registration

Use `DynamicEventListenerRegistry` to register listeners programmatically at runtime:

```java
@Service
public class TenantEventSubscriptionService {

    private final DynamicEventListenerRegistry registry;

    public void subscribeTenant(String tenantId) {
        registry.registerListener(
            "tenant-" + tenantId,              // listener ID
            tenantId + "-events",              // destination
            new String[]{"order.*"},           // event type patterns
            PublisherType.KAFKA,               // consumer type
            (event, headers) -> {              // handler
                log.info("Tenant {} event: {}", tenantId, event);
                return Mono.empty();
            }
        );
    }

    public void unsubscribeTenant(String tenantId) {
        registry.unregisterListener("tenant-" + tenantId);
    }
}
```

Dynamic listeners are automatically picked up by Kafka/RabbitMQ consumers and trigger topic subscription refresh.

## EventEnvelope

`EventEnvelope` (`org.fireflyframework.eda.event.EventEnvelope`) is a Java record that wraps every event with metadata. Use the static factory methods to construct envelopes.

```java
// Minimal envelope
EventEnvelope envelope = EventEnvelope.minimal("order-events", "order.created", orderEvent);

// Full publishing envelope
EventEnvelope envelope = EventEnvelope.forPublishing(
    "order-events", "order.created", orderEvent,
    UUID.randomUUID().toString(),            // transactionId
    Map.of("priority", "high"),              // headers
    EventEnvelope.EventMetadata.minimal("corr-123"),
    Instant.now(), "KAFKA", "default"        // publisherType, connectionId
);

// Transform an envelope
EventEnvelope enriched = envelope.withPayload(transformedPayload)
    .withHeaders(Map.of("enriched", "true"));
```

**EventMetadata** fields: `correlationId`, `causationId`, `version`, `source`, `userId`, `sessionId`, `tenantId`, `custom` (Map).

**Consumer-side operations**: `envelope.acknowledge()` to ack, `envelope.reject(error)` to nack and trigger redelivery. Both return `Mono<Void>`.

## Error Handling Strategies

The `ErrorHandlingStrategy` enum (`org.fireflyframework.eda.annotation.ErrorHandlingStrategy`) controls what happens when an `@EventListener` method fails:

| Strategy | Behavior |
|---|---|
| `LOG_AND_CONTINUE` | Log the error, acknowledge the message, continue processing |
| `LOG_AND_RETRY` | Retry with exponential backoff up to `maxRetries`, then fall through |
| `REJECT_AND_STOP` | Nack the message, propagate the error, may stop the consumer |
| `DEAD_LETTER` | Acknowledge the original, forward to `{destination}.dlq` with error metadata |
| `IGNORE` | Swallow the error silently, acknowledge, continue |
| `CUSTOM` | Delegate to `CustomErrorHandler` beans in the application context |

### Dead Letter Queue

When `errorStrategy = ErrorHandlingStrategy.DEAD_LETTER`, the `EventListenerProcessor` publishes a `DeadLetterQueueEvent` via Spring's `ApplicationEventPublisher`. The `DeadLetterQueueHandler` catches it and forwards the failed event to `{original-destination}.dlq` with enriched headers:

- `dlq_reason` -- `"listener_error"`
- `dlq_error_message` -- exception message
- `dlq_error_class` -- exception class name
- `dlq_timestamp` -- failure timestamp
- `dlq_original_destination` -- where the event was originally consumed from

```java
@EventListener(
    destinations = {"payment-events"},
    errorStrategy = ErrorHandlingStrategy.DEAD_LETTER
)
public Mono<Void> handlePayment(EventEnvelope envelope) {
    // If this throws, the event goes to "payment-events.dlq"
    return paymentService.process(envelope.payload());
}
```

### Custom Error Handlers

Implement `CustomErrorHandler` (`org.fireflyframework.eda.error.CustomErrorHandler`) and register it as a Spring bean:

```java
@Component
public class AlertingErrorHandler implements CustomErrorHandler {

    @Override
    public Mono<Void> handleError(Object event, Map<String, Object> headers,
                                  Throwable error, String listenerMethod) {
        return alertService.sendAlert("Event processing failed in " + listenerMethod,
            error.getMessage());
    }

    @Override
    public String getHandlerName() {
        return "alerting-error-handler";
    }

    @Override
    public boolean canHandle(Class<? extends Throwable> errorType) {
        return RuntimeException.class.isAssignableFrom(errorType);
    }

    @Override
    public int getPriority() {
        return 100; // higher = runs first
    }
}
```

Use it with `errorStrategy = ErrorHandlingStrategy.CUSTOM` on `@EventListener`.

## Event Filtering

The `EventFilter` interface (`org.fireflyframework.eda.filter.EventFilter`) provides composable server-side filtering. Register filter beans to apply them globally to consumers.

```java
// Static factory methods
EventFilter byDest = EventFilter.byDestination("order-events");
EventFilter byType = EventFilter.byEventType("order.created");
EventFilter byHeader = EventFilter.byHeader("priority", "high");
EventFilter byPresence = EventFilter.byHeaderPresence("correlation_id");

// Compose with AND/OR/NOT
EventFilter composite = EventFilter.and(
    EventFilter.byDestination("order-events"),
    EventFilter.or(
        EventFilter.byEventType("order.created"),
        EventFilter.byEventType("order.updated")
    ),
    EventFilter.not(EventFilter.byHeader("source", "internal"))
);
boolean matches = composite.matches(envelope);
```

**Built-in implementations**: `EventTypeFilter` (wildcard pattern matching on event class/type headers), `HeaderEventFilter` (header name + value pattern), `DestinationEventFilter`, `CompositeEventFilter` (AND/OR logic). Register a filter as a `@Bean` for global application.

## Serialization

The `MessageSerializer` interface (`org.fireflyframework.eda.serialization.MessageSerializer`) is pluggable. Three implementations are auto-configured:

### JSON (Default)

`JsonMessageSerializer` uses Jackson `ObjectMapper`. It is `@Primary` with priority 100.

```yaml
firefly:
  eda:
    default-serialization-format: json
```

Works with any POJO, String, or byte array. Content type: `application/json`.

### Avro

`AvroMessageSerializer` is auto-configured when `org.apache.avro.Schema` is on the classpath. Priority 80.

Supports three modes:
1. **SpecificRecord** -- Avro-generated classes (most efficient)
2. **GenericRecord** -- Schema-based generic records
3. **Reflection** -- Plain POJOs serialized via `ReflectData`

Content type: `application/avro`.

### Protobuf

`ProtobufMessageSerializer` is auto-configured when `com.google.protobuf.Message` is on the classpath. Priority 90.

Only works with Protocol Buffer generated classes that extend `Message`. Uses the static `parser()` method on generated classes for deserialization.

Content type: `application/x-protobuf`.

### Custom Serializer

Implement `MessageSerializer` and register as a Spring bean. Implement `serialize()`, `deserialize()`, `getFormat()` (return `SerializationFormat.CUSTOM`), and `getContentType()`. Reference it by bean name in `@EventPublisher(serializer = "myCustomSerializer")`.

## Resilience

The `ResilientEventPublisher` wraps any `EventPublisher` with Resilience4j circuit breaker, retry, and rate limiting. It is applied automatically by `EventPublisherFactory` when `ResilientEventPublisherFactory` is available and `firefly.eda.resilience.enabled=true` (default).

### Circuit Breaker

Prevents cascading failures. When the failure rate exceeds the threshold, the circuit opens and publish operations fail fast.

```yaml
firefly:
  eda:
    resilience:
      enabled: true
      circuit-breaker:
        enabled: true
        failure-rate-threshold: 50        # percentage
        slow-call-rate-threshold: 50      # percentage
        slow-call-duration-threshold: 60s
        minimum-number-of-calls: 10
        sliding-window-size: 10
        wait-duration-in-open-state: 60s
        permitted-number-of-calls-in-half-open-state: 3
```

### Retry

Automatically retries failed publish operations with configurable backoff.

```yaml
firefly:
  eda:
    resilience:
      retry:
        enabled: true
        max-attempts: 3
        wait-duration: 500ms
        exponential-backoff-multiplier: 2.0
```

### Rate Limiting

Controls the rate of publish operations. Disabled by default.

```yaml
firefly:
  eda:
    resilience:
      rate-limiter:
        enabled: false
        limit-for-period: 100
        limit-refresh-period: 1s
        timeout-duration: 5s
```

The resilience pipeline applies in this order: rate limiter, then circuit breaker, then retry.

## CQRS Integration

Command handlers can automatically publish their results as domain events using `@PublishDomainEvent` (`org.fireflyframework.cqrs.event.annotation.PublishDomainEvent`). This requires `fireflyframework-eda` on the classpath.

```java
package com.example.order.command;

import org.fireflyframework.cqrs.annotations.CommandHandlerComponent;
import org.fireflyframework.cqrs.command.CommandHandler;
import org.fireflyframework.cqrs.event.annotation.PublishDomainEvent;
import reactor.core.publisher.Mono;

@CommandHandlerComponent
@PublishDomainEvent(destination = "order-events", eventType = "OrderCreated")
public class CreateOrderHandler extends CommandHandler<CreateOrderCommand, OrderCreatedResult> {

    private final OrderService orderService;

    public CreateOrderHandler(OrderService orderService) {
        this.orderService = orderService;
    }

    @Override
    protected Mono<OrderCreatedResult> doHandle(CreateOrderCommand command) {
        return orderService.createOrder(command);
        // OrderCreatedResult is auto-published to "order-events" with type "OrderCreated"
    }
}
```

The `EdaCommandEventPublisher` uses `EventPublisherFactory.getPublisher(PublisherType.AUTO)` and enriches headers with `eventType`, `commandId`, `commandType`, `correlationId`, and `initiatedBy` from the command.

If `eventType` is empty, it defaults to the result class simple name (e.g., `OrderCreatedResult`).

Publishing failures are logged but never fail the command execution.

## Configuration Reference

All properties live under `firefly.eda.*` and are bound to `EdaProperties` (`org.fireflyframework.eda.properties.EdaProperties`).

```yaml
firefly:
  eda:
    enabled: true
    default-publisher-type: AUTO          # AUTO, KAFKA, RABBITMQ, APPLICATION_EVENT, NOOP
    default-connection-id: default
    default-destination: events
    default-serialization-format: json    # json, avro, protobuf
    default-timeout: 30s
    metrics-enabled: true
    health-enabled: true
    tracing-enabled: true

    publishers:
      enabled: false                      # Set true to enable publishing
      application-event:
        enabled: true
        default-destination: application-events
      kafka:
        default:                          # Connection ID (supports multiple named connections)
          enabled: false
          bootstrap-servers: localhost:9092
          default-topic: events
          key-serializer: org.apache.kafka.common.serialization.StringSerializer
          value-serializer: org.apache.kafka.common.serialization.StringSerializer
          properties: {}                  # Additional Kafka producer properties
      rabbitmq:
        default:
          enabled: false
          host: localhost
          port: 5672
          username: guest
          password: guest
          virtual-host: /
          default-exchange: events
          default-routing-key: event

    consumer:
      enabled: false                      # Set true to enable consumption
      group-id: firefly-eda
      concurrency: 1                      # 1-100
      retry:
        enabled: true
        max-attempts: 3
        initial-delay: 1s
        max-delay: 5m
        multiplier: 2.0
      kafka:
        default:
          enabled: false
          bootstrap-servers: localhost:9092
          topics: events
          auto-offset-reset: earliest
      rabbitmq:
        default:
          enabled: false
          host: localhost
          port: 5672
          queues: events-queue
          concurrent-consumers: 1
          max-concurrent-consumers: 5
          prefetch-count: 10

    resilience:
      enabled: true
      circuit-breaker:
        enabled: true
        failure-rate-threshold: 50
        sliding-window-size: 10
        wait-duration-in-open-state: 60s
      retry:
        enabled: true
        max-attempts: 3
        wait-duration: 500ms
        exponential-backoff-multiplier: 2.0
      rate-limiter:
        enabled: false
        limit-for-period: 100
        limit-refresh-period: 1s
        timeout-duration: 5s
```

## Complete Example -- Kafka Publish and Consume

```yaml
# application.yml
firefly:
  eda:
    enabled: true
    default-publisher-type: KAFKA
    publishers:
      enabled: true
      kafka:
        default:
          enabled: true
          bootstrap-servers: localhost:9092
          default-topic: order-events
    consumer:
      enabled: true
      group-id: order-service
      kafka:
        default:
          enabled: true
          bootstrap-servers: localhost:9092
          topics: order-events
```

```java
// --- Event ---
public record OrderCreatedEvent(String orderId, String customerId,
                                BigDecimal totalAmount, Instant createdAt) {}

// --- Publisher ---
@Service
public class OrderService {
    private final EventPublisherFactory publisherFactory;

    public Mono<Order> createOrder(CreateOrderRequest request) {
        return orderRepository.save(request.toOrder())
            .flatMap(order -> {
                var event = new OrderCreatedEvent(order.getId(), order.getCustomerId(),
                    order.getTotalAmount(), Instant.now());
                EventPublisher publisher = publisherFactory.getPublisher(PublisherType.KAFKA);
                return publisher.publish(event, "order-events", Map.of(
                    "eventType", "order.created",
                    "partition_key", order.getId()
                )).thenReturn(order);
            });
    }
}

// --- Consumer ---
@Component
public class InventoryEventListener {
    private final InventoryService inventoryService;

    @EventListener(
        destinations = {"order-events"},
        eventTypes = {"order.created"},
        consumerType = PublisherType.KAFKA,
        groupId = "inventory-service",
        errorStrategy = ErrorHandlingStrategy.DEAD_LETTER,
        maxRetries = 3,
        retryDelayMs = 2000
    )
    public Mono<Void> onOrderCreated(EventEnvelope envelope) {
        return inventoryService.reserveStock(envelope.payload())
            .then(envelope.acknowledge())
            .onErrorResume(error -> envelope.reject(error).then(Mono.error(error)));
    }
}
```

## Complete Example -- RabbitMQ with Exchange Routing

RabbitMQ destinations use the format `exchange/routingKey`. Consumers subscribe to queues configured in properties; the `@EventListener` destinations are used for message routing and filtering after consumption.

```yaml
firefly:
  eda:
    enabled: true
    default-publisher-type: RABBITMQ
    publishers:
      enabled: true
      rabbitmq:
        default:
          enabled: true
          host: localhost
          port: 5672
          default-exchange: domain-events
          default-routing-key: event
    consumer:
      enabled: true
      rabbitmq:
        default:
          enabled: true
          host: localhost
          port: 5672
          queues: order-queue,notification-queue
          concurrent-consumers: 2
          max-concurrent-consumers: 10
          prefetch-count: 20
```

```java
// Publishing to exchange with routing key
@EventPublisher(
    publisherType = PublisherType.RABBITMQ,
    destination = "domain-events/order.created",  // exchange/routingKey
    eventType = "order.created"
)
public Mono<Order> createOrder(CreateOrderRequest request) {
    return orderRepository.save(request.toOrder());
}

// Consuming from RabbitMQ queues
@EventListener(
    destinations = {"domain-events/order.*"},
    consumerType = PublisherType.RABBITMQ,
    errorStrategy = ErrorHandlingStrategy.LOG_AND_RETRY,
    maxRetries = 5
)
public Mono<Void> handleOrderEvents(EventEnvelope envelope) {
    return orderProcessor.process(envelope.payload());
}
```

## Package Structure

```
com.example.myservice/
  event/
    OrderCreatedEvent.java              # event payload record/class
    OrderUpdatedEvent.java
    PaymentCompletedEvent.java
  listener/
    OrderEventListener.java            # @EventListener methods, @Component
    PaymentEventListener.java
  publisher/
    OrderEventPublisher.java           # wrapper around EventPublisherFactory
  filter/
    PriorityEventFilter.java           # implements EventFilter (optional)
  error/
    OrderErrorHandler.java             # implements CustomErrorHandler (optional)
  config/
    EdaConfiguration.java              # additional EDA beans (optional)
```

## Common Mistakes

1. **Blocking in event listeners** -- `@EventListener` methods must return `Mono<Void>`. Never call `.block()`, `Thread.sleep()`, or perform synchronous I/O inside a listener. Use reactive operators to compose async calls.

2. **Forgetting to enable publishers and consumers** -- Both `firefly.eda.publishers.enabled` and `firefly.eda.consumer.enabled` default to `false`. You must explicitly set them to `true` in `application.yml` or your events will not flow.

3. **Missing transport dependency** -- `PublisherType.KAFKA` requires `spring-kafka` on the classpath. `PublisherType.RABBITMQ` requires `spring-amqp` and `spring-rabbit`. Without the dependency, the publisher/consumer will not be auto-configured and `getPublisher()` returns `null`.

4. **RabbitMQ destination format** -- RabbitMQ destinations follow the `exchange/routingKey` convention. Using a bare topic name without the exchange prefix will publish to the default exchange with an empty routing key.

5. **Ignoring acknowledgment** -- When `autoAck = false`, you must call `envelope.acknowledge()` or `envelope.reject(error)` explicitly. Failing to acknowledge means the message stays unprocessed and may be redelivered indefinitely.

6. **Publishing without checking availability** -- Always check `publisher.isAvailable()` or handle `null` from `publisherFactory.getPublisher()`. In test environments or when the broker is down, the publisher may not be available.

7. **Serialization mismatch** -- If the publisher uses `AvroMessageSerializer` but the consumer expects JSON, deserialization will fail silently. Ensure both sides use the same serialization format. The `event_class` header is used to determine the target deserialization class.

8. **Circular dependency with DLQ** -- Do not inject `EventPublisherFactory` into your `@EventListener` component if you also have `ErrorHandlingStrategy.DEAD_LETTER`. The framework handles DLQ publishing through `DeadLetterQueueHandler` via Spring's `ApplicationEventPublisher` to avoid circular dependencies.

9. **Dynamic listeners without consumer refresh** -- When using `DynamicEventListenerRegistry`, the Kafka consumer automatically refreshes topic subscriptions via a callback. However, this causes a brief interruption in message consumption. Plan dynamic listener registration for startup or low-traffic periods.

10. **Overly broad event type patterns** -- Using `eventTypes = {}` (empty) on `@EventListener` matches all events on the destination. This is fine for single-purpose topics but causes unexpected behavior on shared topics. Always specify event types when multiple services publish to the same topic.
