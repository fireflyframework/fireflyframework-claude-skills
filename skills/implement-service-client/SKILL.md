---
name: implement-service-client
description: >
  Use when consuming an external service (REST/SOAP/gRPC) in a Firefly Framework service.
  Covers the builder pattern from fireflyframework-client for REST, SOAP (CXF), gRPC,
  GraphQL, and WebSocket configuration with circuit breakers, service discovery, OAuth2,
  API keys, rate limiting, and deduplication.
---

# Implement Service Client

This skill guides you through creating service clients using the `fireflyframework-client`
abstraction layer. All clients are reactive (Mono/Flux), share a common `ServiceClient`
interface, and integrate with circuit breakers, service discovery, and observability.

## Architecture Overview

```
ServiceClient (interface)  -- org.fireflyframework.client.ServiceClient
  +-- RestClient           -- HTTP/REST via Spring WebClient
  +-- GrpcClient<T>        -- gRPC via ManagedChannel + stubs
  +-- SoapClient           -- SOAP/WSDL via JAX-WS / CXF
```

Static factory methods on `ServiceClient`:

| Factory                 | Builder (o.f.client.builder.*)  | Returns        |
|-------------------------|---------------------------------|----------------|
| `ServiceClient.rest()`  | `RestClientBuilder`             | `RestClient`   |
| `ServiceClient.grpc()`  | `GrpcClientBuilder<T>`          | `GrpcClient<T>`|
| `ServiceClient.soap()`  | `SoapClientBuilder`             | `SoapClient`   |

Helper utilities (not full `ServiceClient` implementations):

| Helper (o.f.client.*)                        | Purpose                      |
|----------------------------------------------|------------------------------|
| `graphql.GraphQLClientHelper`                | GraphQL queries/mutations    |
| `websocket.WebSocketClientHelper`            | WebSocket with reconnect     |
| `oauth2.OAuth2ClientHelper`                  | OAuth2 token management      |
| `security.ApiKeyManager`                     | API key rotation/headers     |
| `security.ClientSideRateLimiter`             | Client-side rate limiting    |
| `deduplication.RequestDeduplicationManager`  | Request deduplication        |

---

## Step 1 -- Add the Dependency

```xml
<dependency>
    <groupId>org.fireflyframework</groupId>
    <artifactId>fireflyframework-client</artifactId>
</dependency>
```

`o.f.client.config.ServiceClientAutoConfiguration` registers default beans when
`firefly.service-client.enabled=true` (default): a `WebClient.Builder` with connection
pooling, a `CircuitBreakerManager`, a default `RestClientBuilder`, a
`GrpcClientBuilderFactory`, and `ServiceClientMetrics` (when Micrometer is present).

---

## Step 2 -- REST Client

### 2a. Build a REST client

```java
import org.fireflyframework.client.ServiceClient;
import org.fireflyframework.client.RestClient;

RestClient client = ServiceClient.rest("payment-service")
    .baseUrl("https://payment-service:8443")
    .timeout(Duration.ofSeconds(30))
    .maxConnections(50)
    .defaultHeader("Accept", "application/json")
    .circuitBreakerManager(circuitBreakerManager)
    .retry(true, 3, Duration.ofMillis(500), Duration.ofSeconds(10), true)
    .build();
```

Key builder methods: `.baseUrl()` (required), `.timeout()` (default 30s),
`.maxConnections()` (default 100), `.defaultHeader()`, `.webClient()`,
`.circuitBreakerManager()`, `.retry()`, `.noRetry()`, `.jsonContentType()`,
`.xmlContentType()`.

### 2b. Fluent RequestBuilder

Every HTTP verb returns a `RequestBuilder<R>` (`o.f.client.RequestBuilder`) that
terminates with `.execute()` (Mono) or `.stream()` (Flux).

