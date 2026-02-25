---
name: implement-cache-strategy
description: Use when adding caching to a Firefly Framework service — covers the unified cache abstraction with Caffeine/Redis/Hazelcast configuration, multi-tier strategies, health monitoring, and invalidation patterns
---

# Implement Cache Strategy

## When to Use

Use this skill when a Firefly Framework service needs caching -- whether for query results, HTTP idempotency, rate-limiting tokens, or any data that benefits from fast retrieval and controlled expiration. The `fireflyframework-cache` module provides a unified reactive `CacheAdapter` interface with pluggable backends (Caffeine, Redis, Hazelcast, JCache) and automatic multi-tier (L1+L2) composition via the `SmartCacheAdapter`.

## Architecture Overview

```
CacheAdapter (reactive interface)
  |
  +-- CaffeineCacheAdapter    (in-memory, always available)
  +-- RedisCacheAdapter        (distributed, optional)
  +-- SmartCacheAdapter        (L1 Caffeine + L2 distributed composite)
  |
  +-- FireflyCacheManager      (delegates to active adapter with fallback)
  |
  +-- CacheManagerFactory      (creates independent cache managers per use-case)
```

All cache operations return `Mono<...>` for non-blocking reactive compatibility. The `FireflyCacheManager` implements `CacheAdapter` itself, so consumers code against one interface regardless of the backend.

## Step 1 -- Add the Dependency

```xml
<dependency>
    <groupId>org.fireflyframework</groupId>
    <artifactId>fireflyframework-cache</artifactId>
    <version>${fireflyframework.version}</version>
</dependency>
```

For Redis support, also add (these are optional in the module POM):

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
<dependency>
    <groupId>io.lettuce</groupId>
    <artifactId>lettuce-core</artifactId>
</dependency>
```

No extra dependency is needed for Caffeine -- it is a required transitive dependency of `fireflyframework-cache`.

## Step 2 -- Configure the Cache Provider

All properties live under the `firefly.cache` prefix and are bound to `org.fireflyframework.cache.properties.CacheProperties`. Auto-configuration is handled by `CacheAutoConfiguration` (core) and `RedisCacheAutoConfiguration` (Redis infrastructure beans).

### Caffeine (default, in-memory)

```yaml
firefly:
  cache:
    enabled: true
    default-cache-type: CAFFEINE
    metrics-enabled: true
    health-enabled: true
    stats-enabled: true
    caffeine:
      enabled: true
      cache-name: default
      key-prefix: "firefly:cache"
      maximum-size: 1000
      expire-after-write: PT1H          # ISO-8601 duration
      expire-after-access: ~            # null = not set
      refresh-after-write: ~
      record-stats: true
      weak-keys: false
      weak-values: false
      soft-values: false
```

### Redis (distributed)

```yaml
firefly:
  cache:
    enabled: true
    default-cache-type: REDIS
    redis:
      enabled: true
      cache-name: default
      host: redis.internal
      port: 6379
      database: 0
      username: ~
      password: ${REDIS_PASSWORD}
      connection-timeout: PT10S
      command-timeout: PT5S
      key-prefix: "firefly:cache"
      default-ttl: PT30M
      enable-keyspace-notifications: false
      max-pool-size: 8
      min-pool-size: 0
      ssl: false
```

### AUTO (let the framework choose)

```yaml
firefly:
  cache:
    default-cache-type: AUTO
```

When set to `AUTO`, the `CacheManagerFactory` probes providers via the SPI (`CacheProviderFactory`) in priority order: Redis (10) > Hazelcast (20) > JCache (30) > Caffeine (40). The first available provider is selected.

### Smart L1+L2 (multi-tier)

When a distributed provider is selected and Caffeine is also enabled, the factory automatically wraps them in a `SmartCacheAdapter` that uses Caffeine as L1 and the distributed cache as L2:

```yaml
firefly:
  cache:
    default-cache-type: REDIS
    caffeine:
      enabled: true          # L1 layer
    redis:
      enabled: true          # L2 layer
    smart:
      enabled: true           # enable L1+L2 composition
      write-strategy: WRITE_THROUGH
      backfill-l1-on-read: true
