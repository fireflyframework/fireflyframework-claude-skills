---
name: implement-rule-engine
description: Use when implementing business rules with the Firefly rule engine — covers YAML DSL for rule definition, AST parsing, compilation, batch evaluation, audit trails, and the rule engine multi-module structure
---

# Implementing Business Rules with the Firefly Rule Engine

## Overview

The Firefly Rule Engine is a YAML DSL-based system for defining, parsing, validating, evaluating, and auditing business rules. It uses AST processing with a visitor pattern, reactive APIs via Project Reactor, and supports Python compilation for offline execution.

## Multi-Module Structure

Five Maven modules under `org.fireflyframework:fireflyframework-rule-engine`:

| Module | Artifact ID | Purpose |
|--------|------------|---------|
| **interfaces** | `fireflyframework-rule-engine-interfaces` | DTOs, enums, validation annotations |
| **models** | `fireflyframework-rule-engine-models` | R2DBC entities (`RuleDefinition`, `AuditTrail`, `Constant`) and repositories |
| **core** | `fireflyframework-rule-engine-core` | DSL parser, AST nodes, evaluation engine, validators, services |
| **web** | `fireflyframework-rule-engine-web` | Spring WebFlux REST controllers |
| **sdk** | `fireflyframework-rule-engine-sdk` | Generated client SDK from OpenAPI spec |

Dependency flow: `web` -> `core` -> `models` -> `interfaces`.

---

## Step 1: Define Rules in YAML DSL

### Naming Conventions (Enforced by Validation)

- **Input variables**: `camelCase` -- `creditScore`, `annualIncome`, `employmentYears`
- **Computed variables**: `snake_case` -- `debt_to_income`, `is_eligible`, `credit_tier`
- **Constants**: `UPPER_CASE` -- `MIN_CREDIT_SCORE`, `MAX_LOAN_AMOUNT`

### Simple Syntax (when/then/else)

```yaml
name: "Credit Assessment"
description: "Basic credit eligibility check"
version: "1.0.0"

inputs:
  creditScore: "number"
  annualIncome: "number"
  existingDebt: "number"

constants:
  - code: MIN_CREDIT_SCORE
    defaultValue: 650

when:
  - creditScore at_least MIN_CREDIT_SCORE
  - annualIncome at_least 50000

then:
  - calculate debt_to_income as existingDebt / annualIncome
  - set credit_tier to "PRIME"
  - set is_eligible to true

else:
  - set credit_tier to "STANDARD"
  - set is_eligible to false

output:
  credit_tier: credit_tier
  is_eligible: is_eligible
  debt_to_income: debt_to_income
```

### Multiple Rules Syntax

```yaml
name: "Loan Processing Pipeline"
rules:
  - name: "Financial Validation"
    when:
      - annualIncome at_least 30000
      - creditScore at_least 600
    then:
      - calculate debt_ratio as existingDebt / annualIncome
      - set financial_check to "PASSED"
    else:
      - set financial_check to "FAILED"

  - name: "Risk Classification"
    when:
      - creditScore at_least 750
    then:
      - set risk_level to "LOW"
      - calculate max_loan as annualIncome * 5
    else:
      - set risk_level to "HIGH"
      - calculate max_loan as annualIncome * 2
```

### Complex Conditions Syntax (Nested if/then/else)

```yaml
name: "Tiered Pricing"
conditions:
  if: "creditScore at_least 750"
  then:
    actions:
      - set tier to "PREMIUM"
    conditions:
      if: "loanAmount at_least 500000"
      then:
        actions:
          - set interest_rate to 3.0
      else:
        actions:
          - set interest_rate to 3.5
  else:
    actions:
      - set interest_rate to 6.0
      - set tier to "STANDARD"
```

### Comparison Operators (`ComparisonOperator`)

| Keyword | Description | Keyword | Description |
|---------|-------------|---------|-------------|
| `at_least` / `>=` | Greater or equal | `at_most` / `<=` | Less or equal |
| `equals` / `==` | Equality | `not_equals` / `!=` | Inequality |
| `greater_than` / `>` | Strictly greater | `less_than` / `<` | Strictly less |
| `between` | Range check | `in` / `in_list` | Membership |
| `contains` | Contains | `starts_with` | Prefix |
| `matches` | Regex | `is_null` / `is_not_null` | Null checks |
| `is_email` | Email format | `is_phone` | Phone format |
| `is_positive` | Positive number | `is_credit_score` | Valid credit score |

Logical operators via `LogicalOperator`: `and` (`&&`), `or` (`||`), `not` (`!`).

### Action Types