```java
// GET with path params
Mono<User> user = client.get("/users/{id}", User.class)
    .withPathParam("id", "123")
    .execute();

// GET with TypeReference for generics
Mono<List<User>> users = client.get("/users", new TypeReference<List<User>>() {})
    .withQueryParam("status", "active")
    .withQueryParam("limit", 10)
    .execute();

// POST with body and headers
Mono<User> created = client.post("/users", User.class)
    .withBody(newUserRequest)
    .withHeader("X-Correlation-Id", correlationId)
    .execute();

// PUT with per-request timeout
Mono<User> updated = client.put("/users/{id}", User.class)
    .withPathParam("id", userId)
    .withBody(updateRequest)
    .withTimeout(Duration.ofSeconds(60))
    .execute();

// PATCH and DELETE follow the same pattern
Mono<Void> deleted = client.delete("/users/{id}", Void.class)
    .withPathParam("id", userId).execute();
```

### 2c. Streaming (SSE)

```java
Flux<TransactionEvent> events = client.stream("/events/transactions", TransactionEvent.class);

Flux<AuditEvent> auditStream = client.get("/audit/stream", AuditEvent.class)
    .withQueryParam("since", Instant.now().minus(Duration.ofHours(1)))
    .stream();
```

---

## Step 3 -- gRPC Client

### 3a. Build a gRPC client

```java
import org.fireflyframework.client.GrpcClient;

GrpcClient<UserServiceGrpc.UserServiceBlockingStub> grpcClient =
    ServiceClient.grpc("user-service", UserServiceGrpc.UserServiceBlockingStub.class)
        .address("user-service:9090")
        .usePlaintext()                  // dev only; .useTransportSecurity() in prod
        .stubFactory(channel -> UserServiceGrpc.newBlockingStub(channel))
        .circuitBreakerManager(cbManager)
        .build();
```

Key builder methods: `.address()` (required), `.stubFactory()` (required),
`.timeout()` (default 30s), `.usePlaintext()`, `.useTransportSecurity()`,
`.channel(ManagedChannel)`, `.circuitBreakerManager()`.

Or use the auto-configured factory:

```java
@Autowired ServiceClientAutoConfiguration.GrpcClientBuilderFactory grpcFactory;

GrpcClient<UserServiceGrpc.UserServiceBlockingStub> client = grpcFactory
    .create("user-service", UserServiceGrpc.UserServiceBlockingStub.class)
    .address("user-service:9090")
    .stubFactory(channel -> UserServiceGrpc.newBlockingStub(channel))
    .build();
```

### 3b. Calls

```java
// Unary with circuit breaker (recommended)
Mono<UserResponse> response = grpcClient.unary(stub ->
    stub.getUserById(UserRequest.newBuilder().setId(123).build()));

// Direct stub access (no circuit breaker)
UserResponse direct = grpcClient.getStub().getUserById(request);

// Server streaming
Flux<UserEvent> events = grpcClient.serverStream(stub ->
    stub.streamUserEvents(UserEventRequest.newBuilder().build()));

// Client streaming
Mono<UploadResult> result = grpcClient.clientStream(
    stub -> stub.uploadData(), Flux.fromIterable(chunks));

// Bidirectional streaming
Flux<ChatMessage> msgs = grpcClient.bidiStream(
    stub -> stub.chat(), Flux.just(msg1, msg2));
```

---

## Step 4 -- SOAP Client

### 4a. Build a SOAP client

```java
import org.fireflyframework.client.SoapClient;

SoapClient soapClient = ServiceClient.soap("weather-service")
    .wsdlUrl("http://www.webservicex.net/globalweather.asmx?WSDL")
    .timeout(Duration.ofSeconds(30))
    .build();

// With auth, TLS, MTOM
SoapClient secure = ServiceClient.soap("payment-gateway")
    .wsdlUrl("https://gateway.example.com/PaymentService?wsdl")
    .credentials("api-user", "secret")
    .enableMtom()
    .trustStore("/certs/truststore.jks", "changeit")
    .keyStore("/certs/keystore.jks", "changeit")
    .enableSchemaValidation()
    .circuitBreakerManager(cbManager)
    .build();

// Multi-service WSDL with QName selection
SoapClient custom = ServiceClient.soap("custom-service")
    .wsdlUrl("http://example.com/service?wsdl")
    .serviceName(new QName("http://example.com/", "CustomService"))
    .portName(new QName("http://example.com/", "CustomPort"))
    .endpointAddress("https://prod.example.com/service")
    .build();
```