```

With `backfill-l1-on-read: true`, a cache miss in L1 that hits in L2 will backfill L1 automatically -- subsequent reads for the same key skip the network entirely.

## Step 3 -- Use the CacheAdapter in Your Service

### Injecting the Default Cache Manager

`CacheAutoConfiguration` creates a `@Primary` `FireflyCacheManager` bean. Inject it wherever you need caching:

```java
import org.fireflyframework.cache.manager.FireflyCacheManager;
import reactor.core.publisher.Mono;

import java.time.Duration;
import java.util.Optional;

@Service
public class AccountService {

    private final FireflyCacheManager cacheManager;

    public AccountService(FireflyCacheManager cacheManager) {
        this.cacheManager = cacheManager;
    }

    public Mono<AccountDTO> getAccount(String accountId) {
        String cacheKey = "account:" + accountId;

        return cacheManager.<String, AccountDTO>get(cacheKey, AccountDTO.class)
            .flatMap(optional -> {
                if (optional.isPresent()) {
                    return Mono.just(optional.get());      // cache hit
                }
                return accountRepository.findById(accountId)
                    .flatMap(dto ->
                        cacheManager.put(cacheKey, dto, Duration.ofMinutes(15))
                            .thenReturn(dto)               // cache miss -> store and return
                    );
            });
    }

    public Mono<Void> updateAccount(String accountId, AccountDTO updated) {
        return accountRepository.save(updated)
            .then(cacheManager.<String>evict("account:" + accountId))
            .then();
    }
}
```

### Core CacheAdapter Operations

The `CacheAdapter` interface (`org.fireflyframework.cache.core.CacheAdapter`) defines these reactive operations:

| Method | Signature | Description |
|--------|-----------|-------------|
| `get` | `<K,V> Mono<Optional<V>> get(K key)` | Retrieve by key |
| `get` | `<K,V> Mono<Optional<V>> get(K key, Class<V> valueType)` | Retrieve with type-safe deserialization |
| `put` | `<K,V> Mono<Void> put(K key, V value)` | Store (uses config default TTL) |
| `put` | `<K,V> Mono<Void> put(K key, V value, Duration ttl)` | Store with explicit TTL |
| `putIfAbsent` | `<K,V> Mono<Boolean> putIfAbsent(K key, V value)` | Atomic conditional store |
| `putIfAbsent` | `<K,V> Mono<Boolean> putIfAbsent(K key, V value, Duration ttl)` | Atomic conditional store with TTL |
| `evict` | `<K> Mono<Boolean> evict(K key)` | Remove single entry |
| `evictByPrefix` | `Mono<Long> evictByPrefix(String keyPrefix)` | Remove all entries with matching prefix |
| `clear` | `Mono<Void> clear()` | Remove all entries |
| `exists` | `<K> Mono<Boolean> exists(K key)` | Check if key is present |
| `keys` | `<K> Mono<Set<K>> keys()` | List all keys (expensive for distributed) |
| `size` | `Mono<Long> size()` | Entry count |
| `getStats` | `Mono<CacheStats> getStats()` | Performance statistics |
| `getHealth` | `Mono<CacheHealth> getHealth()` | Health status |
| `isAvailable` | `boolean isAvailable()` | Availability check |

## Step 4 -- Create Named Caches with CacheManagerFactory

For multiple independent caches (different key namespaces, TTLs, or providers), inject `CacheManagerFactory` and create dedicated managers:

```java
import org.fireflyframework.cache.factory.CacheManagerFactory;
import org.fireflyframework.cache.core.CacheType;
import org.fireflyframework.cache.manager.FireflyCacheManager;

@Configuration
public class CacheConfig {

    @Bean("idempotencyCache")
    public FireflyCacheManager idempotencyCache(CacheManagerFactory factory) {
        return factory.createCacheManager(
            "http-idempotency",              // cache name
            CacheType.REDIS,                  // provider
            "firefly:http:idempotency",       // key prefix
            Duration.ofHours(24)              // default TTL
        );
    }

