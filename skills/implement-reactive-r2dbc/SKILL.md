---
name: implement-reactive-r2dbc
description: Use when creating R2DBC entities, reactive repositories, or Flyway migrations in a Firefly Framework service — covers entity annotations, reactive repository patterns, Flyway migrations, connection pooling, and naming conventions
---

# Implement Reactive R2DBC

## When to Use

Use this skill when you need to define R2DBC entities, create reactive repositories, write Flyway migrations, configure connection pooling and transactions, or implement filtered paginated queries using the Firefly R2DBC module (`fireflyframework-r2dbc`). All operations return `Mono<T>` or `Flux<T>` -- blocking calls are never permitted.

## Dependencies

Add `fireflyframework-r2dbc` to your service POM. The BOM manages the version:

```xml
<dependency>
    <groupId>org.fireflyframework</groupId>
    <artifactId>fireflyframework-r2dbc</artifactId>
</dependency>
```

This transitively brings in `spring-boot-starter-data-r2dbc`, `r2dbc-postgresql`, `flyway-core`, `flyway-database-postgresql`, `springdoc-openapi-starter-webflux-ui`, `fireflyframework-utils` (shared annotations like `@FilterableId`), `mapstruct`, and `lombok`.

## Step 1 -- Define the Entity

Entities use Spring Data R2DBC annotations -- not JPA. Never use `@Entity`, `@GeneratedValue`, or `@ManyToOne` from `jakarta.persistence`.

```java
package com.example.account.domain;

import lombok.*;
import org.springframework.data.annotation.*;
import org.springframework.data.relational.core.mapping.Table;
import org.springframework.data.relational.core.mapping.Column;
import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.UUID;

@Data @Builder @NoArgsConstructor @AllArgsConstructor
@Table("accounts")
public class AccountEntity {
    @Id
    private UUID id;
    @Column("customer_id")
    private UUID customerId;
    @Column("account_number")
    private String accountNumber;
    @Column("account_type")
    private String accountType;
    private BigDecimal balance;
    private String status;
    @CreatedDate
    @Column("created_at")
    private LocalDateTime createdAt;
    @LastModifiedDate
    @Column("updated_at")
    private LocalDateTime updatedAt;
    @Version
    private Long version;
}
```

### Entity annotation reference

| Annotation | Package | Purpose |
|---|---|---|
| `@Id` | `org.springframework.data.annotation` | Marks the primary key field |
| `@Table("name")` | `org.springframework.data.relational.core.mapping` | Maps entity to a database table |
| `@Column("name")` | `org.springframework.data.relational.core.mapping` | Maps field to a column when names differ |
| `@CreatedDate` | `org.springframework.data.annotation` | Auto-populated on insert (requires `@EnableR2dbcAuditing`) |
| `@LastModifiedDate` | `org.springframework.data.annotation` | Auto-populated on insert and update (requires `@EnableR2dbcAuditing`) |
| `@Version` | `org.springframework.data.annotation` | Optimistic locking counter |
| `@Transient` | `org.springframework.data.annotation` | Excludes field from persistence |

### Naming conventions

- **Tables**: lowercase, plural, snake_case -- `accounts`, `transaction_logs`
- **Columns**: lowercase, snake_case -- `customer_id`, `account_number`
- **Entity classes**: PascalCase, singular, suffix `Entity` -- `AccountEntity`
- **Fields**: camelCase -- `customerId`, `accountNumber`
- Use `@Column("snake_case_name")` when Java camelCase differs from column snake_case

### Enabling auditing

`@CreatedDate` and `@LastModifiedDate` require `@EnableR2dbcAuditing`:

```java
@Configuration
@EnableR2dbcAuditing
public class R2dbcAuditingConfig {
}
```

## Step 2 -- Create the Repository

### Option A -- ReactiveCrudRepository

For standard CRUD, extend `ReactiveCrudRepository<T, ID>`:

```java
package com.example.account.repository;

import com.example.account.domain.AccountEntity;
import org.springframework.data.repository.reactive.ReactiveCrudRepository;
import reactor.core.publisher.Flux;
import reactor.core.publisher.Mono;
import java.util.UUID;

public interface AccountRepository extends ReactiveCrudRepository<AccountEntity, UUID> {
    Mono<AccountEntity> findByAccountNumber(String accountNumber);
    Flux<AccountEntity> findByCustomerId(UUID customerId);
    Flux<AccountEntity> findByStatus(String status);
    Mono<Boolean> existsByAccountNumber(String accountNumber);
}
```

