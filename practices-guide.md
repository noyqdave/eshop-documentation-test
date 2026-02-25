# Practices Guide: User Stories, BDD/TDD, Version Control and Use Case Specifications

## Overview
This guide captures practices for expressing, constraining, and documenting system behavior using use case specifications, BDD scenarios, and tests. These practices were refined in a real project where AI-assisted development required executable intent and strong behavioral guardrails.

Without executable intent (BDD scenarios and tests), AI-generated code quickly drifts into implementation detail and unintended behavior.

### How These Practices Were Applied in This Project
In this project, the development workflow followed a specific sequence:
1. **User stories and BDD acceptance tests** drove incremental development.
2. **BDD scenarios constrained AI-generated code** during implementation.
3. **Use case specifications were reverse engineered** from existing code and observed behavior after features were implemented, to normalize flows, terminology, and exception handling.

The practices below reflect both forward-looking guidance (for BDD scenarios that drive development) and retrospective normalization (for use case specifications that document what was built). BDD scenarios were written before implementation; use cases were written after.

---

## BDD (Behavior-Driven Development)

**Note on Project Context**: The problem space approach and business language guidelines prevented AI from generating overly technical implementations.

### 1. Problem Space vs Solution Space

#### ✅ **Correct Approach (Problem Space)**
- Describe business intent and outcomes
- Use business language stakeholders understand
- Focus on what the business gets

#### ❌ **Anti-Pattern (Solution Space)**
```gherkin
Given the system is operational and the database is accessible
And the "create-transaction" feature flag is enabled
When I send a POST request to "/transactions" with the following data:
```json
{
  "amount": 100.00,
  "description": "Office supplies",
  "type": "EXPENSE"
}
```
Then the system should validate the request format
And the system should validate the feature flag
```

#### ✅ **Correct Pattern (Problem Space)**
```gherkin
Given I want to record a business expense
And the transaction amount is $100.00
And the transaction description is "Office supplies"
And the transaction type is "EXPENSE"
When I create the transaction
Then the transaction should be recorded in the ledger
And the transaction should show the correct amount of $100.00
```

### 2. Black Box vs White Box

#### ✅ **Correct Approach (Black Box)**
- Focus on outcomes and business value
- Describe what the system delivers
- Avoid internal implementation details

#### ❌ **Anti-Pattern (White Box)**
```gherkin
Then the system should validate the request format
And the system should track the "transactions.created" metric
And the system should create a domain object with:
- A generated UUID as identifier
- Current timestamp as creation date
And the system should persist the transaction to the database
And the system should return HTTP 200 with the created transaction object
```

#### ✅ **Correct Pattern (Black Box)**
```gherkin
Then the transaction should be recorded in the ledger
And the transaction should have a unique identifier
And the transaction should show the correct amount of $100.00
And the transaction should show the description "Office supplies"
And the transaction should be marked as an expense
And the transaction should have a timestamp showing when it was created
```

### 3. DRY (Don't Repeat Yourself)

#### ✅ **Correct Approach**
- Reference existing documentation
- Don't repeat business rules already defined
- Keep scenarios focused on behavior

#### ❌ **Anti-Pattern**
```markdown
## Business Rules
- All transactions must have a positive amount
- All transactions must have a description
- Each transaction gets a unique identifier
- Each transaction is timestamped when created
- Transactions are permanently recorded in the ledger
```

#### ✅ **Correct Pattern**
```markdown
## Notes
- These scenarios test the basic flow (happy path) of UC-001: Create Transaction
- Business rules and constraints are defined in the use case specifications
- Scenarios focus on different data variations to ensure system robustness
```

### 4. Proper Gherkin Structure

#### ✅ **Correct Pattern**
```gherkin
Feature: Create Transaction

Scenario: Successfully Create an Expense Transaction
Given I want to record a business expense
And the transaction amount is $100.00
And the transaction description is "Office supplies"
And the transaction type is "EXPENSE"
When I create the transaction
Then the transaction should be recorded in the ledger
And the transaction should have a unique identifier
And the transaction should show the correct amount of $100.00
And the transaction should show the description "Office supplies"
And the transaction should be marked as an expense
And the transaction should have a timestamp showing when it was created
```