Key builder methods: `.wsdlUrl()` (required, auto-extracts credentials from URL),
`.serviceName(QName)`, `.portName(QName)`, `.timeout()`, `.credentials()`,
`.enableMtom()`, `.trustStore()`, `.keyStore()`, `.disableSslVerification()` (dev
only), `.endpointAddress()`, `.enableSchemaValidation()`, `.header()`, `.property()`.

### 4b. Invoke operations

```java
// Fluent builder
Mono<WeatherResponse> weather = soapClient.invoke("GetWeatherByCity")
    .withParameter("city", "New York")
    .withParameter("country", "US")
    .withTimeout(Duration.ofSeconds(10))
    .execute(WeatherResponse.class);

// Request object (e.g., JAXB generated)
Mono<WeatherResponse> weather2 = soapClient.invokeAsync(
    "GetWeatherByCity", request, WeatherResponse.class);

// WSDL introspection
List<String> ops = soapClient.getOperations();
WeatherPort port = soapClient.getPort(WeatherPort.class);
```

---

## Step 5 -- GraphQL Client

```java
import org.fireflyframework.client.graphql.GraphQLClientHelper;
import org.fireflyframework.client.graphql.GraphQLClientHelper.GraphQLConfig;

GraphQLClientHelper graphql = new GraphQLClientHelper(
    "https://api.example.com/graphql",
    GraphQLConfig.builder()
        .timeout(Duration.ofSeconds(60))
        .enableRetry(true).maxRetries(3)
        .enableQueryCache(true)
        .defaultHeader("Authorization", "Bearer " + token)
        .build()
);

// Query with data extraction
Mono<User> user = graphql.query(
    "query GetUser($id: ID!) { user(id: $id) { id name email } }",
    Map.of("id", "123"), "user", User.class);

// Mutation
Mono<User> created = graphql.mutate(mutation, variables, "createUser", User.class);

// Batch
Flux<GraphQLClientHelper.GraphQLResponse<Object>> batch = graphql.executeBatch(List.of(
    GraphQLClientHelper.GraphQLRequest.builder().query("query { users { id } }").build(),
    GraphQLClientHelper.GraphQLRequest.builder().query("query { roles { id } }").build()));
```

---

## Step 6 -- WebSocket Client

```java
import org.fireflyframework.client.websocket.WebSocketClientHelper;
import org.fireflyframework.client.websocket.WebSocketClientHelper.WebSocketConfig;

WebSocketConfig config = WebSocketConfig.builder()
    .enableReconnection(true).maxReconnectAttempts(10)
    .reconnectBackoff(Duration.ofSeconds(2))
    .enableHeartbeat(true).heartbeatInterval(Duration.ofSeconds(30))
    .enableMessageQueue(true).maxQueueSize(1000)
    .header("Authorization", "Bearer " + token)
    .build();

WebSocketClientHelper ws = new WebSocketClientHelper("wss://api.example.com/ws", config);

// Receive with auto-reconnect
ws.receiveMessagesWithReconnection(msg -> processMessage(msg)).subscribe();

// Bidirectional
ws.bidirectional(Flux.just("subscribe:prices"), msg -> log.info(msg)).subscribe();

// Connection pooling
WebSocketClientHelper pooled = WebSocketClientHelper.getPooledConnection(url, config);
```

---

## Step 7 -- Circuit Breaker

`o.f.resilience.CircuitBreakerManager` manages per-service circuit breakers using a
sliding-window failure rate algorithm.