    @Bean("webhookCache")
    public FireflyCacheManager webhookCache(CacheManagerFactory factory) {
        return factory.createCacheManager(
            "webhook-events",
            CacheType.REDIS,
            "firefly:webhooks:events",
            Duration.ofDays(7),
            "Deduplication cache for webhook deliveries",  // description
            "WebhookModule"                                 // requestedBy
        );
    }

    @Bean("sessionCache")
    public FireflyCacheManager sessionCache(CacheManagerFactory factory) {
        return factory.createCacheManager(
            "user-sessions",
            CacheType.CAFFEINE,
            "firefly:sessions",
            Duration.ofMinutes(30)
        );
    }
}
```

The `CacheManagerFactory` automatically wraps distributed providers with `SmartCacheAdapter` when `smart.enabled=true` and Caffeine is available, giving each named cache its own L1+L2 stack.

## Step 5 -- Monitor Cache Health and Statistics

### Health Checks

Every `CacheAdapter` exposes `getHealth()` returning `Mono<CacheHealth>`. The `CacheHealth` model (`org.fireflyframework.cache.core.CacheHealth`) provides:

```java
CacheHealth health = cacheManager.getHealth().block();

health.isHealthy();          // true if UP + available + configured
health.getOverallStatus();   // "UP", "DOWN", "DEGRADED", "NOT_CONFIGURED"
health.getResponseTimeMs();  // latency of the health-check probe
health.getConsecutiveFailures();
health.getErrorMessage();
```

Factory methods for creating health objects in custom adapters:

```java
CacheHealth.healthy(CacheType.REDIS, "my-cache", 3L);
CacheHealth.unhealthy(CacheType.REDIS, "my-cache", "Connection refused", exception);
CacheHealth.notConfigured(CacheType.REDIS, "my-cache");
```

### Statistics

`getStats()` returns `Mono<CacheStats>` with hit/miss rates and sizing:

```java
CacheStats stats = cacheManager.getStats().block();

stats.getHitRate();               // percentage 0..100
stats.getMissRate();
stats.getRequestCount();
stats.getHitCount();
stats.getMissCount();
stats.getEvictionCount();
stats.getEntryCount();
stats.getAverageLoadTimeMillis();
stats.getFormattedEstimatedSize(); // "1.23 MB"
```

Enable stats collection via `firefly.cache.caffeine.record-stats=true` (Caffeine) or via the `stats-enabled` property.

## Step 6 -- Integrate with CQRS Query Caching

The `fireflyframework-cqrs` module bridges into the cache module via `QueryCacheAdapter` and the `@QueryHandlerComponent` annotation.

### Cacheable Query Handlers

Mark a query handler as cacheable and set TTL:

```java
import org.fireflyframework.cqrs.annotations.QueryHandlerComponent;
import org.fireflyframework.cqrs.query.QueryHandler;
import reactor.core.publisher.Mono;

@QueryHandlerComponent(cacheable = true, cacheTtl = 300, cacheKeyFields = {"accountId"})
public class GetAccountBalanceHandler extends QueryHandler<GetAccountBalanceQuery, AccountBalance> {

    @Override
    protected Mono<AccountBalance> doHandle(GetAccountBalanceQuery query) {
        return accountRepository.findBalance(query.getAccountId());
    }
}
```

The `QueryBus` automatically uses `QueryCacheAdapter` (which wraps `FireflyCacheManager`) to cache results. Cache keys are prefixed with `:cqrs:` so they do not collide with other cache users. The full key path becomes `firefly:cache:{cacheName}::cqrs:{queryCacheKey}`.

### Implement the Query with Cache Key

```java
import org.fireflyframework.cqrs.query.Query;

public class GetAccountBalanceQuery implements Query<AccountBalance> {

    private final String accountId;

    public GetAccountBalanceQuery(String accountId) {
        this.accountId = accountId;
    }

    @Override
    public String getCacheKey() {
        return "account-balance:" + accountId;
    }

    @Override
    public boolean isCacheable() {
        return true;
    }

    public String getAccountId() { return accountId; }
}
```

### Event-Driven Cache Invalidation with @InvalidateCacheOn

For eventual consistency between command-side mutations and query-side reads, annotate query handlers with `@InvalidateCacheOn` from `org.fireflyframework.cqrs.cache.annotation`:

```java
import org.fireflyframework.cqrs.annotations.QueryHandlerComponent;
import org.fireflyframework.cqrs.cache.annotation.InvalidateCacheOn;
import org.fireflyframework.cqrs.query.QueryHandler;

