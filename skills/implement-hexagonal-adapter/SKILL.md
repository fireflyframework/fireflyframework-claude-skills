---
name: implement-hexagonal-adapter
description: >
  Use when creating a hexagonal architecture adapter for an existing port in a Firefly Framework
  service. Covers port interface definition, adapter creation with Spring config, and isolated
  adapter tests. References real framework adapter examples (IDP, ECM, Notifications).
---

# Implement a Hexagonal Architecture Adapter in the Firefly Framework

## Overview

The Firefly Framework uses hexagonal architecture across all modules. **Ports** define
technology-agnostic interfaces. **Adapters** implement them against concrete technologies
(Keycloak, S3, SendGrid, etc.). Spring Boot auto-configuration selects the active adapter
at startup.

---

## 1. Real Port/Adapter Patterns in the Framework

### 1.1 IDP Module -- Simple Single-Interface Pattern

**Port:** `org.fireflyframework.idp.adapter.IdpAdapter`

```java
public interface IdpAdapter {
    Mono<ResponseEntity<TokenResponse>> login(LoginRequest request);
    Mono<ResponseEntity<TokenResponse>> refresh(RefreshRequest request);
    Mono<Void> logout(LogoutRequest request);
    Mono<ResponseEntity<IntrospectionResponse>> introspect(String accessToken);
    Mono<ResponseEntity<UserInfoResponse>> getUserInfo(String accessToken);
    Mono<ResponseEntity<CreateUserResponse>> createUser(CreateUserRequest request);
    Mono<Void> changePassword(ChangePasswordRequest request);
    Mono<Void> resetPassword(String username);
    Mono<ResponseEntity<MfaChallengeResponse>> mfaChallenge(String username);
    Mono<Void> mfaVerify(MfaVerifyRequest request);
    Mono<Void> revokeRefreshToken(String refreshToken);
    Mono<ResponseEntity<List<SessionInfo>>> listSessions(String userId);
    Mono<Void> revokeSession(String sessionId);
    Mono<ResponseEntity<List<String>>> getRoles(String userId);
    Mono<Void> deleteUser(String userId);
    Mono<ResponseEntity<UpdateUserResponse>> updateUser(UpdateUserRequest request);
    Mono<ResponseEntity<CreateRolesResponse>> createRoles(CreateRolesRequest request);
    Mono<Void> assignRolesToUser(AssignRolesRequest request);
    Mono<Void> removeRolesFromUser(AssignRolesRequest request);

    default Mono<ResponseEntity<CreateUserResponse>> registerUser(RegisterUserRequest request) {
        return createUser(CreateUserRequest.builder()
                .username(request.getUsername()).email(request.getEmail())
                .password(request.getPassword()).givenName(request.getFirstName())
                .familyName(request.getLastName()).build());
    }
}
```

**Controller injects the port directly:**

```java
@RestController @RequestMapping("/idp") @RequiredArgsConstructor
public class IdpController {
    private final IdpAdapter idpAdapter;

    @PostMapping("/login")
    public Mono<ResponseEntity<TokenResponse>> login(@RequestBody LoginRequest request) {
        return idpAdapter.login(request);
    }
}
```

**Auto-configuration activates only when an adapter bean exists:**

```java
@AutoConfiguration
@ConditionalOnBean(IdpAdapter.class)
@ConditionalOnWebApplication(type = REACTIVE)
@EnableConfigurationProperties(IdpProperties.class)
public class IdpWebAutoConfiguration {
    @Bean @ConditionalOnMissingBean
    public IdpController idpController(IdpAdapter idpAdapter) {
        return new IdpController(idpAdapter);
    }
}
```

**Properties:** `firefly.idp.provider` selects keycloak, cognito, or internal-db.

---

### 1.2 ECM Module -- Multi-Port Registry Pattern

The ECM module defines **16 port interfaces**, a `@EcmAdapter` annotation, `AdapterRegistry`
for discovery, `AdapterSelector` for fallback, and NoOp adapters for graceful degradation.