```java
import org.fireflyframework.resilience.CircuitBreakerManager;
import org.fireflyframework.resilience.CircuitBreakerConfig;

CircuitBreakerConfig config = CircuitBreakerConfig.builder()
    .failureRateThreshold(30.0)
    .minimumNumberOfCalls(3).slidingWindowSize(5)
    .waitDurationInOpenState(Duration.ofSeconds(30))
    .permittedNumberOfCallsInHalfOpenState(2)
    .callTimeout(Duration.ofSeconds(5))
    .build();

// Pre-built profiles
CircuitBreakerConfig ha = CircuitBreakerConfig.highAvailabilityConfig();
CircuitBreakerConfig ft = CircuitBreakerConfig.faultTolerantConfig();

CircuitBreakerManager cbManager = new CircuitBreakerManager(config);

Mono<PaymentResult> result = cbManager.executeWithCircuitBreaker(
    "payment-service",
    () -> paymentClient.post("/payments", PaymentResult.class)
            .withBody(req).execute());

// Monitor
CircuitBreakerState state = cbManager.getState("payment-service");  // CLOSED|OPEN|HALF_OPEN
CircuitBreakerMetrics metrics = cbManager.getMetrics("payment-service");
cbManager.reset("payment-service");
```

---

## Step 8 -- Authentication

### 8a. OAuth2

```java
import org.fireflyframework.client.oauth2.OAuth2ClientHelper;

OAuth2ClientHelper oauth2 = new OAuth2ClientHelper(
    "https://auth.example.com/oauth/token", "client-id", "client-secret",
    OAuth2ClientHelper.OAuth2Config.builder()
        .enableRetry(true).tokenExpirationBuffer(120).build());

// Client Credentials (machine-to-machine)
Mono<String> token = oauth2.getClientCredentialsToken("api.read api.write");

token.flatMap(t -> client.get("/resource", Resource.class)
    .withHeader("Authorization", "Bearer " + t).execute());

// Password Grant (legacy -- deprecated in OAuth 2.1)
Mono<String> userToken = oauth2.getPasswordGrantToken("user", "pass", "profile");

// Refresh
Mono<String> refreshed = oauth2.refreshToken(refreshTokenStr);
Mono<String> auto = oauth2.autoRefreshToken(); // uses cached refresh token
```

### 8b. API Keys

```java
import org.fireflyframework.client.security.ApiKeyManager;

// Static
ApiKeyManager key = ApiKeyManager.simple("svc", "my-api-key-12345");

// Dynamic with rotation from vault
ApiKeyManager dynamic = ApiKeyManager.builder()
    .serviceName("svc").apiKeySupplier(() -> fetchFromVault())
    .rotationInterval(Duration.ofHours(1)).autoRotate(true)
    .headerName("X-API-Key").build();

// Bearer convenience
ApiKeyManager bearer = ApiKeyManager.bearer("svc", "my-token");

// Apply
client.get("/data", Data.class)
    .withHeader(key.getHeaderName(), key.getHeaderValue()).execute();
```

---

## Step 9 -- Service Discovery and Load Balancing

```java
import org.fireflyframework.client.discovery.ServiceDiscoveryClient;

ServiceDiscoveryClient discovery = ServiceDiscoveryClient.kubernetes("default");
// Also: .eureka(url), .consul(url), .staticConfig(Map<String, List<String>>)

Mono<String> endpoint = discovery.resolveEndpoint("user-service");
Flux<ServiceDiscoveryClient.ServiceInstance> healthy =
    discovery.getInstances("user-service").filter(ServiceDiscoveryClient.ServiceInstance::isHealthy);
```

Load balancer strategies (`o.f.client.loadbalancer.LoadBalancerStrategy`):

```java
LoadBalancerStrategy lb = new LoadBalancerStrategy.RoundRobin();
// Also: .Random(), .LeastConnections(), .ZoneAware(zone, fallback), .StickySession(fallback)

discovery.getInstances("user-service").collectList()
    .map(lb::selectInstance)
    .flatMap(opt -> opt.map(i -> call(i.getUri()))
        .orElse(Mono.error(new ServiceUnavailableException("No instances"))));
```

---

## Step 10 -- Rate Limiting and Deduplication

### Rate limiting

