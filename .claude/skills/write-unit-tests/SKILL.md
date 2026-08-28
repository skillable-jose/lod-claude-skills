---
name: write-unit-tests
description: Generate or extend unit tests for a C# class (xUnit + Moq, strict/one-method-deep) or a TypeScript/React/Next.js module or component (Jest or Vitest + Testing Library). Plans coverage per method/behavior before writing code. Use when asked to write, add, or improve unit tests for a class, function, or component.
argument-hint: "[ClassName, function name, component, or file path] [--plan-only]"
---

# Write Unit Tests

Generate unit tests under the target repo's actual testing conventions, not a generic default. Every test this skill writes must satisfy the checklist for its language (Step 5).

**Core behavior is upsert:** an existing test file is extended with missing tests, never rewritten wholesale. Existing passing tests are left untouched unless they conflict with the conventions below and the user asks for a cleanup pass.

## Step 1 — Resolve the Target and Language

Parse `$ARGUMENTS` for a class/function/component name or a file path.

- If a file path is given, read it directly; its extension decides the language track below (`.cs` → C#, `.ts`/`.tsx` → TypeScript).
- If a bare name is given, locate its source file (search `src/`, or the whole repo if there is no `src/` layout). If both a `.cs` and a `.ts` file could match, ask which one.
- If `$ARGUMENTS` is empty, ask the user which class, function, component, or file to test.

Locate the paired test location:
- **C#**: the paired test project `<SourceProject>.Tests` (e.g., `Skillable.Labs.CoreService` -> `Skillable.Labs.CoreService.Tests`). If none exists, tell the user and stop — creating a whole test project is out of scope (use `/bootstrap-clean-arch` for that).
- **TypeScript**: the co-located test file next to the source (`foo.ts` -> `foo.test.ts`, or `.spec.ts` if the repo's own convention uses that — check existing sibling files before assuming).

## Step 2 — Load Testing Conventions

Conventions are, in priority order, for **either** language:

1. The repo's own deployed/rule-file conventions — `<repo-root>/rules/csharp/testing.md` (+ `coding-style.md` virtual-method section) or `<repo-root>/rules/typescript/testing.md`, if `/install-clean-arch-rules` has been run. Read in full, defer to them over anything below.
2. A repo-root testing standard doc if one exists outside `rules/` (e.g., `unit-tests.md`, `docs/TESTING.md`, `TESTING.md`, or a `## Testing` section in the root `CLAUDE.md`/`README.md`) — read it and treat it as authoritative for this repo.
3. **The actual tooling already in the repo** — always cross-check 1 and 2 against reality before writing a single test:
   - C#: open a sibling `.Tests.csproj` and check whether it references `xunit`, `MSTest.TestFramework`, or `NUnit`, and whether it references `Moq` or `NSubstitute`.
   - TypeScript: open `package.json` — the `test`/`test:coverage` script's binary (`jest` vs `vitest`) is ground truth, even if the other package also sits in `devDependencies` unused (a stalled migration is common). Check `jest.config.*` / `vitest.config.*` for `coverageThreshold`, `testEnvironment`, and setup files.
   - If the doc from step 1/2 disagrees with what's actually configured and running, **the actual tooling wins** — note the discrepancy to the user once, then proceed with what's real.
4. The bundled standards below, only if nothing above resolves the question.

### Bundled Standard — C# (xUnit + Moq, Strict)

- **Framework**: xUnit (`[Fact]`, `[Theory]`/`[InlineData]`). Never MSTest or NUnit attributes.
- **Mocking**: Moq, always `MockBehavior.Strict`.
- **Test class name**: `<ClassUnderTest>Test` — singular, no trailing `s`.
- **Mock field naming**: underscore-prefixed camelCase, suffixed `Mock`, no abbreviations (e.g., `_labProfileRepositoryMock`).
- **Setup**: constructor-based (xUnit re-instantiates the class per test — no `[TestInitialize]`). Use `IDisposable.Dispose()` for teardown if needed.
- **Mock the SUT itself** via factory-constructor syntax (`new Mock<T>(() => new T(deps...), MockBehavior.Strict)`) so virtual sibling methods can be intercepted — tests must go exactly **one method deep**.
- **Test method naming**: `<MethodName>_Should<Result>_When<Condition>` or `_Given<Condition>`. Never vague (`_Works`, `_Success`).
- **Every `.Setup()` chains `.Verifiable(Times.X)`.** Every test ends by calling `.VerifyAll()` on the SUT mock and every dependency mock that has a setup.
- **Virtual method under test**: `.Setup(...).CallBase().Verifiable(...)`. **Virtual method called internally**: `.Setup(...).ReturnsAsync(...).Verifiable(...)` (no `CallBase`). **Non-virtual method under test**: no setup needed, real code runs directly.
- **Non-virtual/static methods on the SUT**: flag as a warning, recommend making them `virtual`, do not silently write a test that lets them execute real sibling logic.
- **Parameter matching**: prefer exact values or `It.Is<T>(...)` predicates over `It.IsAny<T>()`.
- **Assertions**: xUnit `Assert` (`Assert.Equal`, `Assert.NotNull`, `Assert.ThrowsAsync<T>`, ...). Never `[ExpectedException]`-style attributes.
- **Time-dependent code**: inject and use `FakeTimeProvider`, never `DateTime.Now`/`UtcNow` directly in the test.
- **AAA**: every test has `// Arrange`, `// Act`, `// Assert` comments in that order.

### Bundled Standard — TypeScript (Jest or Vitest + Testing Library)

- **Framework**: whichever the repo's `test` script actually runs (Step 2.3) — most Next.js apps run **Jest** via `next/jest`; a Vite SPA more often runs **Vitest**. Jest's `describe`/`it`/`expect`/`jest` globals are ambient — don't add a test-framework import if sibling test files don't have one. Vitest requires `import { describe, it, expect, vi } from 'vitest'`.
- **Test naming**: nested `describe`/`it`, not method-name strings. Outer `describe` = the class/module/component name, exactly matching the file's subject. Inner `describe` = the method name (for services/functions) or the user action/rendered state (for components). Every `it(...)` string starts with `'should'` and reads as a full sentence — never `it('works')`, `it('handles error')`.
- **Two module shapes, two mocking approaches**:
  - **Class with constructor-injected dependencies** (the `frontend-arch.md` service/repository pattern, if this repo uses it): construct the mock dependency object explicitly and inject it — no module mocking (`vi.mock`/`jest.mock`).
  - **Plain exported functions** (e.g. a `fetch`-calling API module, the common case in real Next.js codebases with no DI layer): there's no constructor to inject into, so `jest.mock('../modulePath')` / `vi.mock(...)` on the dependency module is the correct, expected tool. Mock `global.fetch` directly when the function under test calls `fetch` itself.
- **Always clear mocks** in `beforeEach` (`jest.clearAllMocks()` / `vi.clearAllMocks()`) — never share mock state across tests.
- **Assert both** the return value AND the mock call (count + arguments) for non-component tests.
- **Components**: React Testing Library, query by role/label/visible text — never `data-testid` unless there's truly no semantic alternative. Assert what the user sees/does, never component state or internal props/classes.
- **Provider wrapping**: use whatever the repo's actual render helper/provider setup is (look for an existing `renderHelper`/custom `render` wrapper before assuming `<ServicesProvider>` — that only exists in repos actually using the `frontend-arch.md` layered composition root).
- **AAA**: label `// Arrange`, `// Act`, `// Assert` (or `// Arrange + Act` when a component test's render *is* the act).
- **Exceptions**: `await expect(fn()).rejects.toThrow(ErrorType)`, not try/catch.

## Step 3 — Analyze the Target

**C# class:**
1. List constructor dependencies (interfaces to mock).
2. List every public, internal, and protected method. For each, note whether it's `virtual` (flag if not — see the warning rule), what sibling methods on the same class it calls (need SUT-mock setups), what external dependency calls it makes, its branches/early-returns, and anything it can throw.
3. Cross-reference the existing test file to plan only the coverage gaps.

**TypeScript module/component:**
1. Determine the shape: a class with constructor-injected dependencies, a set of plain exported functions, a hook, or a component. This decides the mocking approach (Step 2).
2. List every exported function/method/component prop-driven behavior. For each, note its dependencies (modules to mock, or props/context it reads), its branches/conditional rendering, user interactions it exposes (clicks, form submits), and anything it can throw or reject.
3. Cross-reference the existing `*.test.ts(x)` file to plan only the coverage gaps.

## Step 4 — Build and Present the Test Plan

Output a table before writing any code. C# example:

```
| Method | Virtual? | Calls (same class) | Planned tests |
|---|---|---|---|
| CreateLabProfileAsync | Yes | ValidateAsync (dep) | ShouldReturnLabProfile_WhenRequestIsValid; ShouldThrowValidationException_WhenNameIsEmpty |
| ProcessLabProfileAsync | No — ⚠ flagged | GetLabProfileAsync (virtual, same class) | ShouldUpdateStatus_GivenValidTransition; ShouldThrowException_GivenInvalidTransition |
```

TypeScript example:

```
| Method / behavior | Mocking approach | Planned tests |
|---|---|---|
| getRecommendationSummary | jest.mock (plain function, fetch-based) | should call the API with the correct URL and options; should wrap a network error in ApiError |
| ImpactStats (component) | renderHelper, click handlers as props | renders the impact counts; calls handleClick with the right impact when a stat button is clicked |
```

For each C# method flagged non-virtual/static, add a line above the table calling it out and recommending the `virtual` fix. Do not apply the fix yourself unless the user asks — it's a source-code change outside this skill's scope, only note it.

If `--plan-only` was passed, stop here after presenting the table.

Otherwise, ask the user to confirm the plan (or a subset of it) before writing test code — same upsert-review pattern as the other skills in this collection. Do not default to "write everything" silently.

## Step 5 — Write the Tests

For each confirmed test, follow the Bundled Standard for the resolved language (or the repo-specific conventions from Step 2, which win on conflict):

**C#:**
1. If the test file doesn't exist, create `<ClassUnderTest>Test.cs` in the test project, mirroring the source file's folder path.
2. If it exists, add new test methods and any newly-needed mock fields; don't reorder or rewrite existing passing tests.
3. Class naming, field naming, constructor setup, Verifiable/VerifyAll, CallBase, AAA — all per Step 2.
4. Cover, per method: happy path, each distinct branch, reasonable edge cases, exception propagation. Separate interaction tests (a call happened) from state/return-value tests when a method does both.

**TypeScript:**
1. If the test file doesn't exist, create it co-located with the source (matching the repo's existing `.test.ts(x)` vs `.spec.ts` convention).
2. If it exists, add new `it`/`describe` blocks; don't reorder or rewrite existing passing tests.
3. Nested `describe`/`it` naming, mocking approach (constructor injection vs module mock), `beforeEach` mock clearing, AAA — all per Step 2.
4. Cover, per function/behavior: happy path, each distinct branch/conditional render, user interactions, and error/rejection paths.

## Step 6 — Build and Run

**C#:**
1. `dotnet build` the test project. Fix compile errors in the test file (never in production code, unless the user approved a `virtual` fix from Step 4).
2. `dotnet test` scoped to the affected test project.
3. If a failure reveals an actual production bug, stop and report it — do not weaken the test to force a pass.
4. If `coverlet` is configured, report before/after coverage for the modified class.

**TypeScript:**
1. Run the repo's actual test command (`pnpm test <pattern>` / `npm test -- <pattern>` / `yarn test <pattern>`), scoped to the file just written, using whichever runner Step 2.3 confirmed.
2. If a failure reveals an actual production bug, stop and report it — do not weaken the test to force a pass.
3. If coverage thresholds are configured (`jest.config.*`'s `coverageThreshold`, or equivalent), run the coverage script and report before/after for the modified file.

## Step 7 — Report

Summarize: methods/behaviors now covered, number of tests added, any C# methods still flagged as non-virtual (and thus untestable in isolation), and the final test-run result.

## Constraints

- **Never assume a framework — verify it.** Check the actual test script/csproj packages (Step 2.3) before writing a single line; a repo can have an unused testing library sitting in its dependency list from an abandoned migration.
- **C# — strict mocks only, one method deep.** Never `MockBehavior.Loose`; sibling SUT method calls are always mocked (with `CallBase` only when that sibling *is* the method under test). Never silently skip the non-virtual warning.
- **TypeScript — module mocking is scoped, not default.** Use it for plain function/fetch modules; use constructor injection for DI'd service classes. Don't wrap tests in a provider (`ServicesProvider` or otherwise) that doesn't actually exist in this codebase.
- **Never rewrite existing tests wholesale** — extend, don't replace, unless the user explicitly asks for a cleanup.
- **Never edit production code from this skill** — only flag needed changes (e.g., adding `virtual`) and let the user decide.
- **Repo-specific conventions win** — deployed rules or a repo-root testing doc override the bundled standard; actual configured tooling overrides both if they disagree.