All return types must be reactive: `Mono<T>` for single results, `Flux<T>` for collections.

### Option B -- R2dbcEntityTemplate

For complex queries, inject `R2dbcEntityTemplate` and build with `Criteria` and `Query`:

```java
package com.example.account.repository;

import com.example.account.domain.AccountEntity;
import lombok.RequiredArgsConstructor;
import org.springframework.data.r2dbc.core.R2dbcEntityTemplate;
import org.springframework.data.relational.core.query.Criteria;
import org.springframework.data.relational.core.query.Query;
import org.springframework.data.relational.core.query.Update;
import org.springframework.stereotype.Repository;
import reactor.core.publisher.Flux;
import reactor.core.publisher.Mono;
import java.math.BigDecimal;
import java.util.UUID;

@Repository
@RequiredArgsConstructor
public class AccountCustomRepository {
    private final R2dbcEntityTemplate template;

    public Flux<AccountEntity> findActiveWithMinBalance(BigDecimal minBalance) {
        return template.select(AccountEntity.class)
                .matching(Query.query(
                        Criteria.where("status").is("ACTIVE")
                                .and("balance").greaterThanOrEquals(minBalance)))
                .all();
    }

    public Mono<Long> countByStatus(String status) {
        return template.select(AccountEntity.class)
                .matching(Query.query(Criteria.where("status").is(status)))
                .count();
    }

    public Mono<Long> updateBalance(UUID id, BigDecimal newBalance) {
        return template.update(AccountEntity.class)
                .matching(Query.query(Criteria.where("id").is(id)))
                .apply(Update.update("balance", newBalance));
    }
}
```

### Option C -- DatabaseClient for raw SQL

For complex joins, CTEs, or database-specific syntax:

```java
@Repository
@RequiredArgsConstructor
public class AccountNativeRepository {
    private final DatabaseClient databaseClient;

    public Mono<BigDecimal> getTotalBalanceByCustomer(UUID customerId) {
        return databaseClient.sql("""
                SELECT COALESCE(SUM(balance), 0) as total
                FROM accounts
                WHERE customer_id = :customerId AND status = 'ACTIVE'
                """)
                .bind("customerId", customerId)
                .map((row, meta) -> row.get("total", BigDecimal.class))
                .one();
    }
}
```

## Step 3 -- Implement Filtered Pagination with FilterUtils

The framework provides `FilterUtils`, `FilterRequest`, and `PaginationUtils` in `org.fireflyframework.core.filters` and `org.fireflyframework.core.queries` for filtered, paginated queries backed by `R2dbcEntityTemplate`.

### Define a filter class

Mirror the fields you want to filter on. Use `@FilterableId` from `org.fireflyframework.utils.annotations` on ID-like fields that should appear as exact-match filters:

```java
package com.example.account.dto;

import lombok.*;
import org.fireflyframework.utils.annotations.FilterableId;
import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.List;
import java.util.UUID;

@Data @Builder @NoArgsConstructor @AllArgsConstructor
public class AccountFilter {
    @FilterableId
    private UUID customerId;
    private String accountNumber;
    private String accountType;
    private String status;
    private BigDecimal balance;
    private LocalDateTime createdAt;
    private List<String> tags;
}
```

### ID field filtering rules

| Field condition | Exact filter | Range filter |
|---|---|---|
| `@Id` or named `id` | Excluded | Excluded |
| Ends with `Id` without `@FilterableId` | Excluded | Excluded |
| Ends with `Id` with `@FilterableId` | Included | Excluded |
| Normal field (e.g., `status`, `balance`) | Included | Included (if numeric/date) |

### Use FilterUtils in a service

Create a `GenericFilter` with `FilterUtils.createFilter()` -- pass the entity class, a mapper function, and optionally `FilterOptions`:

```java
package com.example.account.service;

import com.example.account.domain.AccountEntity;
import com.example.account.dto.*;
import com.example.account.mapper.AccountMapper;
import org.fireflyframework.core.filters.*;
import org.fireflyframework.core.queries.PaginationResponse;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;
import reactor.core.publisher.Mono;

@Service
@RequiredArgsConstructor
public class AccountQueryService {
    private final AccountMapper accountMapper;

    public Mono<PaginationResponse<AccountDTO>> findAccounts(
            FilterRequest<AccountFilter> filterRequest) {
        FilterUtils.GenericFilter<AccountFilter, AccountEntity, AccountDTO> filter =
                FilterUtils.createFilter(AccountEntity.class, accountMapper::toDto);
        return filter.filter(filterRequest);
    }

    // With case-insensitive string matching
    public Mono<PaginationResponse<AccountDTO>> findCaseInsensitive(
            FilterRequest<AccountFilter> filterRequest) {
        FilterUtils.FilterOptions options = FilterUtils.FilterOptions.builder()
                .caseInsensitiveStrings(true)
                .includeInheritedFields(false)
                .build();
        FilterUtils.GenericFilter<AccountFilter, AccountEntity, AccountDTO> filter =
                FilterUtils.createFilter(AccountEntity.class, accountMapper::toDto, options);
        return filter.filter(filterRequest);
    }
}
```