@QueryHandlerComponent(cacheable = true, cacheTtl = 300)
@InvalidateCacheOn(eventTypes = {"AccountCreated", "AccountUpdated", "AccountClosed"})
public class GetAccountQueryHandler extends QueryHandler<GetAccountQuery, AccountDTO> {

    @Override
    protected Mono<AccountDTO> doHandle(GetAccountQuery query) {
        return accountRepository.findById(query.getAccountId());
    }
}
```

When domain events with matching `eventType` values arrive (via the EDA module), the `EventDrivenCacheInvalidator` scans registered handlers and clears the CQRS query cache automatically. This requires `fireflyframework-eda` on the classpath.

The invalidator is initialized at startup by scanning all `QueryHandler` beans for `@InvalidateCacheOn` annotations and building an event-type-to-handler mapping.

## Step 7 -- Custom Serialization

The default serializer is `JsonCacheSerializer` (using Jackson `ObjectMapper`). It handles:
- Primitives and strings passed through directly for performance
- Complex objects serialized to JSON strings

To provide a custom serializer, implement `CacheSerializer` and declare it as a bean:

```java
import org.fireflyframework.cache.serialization.CacheSerializer;
import org.fireflyframework.cache.serialization.SerializationException;

@Component
public class ProtobufCacheSerializer implements CacheSerializer {

    @Override
    public Object serialize(Object object) throws SerializationException {
        // Custom serialization logic
    }

    @Override
    public <T> T deserialize(Object data, Class<T> type) throws SerializationException {
        // Custom deserialization logic
    }

    @Override
    public boolean supports(Class<?> type) {
        return true;
    }

    @Override
    public String getType() {
        return "protobuf";
    }
}
```

The `@ConditionalOnMissingBean` on the default serializer ensures your custom bean takes precedence.

## Step 8 -- Extend with Custom Providers (SPI)

The cache module supports pluggable providers via `java.util.ServiceLoader`. Implement `CacheProviderFactory` and register it in `META-INF/services/org.fireflyframework.cache.spi.CacheProviderFactory`:

```java
import org.fireflyframework.cache.core.CacheAdapter;
import org.fireflyframework.cache.core.CacheType;
import org.fireflyframework.cache.spi.CacheProviderFactory;

public class MemcachedProvider implements CacheProviderFactory {

    @Override
    public CacheType getType() { return CacheType.JCACHE; }

    @Override
    public int priority() { return 25; } // between Hazelcast(20) and JCache(30)

    @Override
    public boolean isAvailable(ProviderContext ctx) {
        try {
            Class.forName("net.spy.memcached.MemcachedClient");
            return true;
        } catch (ClassNotFoundException e) {
            return false;
        }
    }