**Ports** (in `org.fireflyframework.ecm.port.*`):

| Package          | Interface                   | Purpose                 |
|------------------|-----------------------------|-------------------------|
| `port.document`  | `DocumentPort`              | Document CRUD           |
| `port.document`  | `DocumentContentPort`       | Binary content storage  |
| `port.document`  | `DocumentVersionPort`       | Version management      |
| `port.document`  | `DocumentSearchPort`        | Search and indexing     |
| `port.folder`    | `FolderPort`                | Folder CRUD             |
| `port.folder`    | `FolderHierarchyPort`       | Folder tree navigation  |
| `port.security`  | `PermissionPort`            | Access control          |
| `port.security`  | `DocumentSecurityPort`      | Encryption, DRM         |
| `port.audit`     | `AuditPort`                 | Audit trail             |
| `port.esignature`| `SignatureEnvelopePort`      | eSignature envelopes    |
| `port.esignature`| `SignatureRequestPort`       | Signature requests      |
| `port.esignature`| `SignatureValidationPort`    | Signature verification  |
| `port.esignature`| `SignatureProofPort`         | Proof documents         |
| `port.idp`       | `DocumentExtractionPort`     | Text/OCR extraction     |
| `port.idp`       | `DocumentClassificationPort` | Classification          |
| `port.idp`       | `DocumentValidationPort`     | Validation              |

**Example port -- `DocumentPort`:**

```java
public interface DocumentPort {
    Mono<Document> createDocument(Document document, byte[] content);
    Mono<Document> getDocument(UUID documentId);
    Mono<Document> updateDocument(Document document);
    Mono<Void> deleteDocument(UUID documentId);
    Mono<Boolean> existsDocument(UUID documentId);
    Flux<Document> getDocumentsByFolder(UUID folderId);
    Flux<Document> getDocumentsByOwner(UUID ownerId);
    Flux<Document> getDocumentsByStatus(DocumentStatus status);
    Mono<Document> moveDocument(UUID documentId, UUID targetFolderId);
    Mono<Document> copyDocument(UUID documentId, UUID targetFolderId, String newName);
    String getAdapterName();
}
```

**`@EcmAdapter` marks concrete adapters:**

```java
@Target(ElementType.TYPE) @Retention(RetentionPolicy.RUNTIME) @Component
public @interface EcmAdapter {
    String type();                          // "s3", "azure-blob", "local-search"
    int priority() default 0;              // higher = preferred
    boolean enabled() default true;
    String description() default "";
    String[] requiredProperties() default {};
    AdapterFeature[] supportedFeatures() default {};
    AdapterProfile minimumProfile() default AdapterProfile.BASIC;
}
```

**Real adapter -- `LocalDocumentSearchAdapter`:**

```java
@Slf4j
@EcmAdapter(
    type = "local-search",
    description = "Local in-memory DocumentSearchPort adapter",
    supportedFeatures = { AdapterFeature.SEARCH, AdapterFeature.METADATA_SEARCH }
)
public class LocalDocumentSearchAdapter implements DocumentSearchPort {
    private final Map<UUID, Document> index = new ConcurrentHashMap<>();

    @Override
    public Flux<Document> fullTextSearch(String query, Integer limit) {
        String q = query == null ? "" : query.toLowerCase();
        return Flux.fromStream(index.values().stream()
            .filter(d -> contains(d.getName(), q) || contains(d.getDescription(), q))
            .limit(limitOrUnlimited(limit)));
    }
    // ... remaining methods
}
```

**`AdapterRegistry` discovers adapters at startup:**

```java
public class AdapterRegistry {
    private final Map<String, List<AdapterInfo>> adaptersByType = new ConcurrentHashMap<>();
    private final Map<Class<?>, List<AdapterInfo>> adaptersByInterface = new ConcurrentHashMap<>();

    @PostConstruct
    public void initialize() {
        Map<String, Object> adapters = applicationContext.getBeansWithAnnotation(EcmAdapter.class);
        // Registers each adapter by type AND by implemented port interfaces
        // (checks package prefix "org.fireflyframework.ecm.port")
    }

    public <T> Optional<T> getAdapter(Class<T> interfaceClass) { /* priority-based */ }
}
```