### Build a FilterRequest

`FilterRequest<T>` combines filter criteria, range filters, pagination, and options:

```java
// Simple filter with pagination
FilterRequest<AccountFilter> request = FilterRequest.<AccountFilter>builder()
        .filters(AccountFilter.builder().status("ACTIVE").accountType("SAVINGS").build())
        .pagination(PaginationRequest.builder()
                .pageNumber(0).pageSize(20).sortBy("createdAt").sortDirection("DESC").build())
        .build();

// Range filters for numeric and date fields
Map<String, RangeFilter.Range<?>> ranges = new HashMap<>();
ranges.put("balance", new RangeFilter.Range<>(new BigDecimal("1000"), new BigDecimal("50000")));
ranges.put("createdAt", new RangeFilter.Range<>(
        LocalDateTime.now().minusDays(30), LocalDateTime.now()));

FilterRequest<AccountFilter> rangeRequest = FilterRequest.<AccountFilter>builder()
        .filters(AccountFilter.builder().status("ACTIVE").build())
        .rangeFilters(RangeFilter.builder().ranges(ranges).build())
        .pagination(new PaginationRequest(0, 10, "balance", "DESC"))
        .build();

// NULL / NOT NULL filtering
AccountFilter filter = AccountFilter.builder().build();
FilterRequest.setNullFilter(filter, "accountType");    // WHERE account_type IS NULL
FilterRequest.setNotNullFilter(filter, "status");       // WHERE status IS NOT NULL
```

### Expose filtering in a controller

Use `@ParameterObject` from SpringDoc. The framework's `FilterParameterCustomizer` auto-generates Swagger parameters:

```java
@RestController
@RequestMapping("/api/v1/accounts")
@RequiredArgsConstructor
public class AccountController {
    private final AccountQueryService accountQueryService;

    @GetMapping
    public Mono<PaginationResponse<AccountDTO>> listAccounts(
            @ParameterObject FilterRequest<AccountFilter> filterRequest) {
        return accountQueryService.findAccounts(filterRequest);
    }
}
```

Generated query parameters follow this pattern:

```
GET /api/v1/accounts?pagination.pageNumber=0&pagination.pageSize=10&
    pagination.sortBy=createdAt&pagination.sortDirection=DESC&
    filters.status=ACTIVE&rangeFilters.ranges[balance].from=1000&
    rangeFilters.ranges[balance].to=5000&options.caseInsensitiveStrings=true
```

### PaginationResponse structure

`PaginationResponse<T>` in `org.fireflyframework.core.queries` contains:
- `content` (List<T>) -- items for the current page
- `totalElements` (long) -- total items across all pages
- `totalPages` (int) -- total number of pages
- `currentPage` (int) -- current page number (zero-based)

### Using PaginationUtils directly

For custom queries where `FilterUtils` does not apply:

```java
public Mono<PaginationResponse<AccountDTO>> findCustom(PaginationRequest paginationRequest) {
    return PaginationUtils.paginateQuery(
            paginationRequest,
            accountMapper::toDto,
            pageable -> template.select(AccountEntity.class)
                    .matching(Query.query(Criteria.where("status").is("ACTIVE")).with(pageable))
                    .all(),
            () -> template.select(AccountEntity.class)
                    .matching(Query.query(Criteria.where("status").is("ACTIVE")))
                    .count()
    );
}
```

## Step 4 -- Write Flyway Migrations

Place SQL migration files in `src/main/resources/db/migration/` with the naming convention:

```
V{version}__{description}.sql       -- versioned (executed once, in order)
R__{description}.sql                 -- repeatable (re-applied when changed)
```

Version format: sequential numbers (`V1`, `V2`) or timestamps (`V20260225120000`). Description uses snake_case. Double underscore separates version from description.

### Initial table creation

