# Testing Requirements

## Minimum Test Coverage: 80%

Test Types (ALL required):
1. **Unit Tests** - Individual functions, utilities, components
2. **Integration Tests** - API endpoints, database operations
3. **E2E Tests** - Critical user flows (framework chosen per language)

## Test-Driven Development

MANDATORY workflow:
1. Write test first (RED)
2. Run test - it should FAIL
3. Write minimal implementation (GREEN)
4. Run test - it should PASS
5. Refactor (IMPROVE)
6. Verify coverage (80%+)

## Test Class Naming

Test class names MUST follow the pattern: `<ClassUnderTest>Tests`

Examples:
- `UserService` -> `UserServiceTests`
- `OrderValidator` -> `OrderValidatorTests`
- `ClaimDataMapper` -> `ClaimDataMapperTests`

## Test Naming — Intent Rule (Language-Agnostic)

Every test name MUST communicate three things, regardless of syntax:

1. **What is under test** — the unit, method, or component being exercised
2. **What outcome is expected** — the result, return value, side-effect, or error
3. **Under what condition** — the input state or scenario that triggers the outcome

The syntax used to express this depends on the language and framework:
- **C#** uses underscore-separated method names — see [csharp/testing.md](../csharp/testing.md)
- **TypeScript/JavaScript** uses nested `describe`/`it` string blocks — see [typescript/testing.md](../typescript/testing.md)

A test name that omits any of the three elements is incomplete. `GetUser_ShouldWork` and `it('works')` are both forbidden.

## AAA Pattern (Arrange-Act-Assert)

EVERY test MUST follow the Arrange/Act/Assert pattern with clearly marked comment sections:

```
// Arrange
<setup test data, configure mocks>

// Act
<invoke method under test>

// Assert
<verify results and mock interactions>
```

## Unit Test Philosophy: One Method Deep

Unit tests exercise **exactly one method** of the system under test (SUT). The test must not execute logic beyond the single method being tested:

- **Interface dependencies** (injected via constructor): Always mock them. The test controls their return values and verifies their invocations.
- **Internal method calls** (other methods on the same SUT class): Make them `virtual` and mock them on the `Mock<SUT>`. This isolates the method under test from sibling method logic.
- **The method under test itself**: Calls the real implementation through the mock (using `CallBase` for virtual methods, or directly for non-virtual methods).

This means every unit test creates a `Mock<SUT>` — not an instance of the real class. The mock wraps the real class, allowing selective interception of internal method calls while executing the real logic of the method under test.

### Why Mock the SUT?

Mocking the SUT is **not** testing the mock — it is testing the real method logic while controlling the environment around it:

1. Constructor dependencies are mocked as usual (interface mocks injected)
2. The method under test runs its **real code** (via `CallBase` or because it is non-virtual)
3. Any other methods the SUT calls on itself are **intercepted by the mock**, returning controlled values

This guarantees the test goes exactly one method deep and no further.

### All Methods Must Be Virtual — No Static Methods

**CRITICAL**: All public and internal methods on service classes MUST be `virtual`. This is a **class design rule** — see language-specific coding style rules (e.g., [csharp/coding-style.md](../csharp/coding-style.md#virtual-methods-on-service-classes--critical)) for the full rule, rationale, and code examples.

The testing implication: non-virtual methods cannot be intercepted by mock frameworks, so internal calls execute real logic and violate the one-method-deep rule. When you encounter a non-virtual or static method on a service class, **flag it as a warning** and recommend changing it to a virtual instance method before writing tests.

## Mock Standards

### Strict Mocks - MANDATORY

All mocks MUST use strict behavior to ensure:
- All interactions are explicitly defined
- No unexpected calls go unnoticed
- Tests fail if setup is incomplete

### Setup Verification - MANDATORY

Every mock setup call MUST be chained with verification for the expected number of invocations. This replaces ad-hoc `Verify()` calls scattered through the Assert section.

### VerifyAll() - MANDATORY

Every test MUST call `VerifyAll()` on:
1. The service/class mock itself (the `Mock<SUT>`)
2. ALL dependency mocks that have setup calls

This ensures every setup was actually invoked the expected number of times.

### Parameter Matching - Maximize Specificity

Avoid wildcard/any-match parameters when specific values can be checked:

- **Avoid**: Matching any argument when the exact value is known
- **Prefer**: Passing the exact expected value
- **Best**: Use predicate matching for complex objects

## Exception Testing

Use assertion methods to capture and verify exceptions. Do NOT use declarative exception attributes (e.g., `[ExpectedException]`) because they:
- Cannot verify the exception message
- Cannot verify exception properties
- Cannot assert state after the exception

Instead, use the framework's `ThrowsException` / `ThrowsExceptionAsync` assertion methods to capture the exception, then verify its message and properties.

## Virtual Method Testing

When the class under test has virtual methods:

1. **Testing the virtual method itself**: Configure the mock to call the real implementation (call-base) and verify invocation
2. **Testing a method that calls a virtual method**: Mock the virtual method's return value to isolate the method under test
3. **Testing a non-virtual method**: No special setup needed for the method under test

## Time-Dependent Testing

When testing code that depends on current time:
- Use a fake/testable time provider instead of the system clock
- Set a fixed time in test setup to ensure deterministic results
- Inject the time provider as a dependency

## Troubleshooting Test Failures

1. Use **tdd-guide** agent
2. Check test isolation
3. Verify mocks are correct
4. Fix implementation, not tests (unless tests are wrong)

## Agent Support

- **tdd-guide** - Use PROACTIVELY for new features, enforces write-tests-first