| Action | Syntax | Example |
|--------|--------|---------|
| Set | `set <var> to <value>` | `set is_eligible to true` |
| Calculate | `calculate <var> as <expr>` | `calculate debt_ratio as debt / income` |
| Arithmetic | `add <val> to <var>` | `add 10 to score` |
| Function call | `call <fn> with [args]` | `call round with [score, 2]` |
| forEach | `forEach item in items: <action>` | `forEach p in payments: add p to total` |
| while | `while <cond>: <action>` | `while counter < 10: add 1 to counter` |
| Circuit breaker | `stop with "<message>"` | `stop with "High risk detected"` |

---

## Step 2: Parse Rules with ASTRulesDSLParser

`ASTRulesDSLParser` (`org.fireflyframework.rules.core.dsl.parser`) converts YAML to `ASTRulesDSL` model objects. Sub-parsers: `DSLParser`, `ConditionParser`, `ActionParser`, `ExpressionParser`, `ComplexConditionsParser`.

```java
@Autowired
private ASTRulesDSLParser parser;

Mono<ASTRulesDSL> ruleMono = parser.parseRulesReactive(yamlString);  // Preferred
ASTRulesDSL rule = parser.parseRules(yamlString);                     // Deprecated sync

// ASTRulesDSL model API
rule.isSimpleSyntax();            // when/then/else
rule.isMultipleRulesSyntax();     // rules: [...]
rule.isComplexConditionsSyntax(); // conditions: { if: ... }
rule.getWhenConditions();         // List<Condition>
rule.getThenActions();            // List<Action>
rule.getRules();                  // List<ASTSubRule>
rule.getConditions();             // ASTConditionalBlock
rule.getConstants();              // List<ASTConstantDefinition>
rule.getCircuitBreaker();         // ASTCircuitBreakerConfig

// Cache management
parser.invalidateCache(yamlString);
parser.clearCache();
```

---

## Step 3: Validate Rules with YamlDslValidator

```java
@Autowired
private YamlDslValidator yamlDslValidator;

ValidationResult result = yamlDslValidator.validate(yamlContent);
result.getStatus();                    // VALID, WARNING, ERROR, CRITICAL_ERROR
result.isValid();
result.getSummary().getQualityScore();
result.getIssues().getSyntax();        // Syntax errors
result.getIssues().getNaming();        // Naming convention violations
result.getIssues().getDependencies();  // Undefined variable references
```

REST: `POST /api/v1/validation/yaml` (full) and `GET /api/v1/validation/syntax?yaml=...` (quick).

---

## Step 4: Evaluate Rules with ASTRulesEvaluationEngine

The engine (`org.fireflyframework.rules.core.dsl.evaluation`) uses `ASTVisitor<T>`, `ExpressionEvaluator`, and `ActionExecutor`.

```java
@Autowired
private ASTRulesEvaluationEngine evaluationEngine;

Map<String, Object> inputData = Map.of("creditScore", 720);

// Reactive (preferred)
Mono<ASTRulesEvaluationResult> resultMono =
    evaluationEngine.evaluateRulesReactive(yamlRule, inputData);

// Synchronous (tests only)
ASTRulesEvaluationResult result = evaluationEngine.evaluateRules(yamlRule, inputData);

result.isSuccess();
result.isConditionMet();
result.getOutputData();              // Map<String, Object>
result.getOutputValue("approval");   // Specific output
result.getExecutionTimeMs();
result.isCircuitBreakerTriggered();
result.getCircuitBreakerMessage();
```

### EvaluationContext Variable Resolution

`EvaluationContext` resolves variables in priority order: computed (`snake_case`) > input (`camelCase`) > constants (`UPPER_CASE`). Constants are auto-detected from `UPPER_CASE` references in the AST and loaded from the database via `ConstantService`.

### REST API Endpoints

```
POST /api/v1/rules/evaluate/direct   -- Base64-encoded YAML (RulesEvaluationRequestDTO)
POST /api/v1/rules/evaluate/plain    -- Plain YAML (PlainYamlEvaluationRequestDTO)
POST /api/v1/rules/evaluate/by-code  -- Stored rule code (RuleEvaluationByCodeRequestDTO)
```

All return `RulesEvaluationResponseDTO` with: `success`, `conditionResult`, `outputData`, `circuitBreakerTriggered`, `error`, `executionTimeMs`.

---

## Step 5: Batch Evaluation

`BatchRulesEvaluationService` evaluates multiple stored rules concurrently.