```java
import org.fireflyframework.client.security.ClientSideRateLimiter;

ClientSideRateLimiter rl = ClientSideRateLimiter.builder()
    .serviceName("payment-service")
    .maxRequestsPerSecond(50.0).maxConcurrentRequests(10)
    .strategy(ClientSideRateLimiter.RateLimitStrategy.TOKEN_BUCKET)
    .build();

if (rl.tryAcquire()) {
    return client.post("/pay", Result.class).withBody(req).execute()
        .doFinally(s -> rl.release());
}
```

### Deduplication

```java
import org.fireflyframework.client.deduplication.RequestDeduplicationManager;

RequestDeduplicationManager dedup = new RequestDeduplicationManager(Duration.ofMinutes(5));
String key = dedup.generateFingerprint("POST", "/payments", request);

Mono<Result> result = dedup.executeWithDeduplication(key,
    client.post("/payments", Result.class).withBody(request)
        .withHeader("Idempotency-Key", key).execute());
```

---

## Step 11 -- Interceptors

```java
import org.fireflyframework.client.interceptor.*;

public class CorrelationIdInterceptor implements ServiceClientInterceptor {
    @Override
    public Mono<InterceptorResponse> intercept(InterceptorRequest req, InterceptorChain chain) {
        String cid = MDC.get("correlationId");
        if (cid != null) req.getHeaders().put("X-Correlation-Id", cid);
        return chain.proceed(req);
    }
    @Override public int getOrder() { return 10; }
}
```

Built-in: `LoggingInterceptor`, `MetricsInterceptor`, `RequestResponseLoggingInterceptor`,
`HttpCacheInterceptor` (`o.f.client.cache`), `ChaosEngineeringInterceptor` (test only).

---

## Configuration Reference

All properties under `firefly.service-client`, bound by `ServiceClientProperties`.
Environment (`DEVELOPMENT`/`TESTING`/`PRODUCTION`) applies automatic defaults.

```yaml
firefly:
  service-client:
    enabled: true
    default-timeout: 30s
    environment: PRODUCTION
    rest:
      max-connections: 200           # dev=50, test=20, prod=200
      response-timeout: 30s
      connect-timeout: 10s
      compression-enabled: true
      logging-enabled: false         # true in dev/test
      max-retries: 3
    grpc:
      call-timeout: 30s
      use-plaintext-by-default: false  # true in dev
      max-inbound-message-size: 4194304
      max-concurrent-streams: 200
    soap:
      default-timeout: 30s
      mtom-enabled: false
      schema-validation-enabled: true
      wsdl-cache-enabled: true
      soap-version: "1.1"
    circuit-breaker:
      enabled: true
      failure-rate-threshold: 50.0
      sliding-window-size: 10
      minimum-number-of-calls: 5
      wait-duration-in-open-state: 60s
      permitted-number-of-calls-in-half-open-state: 3
      call-timeout: 30s
    retry:
      enabled: true
      max-attempts: 3
      wait-duration: 500ms
      exponential-backoff-multiplier: 2.0
      jitter-enabled: true
    metrics:
      enabled: true
      histogram-enabled: true
    security:
      default-auth-type: NONE       # NONE, BASIC, BEARER, OAUTH2, CUSTOM
      ssl-validation-enabled: true
```

---

## Full Example -- Payment Gateway Adapter