    @Override
    public CacheAdapter create(String cacheName, String keyPrefix,
                                Duration defaultTtl, ProviderContext ctx) {
        // Build and return your CacheAdapter implementation
    }
}
```

Built-in SPI providers and their priorities:
- `RedisProvider` -- priority 10
- `HazelcastProvider` -- priority 20
- `JCacheProvider` -- priority 30
- `CaffeineProvider` -- priority 40

## Configuration Reference

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `firefly.cache.enabled` | boolean | `true` | Master switch for the cache module |
| `firefly.cache.default-cache-type` | CacheType | `CAFFEINE` | CAFFEINE, REDIS, HAZELCAST, JCACHE, AUTO, NOOP |
| `firefly.cache.metrics-enabled` | boolean | `true` | Enable Micrometer metrics |
| `firefly.cache.health-enabled` | boolean | `true` | Enable health checks |
| `firefly.cache.stats-enabled` | boolean | `true` | Enable statistics collection |
| `firefly.cache.caffeine.enabled` | boolean | `true` | Enable Caffeine provider |
| `firefly.cache.caffeine.cache-name` | String | `default` | Caffeine cache name |
| `firefly.cache.caffeine.key-prefix` | String | `firefly:cache` | Key prefix |
| `firefly.cache.caffeine.maximum-size` | long | `1000` | Max entries |
| `firefly.cache.caffeine.expire-after-write` | Duration | `PT1H` | TTL after write |
| `firefly.cache.caffeine.expire-after-access` | Duration | null | TTL after last access |
| `firefly.cache.caffeine.refresh-after-write` | Duration | null | Async refresh interval |
| `firefly.cache.caffeine.record-stats` | boolean | `true` | Track hit/miss stats |
| `firefly.cache.caffeine.weak-keys` | boolean | `false` | WeakReference keys |
| `firefly.cache.caffeine.weak-values` | boolean | `false` | WeakReference values |
| `firefly.cache.caffeine.soft-values` | boolean | `false` | SoftReference values |
| `firefly.cache.redis.enabled` | boolean | `true` | Enable Redis provider |
| `firefly.cache.redis.cache-name` | String | `default` | Redis cache name |
| `firefly.cache.redis.host` | String | `localhost` | Redis host |
| `firefly.cache.redis.port` | int | `6379` | Redis port |
| `firefly.cache.redis.database` | int | `0` | Redis database index |
| `firefly.cache.redis.username` | String | null | ACL username |
| `firefly.cache.redis.password` | String | null | Authentication password |
| `firefly.cache.redis.connection-timeout` | Duration | `PT10S` | Connection timeout |
| `firefly.cache.redis.command-timeout` | Duration | `PT5S` | Command timeout |
| `firefly.cache.redis.key-prefix` | String | `firefly:cache` | Key prefix |
| `firefly.cache.redis.default-ttl` | Duration | null | Default entry TTL |
| `firefly.cache.redis.enable-keyspace-notifications` | boolean | `false` | Key expiration events |
| `firefly.cache.redis.max-pool-size` | int | `8` | Max connections |
| `firefly.cache.redis.min-pool-size` | int | `0` | Min connections |
| `firefly.cache.redis.ssl` | boolean | `false` | TLS encryption |
| `firefly.cache.smart.enabled` | boolean | `true` | Enable L1+L2 composition |
| `firefly.cache.smart.write-strategy` | String | `WRITE_THROUGH` | Write strategy |
| `firefly.cache.smart.backfill-l1-on-read` | boolean | `true` | Backfill L1 on L2 hit |

## Predefined Configuration Profiles

The `CaffeineCacheConfig` and `RedisCacheConfig` classes provide factory methods for common scenarios:

```java
// Caffeine presets
CaffeineCacheConfig.defaultConfig();            // 1000 entries, 1h TTL
CaffeineCacheConfig.highPerformanceConfig();     // 10000 entries, 2h access-based TTL
CaffeineCacheConfig.memoryEfficientConfig();     // 100 entries, 15m TTL, soft values
CaffeineCacheConfig.longLivedConfig();           // 5000 entries, 24h TTL

// Redis presets
RedisCacheConfig.defaultConfig();                // 8 connections, 10s timeout
RedisCacheConfig.highPerformanceConfig();         // 16 connections, 2s timeout, 1h TTL
RedisCacheConfig.productionConfig();              // 12 connections, keyspace notifications, 30m TTL
RedisCacheConfig.secureConfig();                  // SSL on port 6380
```

## Anti-Patterns

### Do NOT cache mutable references with Caffeine

Caffeine stores object references. If you cache a mutable object and then mutate it, every cache reader sees the mutation. Always cache immutable DTOs or defensive copies.

### Do NOT use `keys()` or `size()` in hot paths

Both `keys()` and `size()` can be expensive, especially with Redis (scan-based). Use them only in monitoring, health checks, or admin endpoints -- never in request-handling code.

### Do NOT set TTL to zero or null without intent

A `null` TTL on `put(key, value)` uses the adapter's configured default. For Caffeine, entries without TTL stay until evicted by size. For Redis with no `default-ttl`, entries persist indefinitely. Always set an explicit TTL in production.

### Do NOT ignore the reactive chain

All `CacheAdapter` methods return `Mono`. You must subscribe to them or compose them into your reactive pipeline. Calling `.block()` in a non-blocking context (WebFlux request thread) will deadlock.

```java
// WRONG -- fire-and-forget, put may never execute
cacheManager.put(key, value);