```sql
-- V1__create_accounts_table.sql

CREATE TABLE accounts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    customer_id     UUID NOT NULL,
    account_number  VARCHAR(20) NOT NULL UNIQUE,
    account_type    VARCHAR(50) NOT NULL,
    balance         NUMERIC(19, 4) NOT NULL DEFAULT 0,
    status          VARCHAR(20) NOT NULL DEFAULT 'ACTIVE',
    created_at      TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMP NOT NULL DEFAULT NOW(),
    version         BIGINT NOT NULL DEFAULT 0
);

CREATE INDEX idx_accounts_customer_id ON accounts (customer_id);
CREATE INDEX idx_accounts_status ON accounts (status);
```

### Adding columns

```sql
-- V2__add_currency_to_accounts.sql

ALTER TABLE accounts ADD COLUMN currency VARCHAR(3) NOT NULL DEFAULT 'USD';
ALTER TABLE accounts ADD COLUMN branch_id UUID;

CREATE INDEX idx_accounts_branch_id ON accounts (branch_id);
```

### Creating related tables

```sql
-- V3__create_transactions_table.sql

CREATE TABLE transactions (
    id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    account_id       UUID NOT NULL REFERENCES accounts(id),
    amount           NUMERIC(19, 4) NOT NULL,
    transaction_type VARCHAR(50) NOT NULL,
    description      TEXT,
    created_at       TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_transactions_account_id ON transactions (account_id);
CREATE INDEX idx_transactions_created_at ON transactions (created_at);
```

### SQL style conventions

- Uppercase SQL keywords: `CREATE TABLE`, `NOT NULL`, `DEFAULT`
- Lowercase snake_case for table/column names
- `UUID` primary keys with `DEFAULT gen_random_uuid()`
- `NUMERIC(19, 4)` for monetary values -- never `FLOAT`/`DOUBLE`
- `TIMESTAMP` for date-time columns
- `VARCHAR(n)` with explicit length for bounded strings, `TEXT` for unbounded
- Always add `NOT NULL` constraints, `created_at`/`updated_at` columns, and indexes on foreign keys

## Step 5 -- Configure R2DBC Connection and Transactions

### application.yml

```yaml
spring:
  r2dbc:
    url: r2dbc:postgresql://${DB_HOST:localhost}:${DB_PORT:5432}/${DB_NAME:myservice}
    username: ${DB_USERNAME:postgres}
    password: ${DB_PASSWORD:postgres}
    pool:
      enabled: true
      initial-size: ${DB_POOL_INITIAL_SIZE:5}
      max-size: ${DB_POOL_MAX_SIZE:20}
      max-idle-time: ${DB_POOL_MAX_IDLE_TIME:30m}
      max-life-time: ${DB_POOL_MAX_LIFE_TIME:60m}
      validation-query: SELECT 1

  flyway:
    enabled: true
    url: jdbc:postgresql://${DB_HOST:localhost}:${DB_PORT:5432}/${DB_NAME:myservice}
    user: ${DB_USERNAME:postgres}
    password: ${DB_PASSWORD:postgres}
    locations: classpath:db/migration
    baseline-on-migrate: true

logging:
  level:
    org.springframework.data.r2dbc: INFO
    io.r2dbc: INFO
```

Flyway requires a JDBC URL (`jdbc:postgresql://`) because it runs synchronously at startup. R2DBC uses the reactive URL (`r2dbc:postgresql://`).

### Transaction management

The framework auto-configures `ReactiveTransactionManager` via `R2dbcTransactionAutoConfiguration` (`org.fireflyframework.core.config`), which registers an `R2dbcTransactionManager` backed by the `ConnectionFactory` with `@ConditionalOnMissingBean`.

**Declarative** -- use `@Transactional` on service methods:

```java
@Transactional
public Mono<AccountEntity> transferFunds(UUID fromId, UUID toId, BigDecimal amount) {
    return accountRepository.findById(fromId)
            .flatMap(from -> {
                from.setBalance(from.getBalance().subtract(amount));
                return accountRepository.save(from);
            })
            .flatMap(saved -> accountRepository.findById(toId))
            .flatMap(to -> {
                to.setBalance(to.getBalance().add(amount));
                return accountRepository.save(to);
            });
}
```

**Programmatic** -- use `TransactionalOperator`:

```java
TransactionalOperator operator = TransactionalOperator.create(transactionManager);
return accountRepository.save(account).as(operator::transactional);
```

### FilterUtils auto-configuration

The framework's `R2dbcAutoConfiguration` initializes `FilterUtils` with the `R2dbcEntityTemplate` via `@PostConstruct`. You do not need to call `FilterUtils.initializeTemplate()` in service code -- only in unit tests.