**`AdapterSelector` provides fallback logic:**

```java
public <T> Optional<T> selectAdapter(String preferredType, Class<T> interfaceClass) {
    // 1. Try preferred type  2. Verify interface  3. Fallback to any compatible  4. Empty
}
```

**`EcmPortProvider` is the service-layer factory:**

```java
public Optional<DocumentPort> getDocumentPort() {
    return adapterSelector.selectAdapter(ecmProperties.getAdapterType(), DocumentPort.class);
}
```

**Auto-configuration with feature flags and NoOp fallback:**

```java
@AutoConfiguration
@EnableConfigurationProperties(EcmProperties.class)
@ConditionalOnProperty(prefix = "firefly.ecm", name = "enabled", havingValue = "true", matchIfMissing = true)
public class EcmAutoConfiguration {

    @Bean @ConditionalOnMissingBean
    @ConditionalOnProperty(prefix = "firefly.ecm.features", name = "document-management",
                           havingValue = "true", matchIfMissing = true)
    public DocumentPort documentPort(EcmPortProvider portProvider, NoOpAdapterFactory noOp) {
        return portProvider.getDocumentPort()
            .orElseGet(noOp::createDocumentPort);
    }
    // Same pattern for all 16 ports
}
```

**NoOp adapter uses dynamic proxies** -- query methods return `Mono.empty()`, modification
methods return `Mono.error(UnsupportedOperationException)`, permission checks return
`Mono.just(false)` (DENY by default).

**YAML:**
```yaml
firefly:
  ecm:
    enabled: true
    adapter-type: s3
    features:
      document-management: true
      esignature: false
```

---

### 1.3 Notifications Module -- Provider Pattern

**Provider interfaces** (in `org.fireflyframework.notifications.interfaces.providers`):

```java
public interface EmailProvider {
    Mono<EmailResponseDTO> sendEmail(EmailRequestDTO request);
}

public interface SMSProvider {
    Mono<SMSResponseDTO> sendSMS(SMSRequestDTO request);
}

public interface PushProvider {
    Mono<PushNotificationResponse> sendPush(PushNotificationRequest request);
}
```

**Service injects the provider directly:**

```java
@Service @RequiredArgsConstructor
public class SMSServiceImpl implements SMSService {
    private final SMSProvider smsProvider;

    @Override
    public Mono<SMSResponseDTO> sendSMS(SMSRequestDTO request) {
        return smsProvider.sendSMS(request)
            .onErrorResume(error -> Mono.just(SMSResponseDTO.error(error.getMessage())));
    }
}
```

---

## 2. Step-by-Step: Implement a New Adapter

### Step 1: Define the Port Interface

```java
package org.fireflyframework.<module>.port.<domain>;

public interface PaymentGatewayPort {
    Mono<PaymentResult> processPayment(PaymentRequest request);
    Mono<PaymentResult> refundPayment(String transactionId, RefundRequest request);
    Mono<PaymentStatus> getPaymentStatus(String transactionId);
    Flux<PaymentTransaction> listTransactions(TransactionQuery query);
    String getAdapterName();
}
```

**Port rules:** (1) Reactive return types only. (2) Domain DTOs, no SDK types. (3) Include
`getAdapterName()`. (4) Place in `port` package. (5) Javadoc the contract.

### Step 2: Create Configuration Properties

```java
@Data @Validated @ConfigurationProperties(prefix = "firefly.<module>")
public class PaymentProperties {
    @NotBlank private String provider;
    private Map<String, Object> properties;
}
```

### Step 3: Implement the Adapter

**Simple pattern** (like IDP):