// CORRECT -- chain into the reactive pipeline
return fetchData()
    .flatMap(data -> cacheManager.put(key, data).thenReturn(data));
```

### Do NOT create CacheManagerFactory manually

The factory is auto-configured by `CacheAutoConfiguration` with Redis/Hazelcast/JCache awareness. Inject it instead of constructing it.

### Do NOT skip eviction after mutations

When data changes, always evict or overwrite the corresponding cache entry. Stale data is the most common caching bug. Use `evict()` for single keys or `evictByPrefix()` for bulk invalidation.

### Do NOT cache errors or nulls unintentionally

A `Mono.empty()` from a database query should not be cached as a "present" value. The `QueryCacheAdapter.put()` already guards against null results, but direct `CacheAdapter` usage should check:

```java
return repository.findById(id)
    .flatMap(result ->
        cacheManager.put(cacheKey, result, Duration.ofMinutes(10))
            .thenReturn(result)
    );
// If findById returns empty, put() is never called -- correct behavior
```

### Do NOT mix cache key namespaces

Use distinct key prefixes for each cache to prevent collisions. The `CacheManagerFactory` handles this automatically when you create named caches with different `keyPrefix` values. Never share a `FireflyCacheManager` instance across unrelated use-cases without separate prefixes.

## Key Classes Reference

| Class | Package | Purpose |
|-------|---------|---------|
| `CacheAdapter` | `o.f.cache.core` | Core reactive cache interface |
| `CacheType` | `o.f.cache.core` | Enum: CAFFEINE, REDIS, HAZELCAST, JCACHE, NOOP, AUTO |
| `CacheStats` | `o.f.cache.core` | Hit/miss/eviction statistics model |
| `CacheHealth` | `o.f.cache.core` | Health status model with factory methods |
| `CaffeineCacheAdapter` | `o.f.cache.adapter.caffeine` | Caffeine implementation of CacheAdapter |
| `CaffeineCacheConfig` | `o.f.cache.adapter.caffeine` | Caffeine configuration with preset profiles |
| `RedisCacheAdapter` | `o.f.cache.adapter.redis` | Redis implementation using ReactiveRedisTemplate |
| `RedisCacheConfig` | `o.f.cache.adapter.redis` | Redis configuration with preset profiles |
| `SmartCacheAdapter` | `o.f.cache.adapter.smart` | L1+L2 composite with backfill and write-through |
| `FireflyCacheManager` | `o.f.cache.manager` | Unified manager with primary/fallback delegation |
| `CacheManagerFactory` | `o.f.cache.factory` | Factory for creating named, independent cache managers |
| `CacheSelectionStrategy` | `o.f.cache.manager` | Strategy interface for auto-selection |
| `AutoCacheSelectionStrategy` | `o.f.cache.manager` | Default strategy: Redis > Hazelcast > JCache > Caffeine |
| `CacheProviderFactory` | `o.f.cache.spi` | SPI for pluggable cache providers |
| `CacheSerializer` | `o.f.cache.serialization` | Serialization interface |
| `JsonCacheSerializer` | `o.f.cache.serialization` | Default JSON serializer (Jackson) |
| `CacheException` | `o.f.cache.exception` | Base exception (extends FireflyInfrastructureException) |
| `CacheProperties` | `o.f.cache.properties` | Spring Boot config properties (`firefly.cache.*`) |
| `CacheAutoConfiguration` | `o.f.cache.config` | Core auto-config (Caffeine + factory) |
| `RedisCacheAutoConfiguration` | `o.f.cache.config` | Redis infrastructure auto-config |
| `QueryCacheAdapter` | `o.f.cqrs.cache` | CQRS bridge wrapping FireflyCacheManager |
| `EventDrivenCacheInvalidator` | `o.f.cqrs.cache` | Clears CQRS cache on domain events |
| `InvalidateCacheOn` | `o.f.cqrs.cache.annotation` | Annotation to map event types to cache invalidation |
| `QueryHandlerComponent` | `o.f.cqrs.annotations` | Annotation with cacheable, cacheTtl, cacheKeyFields |