### 5. Business Language Guidelines

**Note**: This section provides specific examples of business language. The core principles are covered in [Problem Space vs Solution Space](#1-problem-space-vs-solution-space) and [Black Box vs White Box](#2-black-box-vs-white-box).

#### ✅ **Use Business Terms**
- "record a business expense" (not "send POST request")
- "transaction should be recorded in the ledger" (not "persist to database")
- "unique identifier" (not "UUID")
- "timestamp showing when it was created" (not "current timestamp")

### 6. TODO Comments in Step Definitions

#### ✅ **Correct Approach**
- TODO comments are acceptable during initial implementation (Red phase of TDD)
- TODO comments must be removed once the method is successfully implemented
- Replace TODO comments with regular explanatory comments if the explanation adds value
- Keep code clean and maintainable by removing outdated TODOs

#### ❌ **Anti-Pattern**
```java
// WRONG: Leaving TODO comments after implementation is complete
@Given("I provide an idempotency key {string}")
public void i_provide_an_idempotency_key(String key) {
    // TODO: Store the idempotency key in shared test context
    // This key will be used by TransactionStepDefinitions when creating transactions
    testContext.setIdempotencyKey(key);  // Implementation is done, TODO should be removed
}
```

#### ✅ **Correct Pattern**
```java
// CORRECT: Remove TODO once implementation is complete
@Given("I provide an idempotency key {string}")
public void i_provide_an_idempotency_key(String key) {
    // Store the idempotency key in shared test context
    // This key will be used by TransactionStepDefinitions when creating transactions
    testContext.setIdempotencyKey(key);
}
```

#### ✅ **When to Remove TODOs**
- After the method implementation is complete and tested
- After the test passes (Green phase of TDD)
- During code review or refactoring phase
- Before committing code to version control

#### ✅ **When to Keep Comments**
- Keep explanatory comments that add value for future maintainers
- Keep comments that explain non-obvious business logic
- Keep comments that document complex test setup or data flow
- Remove only the "TODO:" prefix, keep useful explanations

---

## Test-Driven Development (TDD)

**Note on Project Context**: In this project, test-accompanied development was used rather than strict TDD. Tests and implementation were written together as a tight pair, with test intent defined before implementation was complete. This pattern works well with AI-assisted development, where AI can generate code that often satisfies the test intent on the first try.

### 1. Classic TDD: The Red-Green-Refactor Cycle

#### ✅ **Strict TDD Approach**
1. **Red**: Write a failing test first
   - Write the test
   - **Run the test to see it fail** (critical step - verify it fails for the right reason)
   - Test should fail for the right reason (missing functionality, not compilation error)
   - Test should be specific and focused on one behavior
2. **Green**: Write minimal code to make the test pass
   - Implement only what's needed to satisfy the test
   - Don't add extra features or optimizations yet
3. **Refactor**: Improve code quality while keeping tests green
   - Clean up code, remove duplication, improve naming
   - Ensure all tests still pass

### 2. Test-Accompanied Development (Used in This Project)

#### ✅ **Test-Accompanied Development Approach**
A workflow where tests and implementation are written in close pairing, with the intent and acceptance criteria established in the test before implementation, but without requiring a failed test run before writing production code.

**Core Rule (Non-Negotiable)**: The test must be written before the implementation that satisfies it, behavior must be verified by running the test, and test intent must be clearly expressed.

**Why This Works with AI-Assisted Development**:
- AI can quickly generate minimal code that satisfies a specified test intent
- The cost of "red first" is higher than its cognitive benefit in many modern environments
- You still define expected behavior before writing implementation code
- The test still constrains the implementation
- You still verify correctness with the test

**Risks and Mitigations**:
- **Risk**: You may not see false negatives early (typo in test may be hidden)
- **Mitigation**: Run tests frequently, especially after AI-generated code
- **Risk**: Testing feedback is delayed if you batch changes
- **Mitigation**: Run tests after each small batch of changes
- **Risk**: You might get false confidence if AI code looks right but doesn't satisfy intended behavior
- **Mitigation**: Always verify tests pass and check that they test the right behavior

#### ❌ **Anti-Pattern**
```java
// WRONG: Writing implementation before tests
public class MyService {
    public void doSomething() {
        // Full implementation written first
        // Then tests written to match
    }
}

// WRONG: Writing test after implementation is complete
// Test should define behavior before implementation
public String doSomething() {
    return "expected";  // Implementation written first
}
@Test
public void shouldDoSomething() {
    // Test written after - wrong order
    MyService service = new MyService();
    String result = service.doSomething();
    assertEquals("expected", result);
}
```

#### ✅ **Strict TDD Pattern**
Strict TDD differs only in timing:
- After writing the test, run it to verify it fails for the right reason.
- Then implement the minimal code and re-run to verify it passes.

#### ✅ **Test-Accompanied Development Pattern (Used in This Project)**
```java
// CORRECT: Test intent defined first, then implementation written to satisfy it
@Test
public void shouldDoSomething() {
    // Test written first - defines expected behavior clearly
    MyService service = new MyService();
    String result = service.doSomething();
    assertEquals("expected", result);
}
// Implementation written to satisfy the test intent:
public String doSomething() {
    return "expected";  // Minimal implementation
}
// Run test to verify it passes and behavior is correct
```

### 3. One Test at a Time

#### ✅ **Correct Approach**
- Write one test (or test + implementation pair)
- **For strict TDD**: Run the test to see it fail, then implement
- **For test-accompanied**: Write test and implementation together, then run to verify
- Make it pass (or verify it passes)
- Refactor if needed (see [When to Refactor](#6-tdd-workflow-practices))
- Move to the next test
- Don't write multiple tests before implementing

#### ❌ **Anti-Pattern**
```java
// WRONG: Writing all tests upfront
@Test public void test1() { /* ... */ }
@Test public void test2() { /* ... */ }
@Test public void test3() { /* ... */ }
// Then implementing all at once
```

#### ✅ **Correct Pattern**
```java
// CORRECT: One test at a time
@Test
public void shouldHandleFirstCase() {
    // Write test, define behavior, implement, verify it passes
}

// After first test passes:
@Test
public void shouldHandleSecondCase() {
    // Write next test, define behavior, implement, verify it passes
}
```

### 4. Communicating TDD Intent

#### ✅ **How to Request TDD Workflow**

When requesting features, explicitly state:
- "I want to follow TDD for this feature"
- "Write the first test for [specific behavior]. Define the expected behavior clearly."
- "For strict TDD: Run it to see it fail. Don't implement the code yet."
- "For test-accompanied: Write the test and implementation together, then run to verify."
- "One test at a time, please" (see [One Test at a Time](#3-one-test-at-a-time))

#### ✅ **Example Request Pattern (Strict TDD)**
```
"I want to add [feature] using strict TDD:

1. First, write a failing test for [specific behavior]
2. Run the test to see it fail (verify it fails for the right reason)
3. Then implement just enough code to make it pass
4. Run the test again to verify it passes
5. Then we'll move to the next test

Let's start with test #1: [describe the first test case]"
```

#### ✅ **Example Request Pattern (Test-Accompanied Development)**
```
"I want to add [feature] using test-accompanied development:

1. First, write a test for [specific behavior] that clearly defines expected behavior
2. Write the implementation to satisfy the test intent
3. Run the test to verify it passes and behavior is correct
4. Then we'll move to the next test

Let's start with test #1: [describe the first test case]"
```

### 5. Test Quality in TDD

#### ✅ **Good TDD Tests**
- **Specific**: Test one behavior at a time
- **Focused**: Clear what is being tested
- **Fast**: Run quickly (unit tests, not integration)
- **Independent**: Don't depend on other tests
- **Readable**: Test name clearly describes the behavior

#### ✅ **Test Naming Convention**
```java
// Pattern: should[ExpectedBehavior]When[Condition]
@Test
public void shouldReturnCachedResponseWhenSameIdempotencyKeyAndRequestHash() {
    // Test implementation
}

@Test
public void shouldReturnEmptyOptionalWhenIdempotencyKeyNotFound() {
    // Test implementation
}
```

### 6. TDD Workflow Practices

#### ✅ **Incremental Development**
- Start with the simplest test case
- Build complexity gradually
- Each test should add one new behavior
- Refactor when you see duplication or design issues

#### ✅ **When to Refactor**
- After a test passes (green phase)
- When you see code duplication
- When design becomes unclear
- When tests become hard to read

#### ❌ **When NOT to Refactor**
- While writing the first implementation (green phase)
- If tests are failing (red phase)
- If you're unsure about the design direction

### 7. TDD and Integration Tests

#### ✅ **Correct Approach**
- **BDD scenarios (Cucumber) are written first** - they drive development and will fail initially
- Use TDD for unit tests (fast feedback) - written after BDD scenarios define behavior
- Integration tests can be written after unit tests pass

#### ✅ **Layered Testing Strategy**
1. **BDD Scenarios**: Written first, test end-to-end behavior from user perspective, drive development
2. **Unit Tests (TDD)**: Fast, focused, test one class/component, written to support BDD scenarios
3. **Integration Tests**: Test component interactions, written after unit tests pass

### 8. Testing Framework: JUnit 4

#### ✅ **Correct Approach**
- **Use JUnit 4** for all tests in this project
- Cucumber requires JUnit 4 (via `cucumber-junit` dependency)
- Maven Surefire is configured for JUnit 4 (`surefire-junit4`)
- All test classes should use:
  - `import org.junit.Test;` (not `org.junit.jupiter.api.Test`)
  - `@RunWith(SpringRunner.class)` for Spring Boot tests
  - `public void` test methods (not package-private)

#### ❌ **Anti-Pattern**
```java
// WRONG: Using JUnit 5
import org.junit.jupiter.api.Test;

@SpringBootTest
class MyTest {
    @Test
    void shouldDoSomething() { // WRONG - JUnit 5 syntax
    }
}
```

#### ✅ **Correct Pattern**
```java
// CORRECT: Using JUnit 4
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.test.context.junit4.SpringRunner;

@RunWith(SpringRunner.class)
@SpringBootTest
public class MyTest {
    @Test
    public void shouldDoSomething() { // CORRECT - JUnit 4 syntax
    }
}
```

#### ✅ **Why JUnit 4?**
- **Cucumber Compatibility**: The project uses Cucumber for BDD testing, which requires JUnit 4
- **Maven Configuration**: Surefire plugin is configured with `surefire-junit4` provider
- **Consistency**: All existing tests use JUnit 4, maintaining consistency across the codebase
- **No JUnit 5**: Do not create JUnit 5 tests - they will not run with the current configuration

---

## Version Control Practices

### Reverting Failed Experiments

#### ✅ **Correct Approach**
When an experiment or feature attempt doesn't work out, use version control to revert to the last known good state instead of manually undoing changes.

```bash
# Find the last good commit
git log --oneline

# Reset to that commit (discards all changes)
git reset --hard <commit-hash>

# Clean up any untracked files
git clean -fd

# Verify tests still pass
mvn clean test
```

#### ❌ **Anti-Pattern**
```bash
# WRONG: Manually deleting files and reverting changes one by one
rm -rf src/test/java/.../PostgreSQLIntegrationTest.java
rm docs/postgresql-integration-testing.md
# Edit pom.xml to remove dependencies
# Edit application-test.yml to revert config
# ... many more manual steps
```

#### ✅ **Why This Matters**
- **Faster**: One command vs many manual steps
- **Safer**: Git knows exactly what changed, you might miss something
- **Complete**: Removes all related changes (config, docs, dependencies)
- **Verifiable**: Easy to confirm you're back to a known good state

#### ✅ **When to Use**
- Experimenting with new libraries or frameworks
- Trying architectural approaches that don't pan out
- Feature attempts that prove too complex or incompatible
- Any situation where you want to "start over" from a clean state

#### ✅ **Practice**
- Commit frequently at good checkpoints
- Use descriptive commit messages to identify good states
- Before major experiments, create a branch (can delete if it fails)
- After reset, verify tests pass before continuing

### Cleaning Up Debug Code

#### ✅ **Correct Approach**
- Remove all debug code before committing and pushing
- Debug code includes: `System.out.println()`, debug hooks, temporary logging, diagnostic classes
- Debug code is only needed during development/troubleshooting
- Once the problem is solved, debug code should be removed

#### ❌ **Anti-Pattern**
```java
// WRONG: Leaving debug code in the codebase
public class PropertySourceDebugHook {
    @Before(order = 0)
    public void dumpFeatureFlagResolution() {
        System.out.println("=== DEBUG feature flag resolution ===");
        System.out.println("Active profiles: " + String.join(",", env.getActiveProfiles()));
        // ... many more debug statements
    }
}
// This was useful for debugging, but should be removed once the issue is resolved
```

#### ✅ **Correct Pattern**
```java
// CORRECT: Debug code removed after issue is resolved
// No debug hooks or diagnostic classes remain in the codebase
// All tests still pass without debug code
```

#### ✅ **When to Remove Debug Code**
- After the problem is identified and fixed
- After tests pass and the solution is verified
- Before committing to version control
- During code review (reviewers should flag debug code)

#### ✅ **What Counts as Debug Code**
- Temporary `System.out.println()` statements
- Debug hooks (like `PropertySourceDebugHook`)
- Diagnostic classes created only for troubleshooting
- Temporary logging added to understand behavior
- Test code that prints internal state for debugging

#### ✅ **What to Keep**
- Production logging (using proper logging framework like SLF4J)
- Test assertions and validations
- Error handling code
- Comments that explain non-obvious logic

#### ✅ **Practice**
- Use proper logging frameworks for production code (SLF4J, Log4j, etc.)
- Use debuggers or IDE breakpoints during development instead of print statements
- If you must add debug code, add a TODO comment to remind yourself to remove it
- Run all tests after removing debug code to ensure nothing breaks
- Commit debug code removal in a separate commit with a clear message

---

## Use Case Specifications

**Note on Project Context**: The practices below reflect the discipline required to derive clear, accurate documentation from working code.

### 1. Preconditions vs Runtime Validation

#### ✅ **Correct Approach**
- **Preconditions**: Only system operational requirements
  - "System is operational and database is accessible"
  - "Application server is running"
- **Runtime Validation**: Feature flags, input validation, business rules
  - Checked in basic flow steps
  - Trigger alternative flows when conditions fail

#### ❌ **Anti-Pattern**
```markdown
- **Preconditions**: 
  - Feature flag "create-transaction" is enabled  # WRONG - this is runtime validation
  - System is operational
```

#### ✅ **Correct Pattern**
```markdown
- **Preconditions**: 
  - System is operational and database is accessible  # CORRECT - system requirement
```

### 2. Clean Basic Flows

#### ✅ **Correct Approach**
- Describe only the happy path
- Linear progression without conditional logic
- No branching statements

#### ❌ **Anti-Pattern**
```markdown
2. **System validates feature flag**: The system checks if the "create-transaction" feature flag is enabled. If enabled, processing continues to step 3. If disabled, the system follows alternative flow A1.
```

#### ✅ **Correct Pattern**
```markdown
2. **System validates feature flag**: The system checks if the "create-transaction" feature flag is enabled.
```

### 3. Alternative Flow Triggers

#### ✅ **Correct Approach**
- Alternative flows contain their own trigger descriptions
- Specific step references
- Clear condition descriptions

#### ✅ **Correct Pattern**
```markdown
#### A1: Feature Flag Disabled
- **Trigger**: In step 3, the "create-transaction" feature flag is disabled
- **Steps**:
  1. System checks feature flag and determines it is disabled
  2. System throws FeatureFlagDisabledException
  3. Global exception handler catches the exception
  4. System returns HTTP 403 Forbidden with error message
```

### 4. Exception Flows vs Alternative Flows

#### ✅ **Correct Approach**
- **Alternative Flows**: Specific validation failures or business logic exceptions during execution
- **Exception Flows**: Only actual exceptions that occur during execution
- **Precondition Failures**: Not included in use case (system unavailable, etc.)

#### ❌ **Anti-Pattern**
```markdown
### Exception Flows
- **System Unavailable**: If the application server is down, the request will fail with connection error
```

#### ✅ **Correct Pattern**
```markdown
### Exception Flows
- **Database Connection Lost**: If database connection fails during transaction persistence
```

### 5. Logical Separation of Concerns

#### ✅ **Correct Structure**
- **Preconditions**: What must be true before use case starts
- **Basic Flow**: Clean, linear happy path
- **Alternative Flows**: Specific failure scenarios with clear triggers
- **Exception Flows**: Only execution exceptions

---

## Common Anti-Patterns to Avoid

### Use Case Specifications
1. **Feature flags in preconditions** → Move to runtime validation
2. **Conditional logic in basic flows** → Keep basic flows clean
3. **System unavailable in exception flows** → This is a precondition failure
4. **Missing step references in alternative flows** → Always specify which step triggers the alternative

### BDD Scenarios
1. **Technical implementation details** → Focus on business intent
2. **JSON/HTTP details** → Use business language
3. **Internal system behavior** → Describe outcomes
4. **Repeating existing documentation** → Reference instead of repeat
5. **White box testing language** → Use black box approach

### TDD
1. **Writing implementation before tests** → Always write test before implementation
2. **Writing test after implementation is complete** → Test should define behavior before implementation
3. **Writing multiple tests before implementing** → One test at a time
4. **Skipping the refactor phase** → Clean up code after tests pass
5. **Not running tests to verify behavior** → Always run tests to verify they pass and test the right behavior
6. **Implementing more than needed** → Write minimal code to pass the test
7. **Using JUnit 5** → Use JUnit 4 (required for Cucumber compatibility)
8. **Missing @RunWith annotation** → Always use `@RunWith(SpringRunner.class)` for Spring Boot tests

---

## Quality Checklist

### Use Case Specifications
- [ ] Preconditions contain only system operational requirements (see [Preconditions vs Runtime Validation](#1-preconditions-vs-runtime-validation))
- [ ] Basic flow is clean and linear without conditional logic (see [Clean Basic Flows](#2-clean-basic-flows))
- [ ] Alternative flows have clear triggers with step references (see [Alternative Flow Triggers](#3-alternative-flow-triggers))
- [ ] Exception flows contain only execution exceptions (see [Exception Flows vs Alternative Flows](#4-exception-flows-vs-alternative-flows))
- [ ] No redundant information with other documentation (see [DRY](#3-dry-dont-repeat-yourself))

### BDD Scenarios
- [ ] Written in problem space (see [Problem Space vs Solution Space](#1-problem-space-vs-solution-space))
- [ ] Use black box language (see [Black Box vs White Box](#2-black-box-vs-white-box))
- [ ] Follow DRY principle (see [DRY](#3-dry-dont-repeat-yourself))
- [ ] Use business language (see [Business Language Guidelines](#5-business-language-guidelines))
- [ ] TODO comments removed once step definition methods are implemented (see [TODO Comments in Step Definitions](#6-todo-comments-in-step-definitions))

### TDD
- [ ] Test intent defined before implementation (core principle - see [Test-Accompanied Development](#2-test-accompanied-development-used-in-this-project))
- [ ] Test clearly expresses expected behavior
- [ ] Implementation satisfies the test intent
- [ ] Test is run to verify it passes and behavior is correct
- [ ] For strict TDD: Run test to see it fail before implementing
- [ ] For test-accompanied: Test and implementation written together, then verified
- [ ] Refactor after test passes (see [When to Refactor](#6-tdd-workflow-practices))
- [ ] One test at a time workflow (see [One Test at a Time](#3-one-test-at-a-time))
- [ ] Tests meet quality criteria (see [Test Quality in TDD](#5-test-quality-in-tdd))
- [ ] Use JUnit 4 (not JUnit 5) - see [Testing Framework: JUnit 4](#8-testing-framework-junit-4)

---

## References
- Use case specifications: [Use Case Specifications](use-case-specifications.md)
- BDD scenarios: [BDD Scenarios](bdd-scenarios.md)
- Architecture documentation: [Architecture Overview](../ARCHITECTURE.md)