```java
@Component
@ConditionalOnProperty(name = "firefly.payment.provider", havingValue = "stripe")
public class StripePaymentAdapter implements PaymentGatewayPort {
    private final StripeClient stripeClient;

    @Override
    public Mono<PaymentResult> processPayment(PaymentRequest request) {
        return stripeClient.createCharge(mapToStripeCharge(request))
            .map(this::mapToPaymentResult);
    }

    @Override
    public String getAdapterName() { return "StripePaymentAdapter"; }
}
```

**ECM registry pattern:**

```java
@EcmAdapter(
    type = "azure-blob",
    description = "Azure Blob Storage adapter for document content",
    requiredProperties = { "account-name", "account-key", "container-name" },
    supportedFeatures = { AdapterFeature.CONTENT_STORAGE, AdapterFeature.STREAMING },
    minimumProfile = AdapterProfile.STANDARD
)
public class AzureBlobContentAdapter implements DocumentContentPort {
    @Override
    public Mono<String> storeContent(UUID documentId, byte[] content, String mimeType) {
        return Mono.fromCallable(() -> {
            blobClient.getBlobContainerClient("documents")
                .getBlobClient(generatePath(documentId)).upload(...);
            return generatePath(documentId);
        }).subscribeOn(Schedulers.boundedElastic());
    }
}
```

**Adapter rules:** (1) One class per port per technology. (2) Use `@ConditionalOnProperty`
or `@EcmAdapter`. (3) Translate SDK types internally. (4) Wrap blocking calls with
`Mono.fromCallable().subscribeOn(boundedElastic())`.

### Step 4: Wire Auto-Configuration

```java
@AutoConfiguration
@ConditionalOnBean(PaymentGatewayPort.class)
@EnableConfigurationProperties(PaymentProperties.class)
public class PaymentAutoConfiguration {
    @Bean @ConditionalOnMissingBean
    public PaymentController paymentController(PaymentGatewayPort gateway) {
        return new PaymentController(gateway);
    }
}
```

Register in `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`:
```
org.fireflyframework.<module>.config.PaymentAutoConfiguration
```

### Step 5: Write Isolated Tests

**Stub adapter test** (from `EmailServiceImplTest`):

```java
@ExtendWith(SpringExtension.class)
@ContextConfiguration(classes = {EmailServiceImpl.class, EmailServiceImplTest.TestBeans.class})
class EmailServiceImplTest {
    @Configuration
    static class TestBeans {
        @Bean EmailProvider emailProvider() {
            return request -> Mono.just(EmailResponseDTO.success("test-message-id"));
        }
    }

    @Autowired private EmailService emailService;

    @Test
    void sendEmail_returnsSuccess() {
        EmailRequestDTO req = EmailRequestDTO.builder()
                .from("noreply@example.com").to("user@example.com")
                .subject("Hello").text("Hi").build();
        EmailResponseDTO resp = emailService.sendEmail(req).block();
        assertThat(resp.getStatus()).isEqualTo(EmailStatusEnum.SENT);
    }
}
```

**NoOp adapter test** (from `NoOpAdapterLoggingTest`):

```java
@Test
void shouldLogWarningsForDocumentAdapterMethods() {
    DocumentPort adapter = new NoOpGenericAdapter<>("DocumentPort", DocumentPort.class).getProxy();
    StepVerifier.create(adapter.getDocument(UUID.randomUUID())).verifyComplete();
    StepVerifier.create(adapter.deleteDocument(UUID.randomUUID()))
        .expectError(UnsupportedOperationException.class).verify();
}
```

**Auto-config test** (from `EcmAutoConfigurationTest`):

```java
@SpringBootTest(classes = EcmAutoConfiguration.class)
@TestPropertySource(properties = {
    "firefly.ecm.enabled=true",
    "firefly.ecm.features.content-storage=true"
})
class EcmAutoConfigurationTest {
    @Autowired private DocumentContentPort documentContentPort;

    @Test
    void contextLoads_andDocumentContentPortBeanPresent() {
        assertThat(documentContentPort).isNotNull();
    }
}
```