```java
BatchRulesEvaluationRequestDTO request = BatchRulesEvaluationRequestDTO.builder()
    .evaluationRequests(List.of(
        BatchRulesEvaluationRequestDTO.SingleRuleEvaluationRequest.builder()
            .requestId("req-001")
            .ruleDefinitionCode("LOAN_APPROVAL")
            .inputData(Map.of("creditScore", 750))
            .priority(1).continueOnError(true).build()))
    .globalInputData(Map.of("userId", "user123"))
    .batchOptions(BatchRulesEvaluationRequestDTO.BatchOptions.builder()
        .maxConcurrency(10).timeoutSeconds(300)
        .failFast(false).enableCaching(true).build())
    .build();
```

REST: `POST /api/v1/rules/batch/evaluate`, `POST /api/v1/rules/batch/validate`, `GET /api/v1/rules/batch/statistics`, `GET /api/v1/rules/batch/health`.

Response includes `BatchStatus` (SUCCESS, PARTIAL_SUCCESS, FAILED, CANCELLED), per-request results, and `BatchSummary` with `successRate`, `averageProcessingTimeMs`, `cacheHitRate`.

---

## Step 6: Store and Manage Rule Definitions

```java
@Autowired
private RuleDefinitionService ruleDefinitionService;

RuleDefinitionDTO dto = RuleDefinitionDTO.builder()
    .code("credit_scoring_v1")        // Pattern: ^[a-zA-Z][a-zA-Z0-9_]*$
    .name("Credit Scoring Rule v1")
    .yamlContent(yamlString)
    .version("1.0.0")                 // Semver format
    .isActive(true)
    .tags("credit,scoring,loan")
    .build();

Mono<RuleDefinitionDTO> created = ruleDefinitionService.createRuleDefinition(dto);
Mono<RuleDefinitionDTO> rule = ruleDefinitionService.getRuleDefinitionByCode("credit_scoring_v1");
Mono<ASTRulesEvaluationResult> result = ruleDefinitionService.evaluateRuleByCode(
    "credit_scoring_v1", Map.of("creditScore", 720));
Mono<ValidationResult> validation = ruleDefinitionService.validateRuleDefinition(yamlContent);
```

---

## Step 7: Audit Trails

Every operation is audited via `AuditTrailService` with `AuditEventType`:

| Event Type | Entity | Event Type | Entity |
|-----------|--------|-----------|--------|
| `RULE_DEFINITION_CREATE` | RULE_DEFINITION | `RULE_EVALUATION_DIRECT` | RULE_EVALUATION |
| `RULE_DEFINITION_UPDATE` | RULE_DEFINITION | `RULE_EVALUATION_PLAIN` | RULE_EVALUATION |
| `RULE_DEFINITION_DELETE` | RULE_DEFINITION | `RULE_EVALUATION_BY_CODE` | RULE_EVALUATION |
| `YAML_VALIDATION` | VALIDATION | `RULE_EVALUATION_BATCH` | RULE_EVALUATION |

`AuditTrailDTO` captures: `operationType`, `entityId`, `ruleCode`, `userId`, `ipAddress`, `httpMethod`, `endpoint`, `requestData`, `responseData`, `statusCode`, `success`, `executionTimeMs`, `correlationId`, `createdAt`.

```java
Mono<PaginationResponse<AuditTrailDTO>> audits = auditTrailService.getAuditTrails(filterDTO);
Mono<Map<String, Object>> stats = auditTrailService.getAuditTrailStatistics();
Mono<Long> deleted = auditTrailService.deleteOldAuditTrails(90); // retention days
```

---

## Step 8: Python Compilation (Optional)

`PythonCompilationService` compiles YAML DSL to standalone Python functions.

```java
PythonCompiledRule compiled = compilationService.compileRule(yamlDsl, "credit_scoring", true);
compiled.getPythonCode();        // Generated Python source
compiled.getInputVariables();    // Required inputs
compiled.getOutputVariables();   // Expected outputs

Map<String, PythonCompiledRule> batch = compilationService.compileRules(rulesMap, true);
compilationService.getCompilationStats();
```

---

## Configuration

```yaml
firefly:
  rules:
    cache:
      provider: CAFFEINE   # CAFFEINE (default) or REDIS
      caffeine:
        ast-cache:
          maximum-size: 1000
          expire-after-write: 2h
        constants-cache:
          maximum-size: 500
          expire-after-write: 15m
        rule-definitions-cache:
          maximum-size: 200
          expire-after-write: 10m
```

Maven dependencies -- core: `fireflyframework-rule-engine-core`, web: `fireflyframework-rule-engine-web`.

---

## Writing Tests