## Step 6 -- Map Entities to DTOs

Use MapStruct with `@Mapper(componentModel = MappingConstants.ComponentModel.SPRING)` as abstract classes per Firefly conventions:

```java
@Mapper(componentModel = MappingConstants.ComponentModel.SPRING)
public abstract class AccountMapper {
    public abstract AccountDTO toDto(AccountEntity entity);
    public abstract AccountEntity toEntity(AccountDTO dto);
}
```

## Testing

Use `reactor-test` (`StepVerifier`) and Mockito. In `@BeforeEach`, call `FilterUtils.initializeTemplate(mockTemplate)`:

```java
@ExtendWith(MockitoExtension.class)
class AccountServiceTest {
    @Mock private R2dbcEntityTemplate entityTemplate;
    @Mock private ReactiveSelectOperation.ReactiveSelect<AccountEntity> reactiveSelect;
    @Mock private ReactiveSelectOperation.TerminatingSelect<AccountEntity> terminatingSelect;

    @BeforeEach
    void setUp() {
        FilterUtils.initializeTemplate(entityTemplate);
        when(entityTemplate.select(AccountEntity.class)).thenReturn(reactiveSelect);
        when(reactiveSelect.matching(any(Query.class))).thenReturn(terminatingSelect);
    }

    @Test
    void testFilterByStatus() {
        AccountFilter filter = AccountFilter.builder().status("ACTIVE").build();
        FilterRequest<AccountFilter> request = FilterRequest.<AccountFilter>builder()
                .filters(filter)
                .pagination(new PaginationRequest(0, 10, "createdAt", "DESC"))
                .build();

        when(terminatingSelect.all()).thenReturn(Flux.just(
                AccountEntity.builder().id(UUID.randomUUID()).status("ACTIVE").build()));
        when(terminatingSelect.count()).thenReturn(Mono.just(1L));

        FilterUtils.GenericFilter<AccountFilter, AccountEntity, AccountEntity> gf =
                FilterUtils.createFilter(AccountEntity.class, e -> e);

        StepVerifier.create(gf.filter(request))
                .assertNext(response -> {
                    assertThat(response.getContent()).hasSize(1);
                    assertThat(response.getTotalElements()).isEqualTo(1);
                })
                .verifyComplete();
    }
}
```

## Common Mistakes

1. **Using JPA annotations** -- R2DBC does not support `jakarta.persistence`. Use `@Id` from `org.springframework.data.annotation`, `@Table`/`@Column` from `org.springframework.data.relational.core.mapping`. Using `@Entity`, `@GeneratedValue`, or `@ManyToOne` causes silent failures or errors.

2. **Blocking in repository calls** -- Never call `.block()`. Use `flatMap()`, `map()`, `zipWith()` to compose database calls. The entire chain must remain non-blocking.

3. **Forgetting `@EnableR2dbcAuditing`** -- `@CreatedDate` and `@LastModifiedDate` produce `null` without it.

4. **Missing Flyway JDBC URL** -- Flyway cannot use R2DBC URLs. Configure `spring.flyway.url` with `jdbc:postgresql://` separately.

5. **Not initializing FilterUtils in tests** -- `FilterUtils.createFilter()` throws `IllegalStateException` without `R2dbcEntityTemplate`. The framework handles this automatically in production; in tests, call `FilterUtils.initializeTemplate(template)` in `@BeforeEach`.

6. **Using `@GeneratedValue` for UUIDs** -- Not supported in R2DBC. Use `DEFAULT gen_random_uuid()` in SQL or generate UUIDs in Java before saving.

7. **Returning `Optional` from repositories** -- R2DBC returns `Mono<T>` (empty Mono for no result), not `Optional<T>`. Use `switchIfEmpty()` or `defaultIfEmpty()`.

8. **Modifying applied migrations** -- Flyway records checksums in `flyway_schema_history`. Altering an applied migration causes checksum mismatch errors. Always create a new version.

9. **Omitting indexes on foreign keys** -- PostgreSQL does not auto-create indexes on FK columns. Always add explicit indexes.

10. **Using `Float`/`Double` for money** -- Use `BigDecimal` in Java and `NUMERIC(19, 4)` in PostgreSQL.

11. **Forgetting `@Column` for snake_case** -- Without `@Column("customer_id")`, Spring Data R2DBC looks for `customerid` (lowercase, no underscore).

12. **`@Transactional` on private methods** -- Spring proxies do not intercept private methods. Make transactional methods public, or use `TransactionalOperator` for programmatic control.