**Test strategy summary:**

| Test Type              | Verifies                                | Example                    |
|------------------------|-----------------------------------------|----------------------------|
| Unit (stub adapter)    | Service logic with mocked adapter       | `EmailServiceImplTest`     |
| NoOp adapter           | Fallback behavior, logging, errors      | `NoOpAdapterLoggingTest`   |
| Auto-config            | Bean wiring under property combos       | `EcmAutoConfigurationTest` |
| Integration            | Real adapter against testcontainer      | `@Testcontainers` + WireMock |

---

## 3. Which Pattern to Use

| Scenario                                           | Pattern            | Example Module              |
|----------------------------------------------------|--------------------|-----------------------------|
| Single port, simple provider selection             | IDP pattern        | `fireflyframework-idp`      |
| Many ports, feature flags, NoOp fallbacks          | ECM pattern        | `fireflyframework-ecm`      |
| Outbound integration in existing service layer     | Provider pattern   | `fireflyframework-notifications` |

---

## 4. Package Structure

```
org.fireflyframework.<module>/
  adapter/
    <provider>/<ProviderName>Adapter.java
    noop/NoOpAdapterBase.java, NoOpGenericAdapter.java, NoOpAdapterFactory.java
    AdapterRegistry.java                    # ECM pattern only
    AdapterSelector.java                    # ECM pattern only
    @EcmAdapter.java                        # ECM pattern only
  config/
    <Module>AutoConfiguration.java
    <Module>Properties.java
  port/<subdomain>/<PortName>Port.java
  service/<ServiceName>ServiceImpl.java
  web/<Controller>.java
```

---

## 5. Anti-Patterns

**1. Leaking infrastructure types through the port:**
```java
// BAD:  Mono<PutObjectResponse> storeContent(UUID id, PutObjectRequest req);
// GOOD: Mono<String> storeContent(UUID id, byte[] content, String mimeType);
```

**2. Business logic in the adapter** -- validation and rules belong in the service layer,
not the adapter. The adapter only translates between domain DTOs and SDK calls.

**3. Depending on a concrete adapter class:**
```java
// BAD:  private final S3ContentAdapter s3Adapter;
// GOOD: private final DocumentContentPort contentPort;
```

**4. God adapter implementing multiple ports** -- use one adapter class per port per
technology. `S3DocumentAdapter implements DocumentPort` and `S3ContentAdapter implements
DocumentContentPort` as separate classes.

**5. Blocking calls without `Schedulers.boundedElastic()`:**
```java
// BAD:  return Mono.just(blockingSdk.upload(content));
// GOOD: return Mono.fromCallable(() -> blockingSdk.upload(content))
//            .subscribeOn(Schedulers.boundedElastic());
```

**6. Skipping `getAdapterName()`** -- every port should declare it, every adapter must
implement it. Critical for runtime logging and diagnostics.

**7. Permissive NoOp security adapters** -- the framework's `NoOpGenericAdapter` returns
`false` (DENY) for all permission checks. Never return `true` in a NoOp security adapter.

---

## 6. Checklist

- [ ] Port in `port` sub-package, reactive types, domain DTOs only, includes `getAdapterName()`
- [ ] Adapter in `adapter.<provider>`, uses `@ConditionalOnProperty` or `@EcmAdapter`
- [ ] Blocking SDK calls wrapped with `Mono.fromCallable().subscribeOn(boundedElastic())`
- [ ] No business logic or validation in the adapter
- [ ] Auto-config registered in `META-INF/spring/...AutoConfiguration.imports`
- [ ] All beans use `@ConditionalOnMissingBean`
- [ ] Feature flags gate optional ports via `@ConditionalOnProperty`
- [ ] NoOp fallback for graceful degradation (ECM pattern)
- [ ] Unit test with stub adapter via `@ContextConfiguration`
- [ ] Reactive assertions with `StepVerifier` or `.block()`
- [ ] YAML configuration documented
- [ ] Javadoc on port interface