```java
class MyRuleTest {
    private ASTRulesEvaluationEngine evaluationEngine;

    @BeforeEach
    void setUp() {
        DSLParser dslParser = new DSLParser();
        ASTRulesDSLParser parser = new ASTRulesDSLParser(dslParser);
        ConstantService constantService = Mockito.mock(ConstantService.class);
        Mockito.when(constantService.getConstantsByCodes(Mockito.anyList()))
               .thenReturn(Flux.empty());
        evaluationEngine = new ASTRulesEvaluationEngine(parser, constantService, null, null);
    }

    @Test
    void testCreditApproval() {
        String yamlRule = """
            when:
              - creditScore at_least 650
              - annualIncome at_least 50000
            then:
              - set eligibility to "QUALIFIED"
              - calculate max_loan as annualIncome * 4
            else:
              - set eligibility to "NOT_QUALIFIED"
            """;
        Map<String, Object> input = Map.of("creditScore", 720, "annualIncome", new BigDecimal("75000"));
        ASTRulesEvaluationResult result = evaluationEngine.evaluateRules(yamlRule, input);
        assertTrue(result.isSuccess());
        assertEquals("QUALIFIED", result.getOutputData().get("eligibility"));
    }
}
```

To test with database constants, mock `ConstantService` to return `ConstantDTO`:

```java
ConstantDTO minScore = ConstantDTO.builder()
    .code("MIN_CREDIT_SCORE").valueType(ValueType.NUMBER)
    .currentValue(new BigDecimal("650")).build();
when(constantService.getConstantsByCodes(anyList())).thenReturn(Flux.just(minScore));
```

---

## Anti-Patterns

### Do NOT hardcode business values -- use constants
```yaml
# BAD                                    # GOOD
when:                                    constants:
  - creditScore at_least 650               - code: MIN_CREDIT_SCORE
                                             defaultValue: 650
                                         when:
                                           - creditScore at_least MIN_CREDIT_SCORE
```

### Do NOT violate naming conventions
- Inputs must be `camelCase` (not `CREDIT_SCORE` or `credit_score`)
- Computed variables must be `snake_case` (not `approvalStatus`)
- Constants must be `UPPER_CASE`

### Do NOT use synchronous parsing in reactive contexts
```java
// BAD: blocks the event loop
ASTRulesDSL rule = parser.parseRules(yamlString);
// GOOD: reactive
Mono<ASTRulesDSL> ruleMono = parser.parseRulesReactive(yamlString);
```

### Do NOT ignore circuit breaker results
```java
if (result.isSuccess()) {
    if (result.isCircuitBreakerTriggered()) {
        log.warn("Circuit breaker: {}", result.getCircuitBreakerMessage());
    }
    processOutput(result.getOutputData());
}
```

### Do NOT construct ASTRulesEvaluationEngine manually in production
Use `@Autowired` injection. Manual construction (two-arg constructor) is only for unit tests.

### Do NOT skip validation before storing rules
`createRuleDefinition` validates internally, but explicit validation via `validateRuleDefinition()` gives better user feedback.

---

## Key Source Locations

| Component | Package |
|-----------|---------|
| Parser | `org.fireflyframework.rules.core.dsl.parser.ASTRulesDSLParser` |
| AST model | `org.fireflyframework.rules.core.dsl.model.ASTRulesDSL` |
| Visitor interface | `org.fireflyframework.rules.core.dsl.ASTVisitor` |
| Evaluation engine | `org.fireflyframework.rules.core.dsl.evaluation.ASTRulesEvaluationEngine` |
| Evaluation result | `org.fireflyframework.rules.core.dsl.evaluation.ASTRulesEvaluationResult` |
| Evaluation context | `org.fireflyframework.rules.core.dsl.visitor.EvaluationContext` |
| Conditions | `org.fireflyframework.rules.core.dsl.condition.ComparisonCondition`, `LogicalCondition` |
| Actions | `org.fireflyframework.rules.core.dsl.action.SetAction`, `CalculateAction`, `ForEachAction` |
| Validation | `org.fireflyframework.rules.core.validation.YamlDslValidator` |
| Python compiler | `org.fireflyframework.rules.core.dsl.compiler.PythonCompilationService` |
| Services | `RulesEvaluationService`, `BatchRulesEvaluationService`, `RuleDefinitionService`, `AuditTrailService`, `ConstantService` |
| DTOs | `org.fireflyframework.rules.interfaces.dtos.evaluation.*`, `.crud.*`, `.audit.*` |
| Controllers | `RulesEvaluationController`, `BatchRulesEvaluationController`, `ValidationController` |
| Entities | `org.fireflyframework.rules.models.entities.RuleDefinition`, `AuditTrail`, `Constant` |