```java
@Service @Slf4j
public class PaymentGatewayAdapter {
    private final RestClient paymentClient;
    private final OAuth2ClientHelper oauth2;
    private final ClientSideRateLimiter rateLimiter;
    private final RequestDeduplicationManager dedup;

    public PaymentGatewayAdapter(CircuitBreakerManager cbManager) {
        this.oauth2 = new OAuth2ClientHelper("https://auth.payments.com/oauth/token",
            "client-id", "client-secret",
            OAuth2ClientHelper.OAuth2Config.builder().enableRetry(true).tokenExpirationBuffer(120).build());

        this.paymentClient = ServiceClient.rest("payment-gateway")
            .baseUrl("https://api.payments.com/v2").timeout(Duration.ofSeconds(15))
            .maxConnections(50).circuitBreakerManager(cbManager)
            .retry(true, 3, Duration.ofMillis(500), Duration.ofSeconds(5), true).build();

        this.rateLimiter = ClientSideRateLimiter.builder()
            .serviceName("payment-gateway").maxRequestsPerSecond(100.0)
            .maxConcurrentRequests(20)
            .strategy(ClientSideRateLimiter.RateLimitStrategy.TOKEN_BUCKET).build();

        this.dedup = new RequestDeduplicationManager(Duration.ofMinutes(10));
    }

    public Mono<PaymentResult> processPayment(PaymentRequest request) {
        String key = dedup.generateFingerprint("POST", "/payments", request.getTransactionId());
        if (!rateLimiter.tryAcquire()) {
            return Mono.error(new RateLimitExceededException("Rate limit"));
        }
        return dedup.executeWithDeduplication(key,
            oauth2.getClientCredentialsToken("payments.write")
                .flatMap(token -> paymentClient.post("/payments", PaymentResult.class)
                    .withBody(request)
                    .withHeader("Authorization", "Bearer " + token)
                    .withHeader("Idempotency-Key", key)
                    .execute()))
            .doFinally(s -> rateLimiter.release());
    }
}
```

---

## Anti-Patterns

**1. Blocking in reactive chains** -- Never call `.block()` in production reactive code.
Stay reactive end-to-end.

**2. Creating clients per request** -- Build clients once in constructors or
`@PostConstruct`. Each `.build()` creates connection pools and circuit breakers.

**3. Unbounded retry** -- Use `Retry.backoff(3, Duration.ofSeconds(1))` instead of
`.retry(100)`. Let the circuit breaker manage sustained failures.

**4. Hardcoded credentials** -- Use `OAuth2ClientHelper` or `ApiKeyManager` with vault
integration instead of `.defaultHeader("Authorization", "Bearer hardcoded-token")`.

**5. SSL disabled in production** -- `.disableSslVerification()` is only for dev/test.
In production use `.trustStore()`.

**6. Leaking rate limiter permits** -- Always call `rateLimiter.release()` in
`.doFinally()` to handle both success and error paths.

**7. Missing gRPC stub factory** -- `GrpcClientBuilder.build()` throws
`IllegalStateException` if `.stubFactory()` is not set.

---

## Key Classes Quick Reference

| Class | Package (o.f. = org.fireflyframework) | Role |
|---|---|---|
| `ServiceClient` | `o.f.client` | Root interface, static factory |
| `RestClient` | `o.f.client` | HTTP verbs + streaming |
| `GrpcClient<T>` | `o.f.client` | Stub access, unary/streaming |
| `SoapClient` | `o.f.client` | SOAP invoke, WSDL introspection |
| `RequestBuilder<R>` | `o.f.client` | Fluent path/query/header/body |
| `RestClientBuilder` | `o.f.client.builder` | REST client builder |
| `GrpcClientBuilder<T>` | `o.f.client.builder` | gRPC client builder |
| `SoapClientBuilder` | `o.f.client.builder` | SOAP client builder |
| `ServiceClientAutoConfiguration` | `o.f.client.config` | Spring Boot auto-config |
| `ServiceClientProperties` | `o.f.client.config` | `@ConfigurationProperties` |
| `CircuitBreakerManager` | `o.f.resilience` | Per-service CB management |
| `CircuitBreakerConfig` | `o.f.resilience` | CB thresholds/windows |
| `OAuth2ClientHelper` | `o.f.client.oauth2` | Token lifecycle |
| `GraphQLClientHelper` | `o.f.client.graphql` | Query/mutation/batch |
| `WebSocketClientHelper` | `o.f.client.websocket` | Reconnect/heartbeat/queue |
| `ApiKeyManager` | `o.f.client.security` | Key rotation/headers |
| `ClientSideRateLimiter` | `o.f.client.security` | Rate limiting |
| `RequestDeduplicationManager` | `o.f.client.deduplication` | Idempotency |
| `ServiceDiscoveryClient` | `o.f.client.discovery` | K8s/Eureka/Consul/static |
| `LoadBalancerStrategy` | `o.f.client.loadbalancer` | LB algorithms |
| `ServiceClientInterceptor` | `o.f.client.interceptor` | Cross-cutting hooks |
