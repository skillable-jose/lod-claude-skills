---
name: write-unit-tests
description: Generate or extend unit tests for a C# class (xUnit + Moq, strict/one-method-deep) or a TypeScript/React module or component (Vitest baseline, Jest where a repo's actual tooling says so, + Testing Library). Plans coverage per method/behavior before writing code. Use when asked to write, add, or improve unit tests for a class, function, or component.
version: 1.1.0
applies_to: [dotnet, node]
references: [ADR-IDP007]
authors: Jose Murillo Castro
last_updated: 2026-08-28
argument-hint: "[ClassName, function name, component, or file path] [--plan-only]"
---

# Write Unit Tests

## What this does

Generates unit tests under the target repo's **actual** testing conventions, not a generic default — verifying real configured tooling before writing a single test, since a repo's documented convention and its actually-running tooling can disagree (see Step 2). Plans coverage per method/behavior first and asks for confirmation before writing code.

## When to use this

Use when asked to write, add, or improve unit tests for an existing C# class or TypeScript/React module/component.

**Do NOT use this for:**
- Scaffolding a brand-new test project that doesn't exist yet — that's `bootstrap-clean-arch` (C#) or `bootstrap-node-from-template`/`extend-node-solution` (TS, if the `ai-enablement` skill set is installed in this repo).
- Reviewing existing tests for quality/anti-patterns without adding new ones — that's closer to `ai-enablement`'s `review-dotnet-best-practices`/`review-efcore` (C#) or `review-react-best-practices`/`review-typescript-best-practices` (TS), if installed.
- Writing integration/E2E tests as the primary goal — this skill's C# integration-test pattern (`IClassFixture<WebApplicationFactory<Program>>`) and TS Playwright mention are secondary; reach for a dedicated E2E skill/runner for that work.

## Inputs

| Input | Type | Notes |
|---|---|---|
| `$ARGUMENTS` (class/function/component name or file path) | string | If empty, ask which class, function, component, or file to test — never guess. |
| `--plan-only` | flag | Stop after Step 4's plan table; don't write any test code. |

## Outputs

- A per-method/behavior test-plan table, presented for confirmation before any code is written.
- New or extended test file(s), following the resolved language's conventions.
- A test-run result (`dotnet test` / the repo's `test` script) and, if configured, a before/after coverage delta.

## How it works

### Step 1 — Resolve the Target and Language

Parse `$ARGUMENTS` for a class/function/component name or a file path.

- If a file path is given, read it directly; its extension decides the language track below (`.cs` → C#, `.ts`/`.tsx` → TypeScript).
- If a bare name is given, locate its source file (search `src/`, or the whole repo if there is no `src/` layout). If both a `.cs` and a `.ts` file could match, ask which one.
- If `$ARGUMENTS` is empty, ask the user which class, function, component, or file to test.

Locate the paired test location:
- **C#**: the paired test project `<SourceProject>.Tests` (e.g., `Skillable.Labs.CoreService` -> `Skillable.Labs.CoreService.Tests`). If none exists, tell the user and stop — creating a whole test project is out of scope (use `/bootstrap-clean-arch` for that).
- **TypeScript**: the co-located test file next to the source (`foo.ts` -> `foo.test.ts`, or `__tests__/foo.test.ts` — check existing sibling files/conventions before assuming).

### Step 2 — Load Testing Conventions

Conventions are, in priority order, for **either** language:

1. The repo's own deployed/rule-file conventions — `<repo-root>/rules/csharp/testing.md` (+ `coding-style.md` virtual-method section) or `<repo-root>/rules/typescript/testing.md`, if `/install-clean-arch-rules` has been run. Read in full, defer to them over anything below.
2. A repo-root testing standard doc if one exists outside `rules/` (e.g., `unit-tests.md`, `docs/TESTING.md`, `TESTING.md`, or a `## Testing` section in the root `CLAUDE.md`/`README.md`) — read it and treat it as authoritative for this repo.
3. **The actual tooling already in the repo** — always cross-check 1 and 2 against reality before writing a single test:
   - C#: open a sibling `.Tests.csproj` and check whether it references `xunit`, `MSTest.TestFramework`, or `NUnit`, and whether it references `Moq` or `NSubstitute`.
   - TypeScript: open `package.json` — the `test`/`test:coverage` script's binary (`vitest` vs `jest`) is ground truth, even if the other package also sits in `devDependencies` unused (a stalled migration is common). Check `vitest.config.*` / `jest.config.*` for `coverageThreshold`/`coverage`, `testEnvironment`, and setup files.
   - If the doc from step 1/2 disagrees with what's actually configured and running, **the actual tooling wins** — note the discrepancy to the user once, then proceed with what's real.
4. The bundled standards below, only if nothing above resolves the question.

### Bundled Standard — C# (xUnit + Moq, Strict)

This is `lab-on-demand`'s actual, documented convention (its own `CLAUDE.md`/`unit-tests.md`, confirmed against real `.csproj` files) — **not** the same as `ai-enablement`'s generic C# default (`_shared/csharp-coding-standards.md`, which prescribes MSTest + FluentAssertions and explicitly says not to add `// Arrange`/`// Act`/`// Assert` comments). Both are legitimate; Step 2's priority order exists precisely to resolve which one applies to a given repo — this bundle is the fallback of last resort, used only when neither a deployed rule file nor a repo-root doc nor the repo's actual `.csproj` packages settle it.

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
- **Result<T> methods (per ADR-IDP001, if the SUT uses it)**: assert `result.IsSuccess`/`result.IsFailure` and the `ResultError.Kind`/message directly — never wrap a `Result`-returning call in `Assert.ThrowsAsync`, since a `Result` failure is a return value, not a thrown exception.
- **Time-dependent code**: inject and use `FakeTimeProvider`, never `DateTime.Now`/`UtcNow` directly in the test.
- **AAA**: every test has `// Arrange`, `// Act`, `// Assert` comments in that order (this is the `lab-on-demand`-specific convention — drop the comments if the repo's actual style, per Step 2, says otherwise).

### Bundled Standard — TypeScript (Vitest baseline, Jest where the repo actually runs it)

The org's recorded frontend-architecture decision names **Vitest + Testing Library (+ MSW for integration tests)** as the baseline test stack — treat Vitest as the default assumption for a *new* test file in a repo with no stronger signal. Jest is not a sanctioned baseline; it shows up only because some real, already-installed repos (pre-dating that decision) run it — Step 2.3's "actual tooling wins" check exists specifically to catch that and use Jest correctly when it's what's really configured, without treating it as equally endorsed.

- **Framework**: whichever Step 2.3 confirms is actually configured. Vitest's `describe`/`it`/`expect`/`vi` come from an explicit `import ... from 'vitest'`. Jest's globals are ambient — don't add a test-framework import if sibling Jest test files don't have one.
- **Test naming**: nested `describe`/`it`, not method-name strings. Outer `describe` = the class/module/component name, exactly matching the file's subject. Inner `describe` = the method name (for services/functions) or the user action/rendered state (for components). Every `it(...)` string starts with `'should'` and reads as a full sentence — never `it('works')`, `it('handles error')`.
- **Two module shapes, two mocking approaches**:
  - **Feature-folder `api/` functions** (the org's actual baseline shape — plain exported functions in `features/<name>/api/<feature>.api.ts`, calling the shared `shared/api/client.ts`, no DI class): there's no constructor to inject into, so `vi.mock('../modulePath')` / `jest.mock(...)` on the dependency module is the correct, expected tool. Mock `global.fetch` (or the shared client) directly when the function under test calls it itself.
  - **Class with constructor-injected dependencies** (only if this repo deliberately uses that pattern — it is not the org baseline): construct the mock dependency object explicitly and inject it — no module mocking.
- **Always clear mocks** in `beforeEach` (`vi.clearAllMocks()` / `jest.clearAllMocks()`) — never share mock state across tests.
- **Assert both** the return value AND the mock call (count + arguments) for non-component tests.
- **Components**: Testing Library, query by role/label/visible text — never `data-testid` unless there's truly no semantic alternative. Assert what the user sees/does, never component state or internal props/classes.
- **Provider wrapping**: use whatever the repo's actual render helper/provider setup is (look for an existing `renderHelper`/custom `render` wrapper before assuming any particular provider pattern exists).
- **AAA**: label `// Arrange`, `// Act`, `// Assert` (or `// Arrange + Act` when a component test's render *is* the act).
- **Exceptions/failures**: `await expect(fn()).rejects.toThrow(ErrorType)` for a thrown exception; for the org's `ApiResponse<T>` discriminated union, assert on `response.success === false` and `response.error.code`/`message` directly — never wrap it in `.rejects`, since a failure `ApiResponse` is a return value, not a throw.

### Step 3 — Analyze the Target

**C# class:**
1. List constructor dependencies (interfaces to mock).
2. List every public, internal, and protected method. For each, note whether it's `virtual` (flag if not — see the warning rule), what sibling methods on the same class it calls (need SUT-mock setups), what external dependency calls it makes, its branches/early-returns, and anything it can throw or return as a `Result<T>` failure.
3. Cross-reference the existing test file to plan only the coverage gaps.

**TypeScript module/component:**
1. Determine the shape: a feature `api/` function module, a hook, a component, or (rarely) a class with constructor-injected dependencies. This decides the mocking approach (see Bundled Standard above).
2. List every exported function/method/component prop-driven behavior. For each, note its dependencies (modules to mock, or props/context it reads), its branches/conditional rendering, user interactions it exposes (clicks, form submits), and anything it can throw or return as an `{ success: false }` `ApiResponse`.
3. Cross-reference the existing `*.test.ts(x)` file to plan only the coverage gaps.

### Step 4 — Build and Present the Test Plan

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
| getRecommendationSummary | vi.mock/jest.mock (feature api function) | should call the API with the correct URL and options; should return a failure ApiResponse on a network error |
| ImpactStats (component) | renderHelper, click handlers as props | renders the impact counts; calls handleClick with the right impact when a stat button is clicked |
```

For each C# method flagged non-virtual/static, add a line above the table calling it out and recommending the `virtual` fix. Do not apply the fix yourself unless the user asks — it's a source-code change outside this skill's scope, only note it.

If `--plan-only` was passed, stop here after presenting the table.

Otherwise, ask the user to confirm the plan (or a subset of it) before writing test code — same upsert-review pattern as the other skills in this collection. Do not default to "write everything" silently.

### Step 5 — Write the Tests

For each confirmed test, follow the Bundled Standard for the resolved language (or the repo-specific conventions from Step 2, which win on conflict):

**C#:**
1. If the test file doesn't exist, create `<ClassUnderTest>Test.cs` in the test project, mirroring the source file's folder path.
2. If it exists, add new test methods and any newly-needed mock fields; don't reorder or rewrite existing passing tests.
3. Class naming, field naming, constructor setup, Verifiable/VerifyAll, CallBase, AAA — all per Step 2.
4. Cover, per method: happy path, each distinct branch, reasonable edge cases, exception propagation, and `Result<T>` failure paths where applicable. Separate interaction tests (a call happened) from state/return-value tests when a method does both.

**TypeScript:**
1. If the test file doesn't exist, create it co-located with the source (matching the repo's existing `.test.ts(x)` convention).
2. If it exists, add new `it`/`describe` blocks; don't reorder or rewrite existing passing tests.
3. Nested `describe`/`it` naming, mocking approach (module mock for feature `api/` functions vs constructor injection for the rarer DI class), `beforeEach` mock clearing, AAA — all per Step 2.
4. Cover, per function/behavior: happy path, each distinct branch/conditional render, user interactions, and error/failure paths (thrown exceptions and `{ success: false }` `ApiResponse`s alike).

### Step 6 — Build and Run

**C#:**
1. `dotnet build` the test project. Fix compile errors in the test file (never in production code, unless the user approved a `virtual` fix from Step 4).
2. `dotnet test` scoped to the affected test project.
3. If a failure reveals an actual production bug, stop and report it — do not weaken the test to force a pass.
4. If `coverlet` is configured, report before/after coverage for the modified class.

**TypeScript:**
1. Run the repo's actual test command (`pnpm test <pattern>` / `npm test -- <pattern>` / `yarn test <pattern>`), scoped to the file just written, using whichever runner Step 2.3 confirmed.
2. If a failure reveals an actual production bug, stop and report it — do not weaken the test to force a pass.
3. If coverage thresholds are configured, run the coverage script and report before/after for the modified file.

### Step 7 — Report

Summarize: methods/behaviors now covered, number of tests added, any C# methods still flagged as non-virtual (and thus untestable in isolation), and the final test-run result.

## Anti-patterns

- ❌ Assuming a framework instead of checking the actual `.csproj`/`package.json` (Step 2.3) — a repo can have an unused testing library sitting in its dependency list from an abandoned migration.
- ❌ C#: `MockBehavior.Loose`, or letting a sibling SUT method call execute real logic instead of mocking it (breaks one-method-deep isolation).
- ❌ C#: silently skipping the non-virtual/static warning instead of surfacing it in the plan table.
- ❌ TypeScript: wrapping a `Result<T>`/`ApiResponse<T>` failure assertion in `.rejects.toThrow()` — it's a return value, not a throw.
- ❌ TypeScript: inventing a provider wrapper (e.g. a `ServicesProvider`) that doesn't actually exist in this codebase, instead of using its real render helper.
- ❌ Rewriting existing passing tests wholesale instead of extending — unless the user explicitly asked for a cleanup pass.
- ❌ Editing production code from this skill beyond what the user explicitly approved (e.g. a `virtual` fix).

## Validation

- [ ] Test framework/mocking library confirmed against actual `.csproj`/`package.json`, not assumed from a doc
- [ ] Test-plan table presented and confirmed before any test code was written (unless `--plan-only`)
- [ ] C#: every mock is `MockBehavior.Strict`; every `.Setup()` chains `.Verifiable`; every test calls `.VerifyAll()` on all mocks with setups
- [ ] C#: non-virtual/static SUT methods flagged, not silently tested around
- [ ] TypeScript: mocking approach matches the module's actual shape (feature `api/` function vs DI class)
- [ ] Existing passing tests untouched except for the newly added/extended ones
- [ ] `dotnet test` / the repo's test script passes for the affected file(s)

## Related skills

- **install-clean-arch-rules** — deploys the `/rules/csharp/testing.md` or `/rules/typescript/testing.md` this skill checks first, in Step 2.
- **bootstrap-clean-arch** — creates the test project this skill extends, if one doesn't exist yet.
- **`ai-enablement`'s `review-dotnet-best-practices` / `review-efcore` / `review-react-best-practices` / `review-typescript-best-practices`** (if installed) — review existing test quality; this skill adds new tests instead.

## Notes for skill authors

The C# and TypeScript "Bundled Standard" sections each explain *why* they diverge from the org's generic default (`csharp-coding-standards.md`'s MSTest+FluentAssertions; the frontend ADR's Vitest baseline) rather than silently picking one. Keep that framing whenever this file is edited — a silent divergence reads as an oversight to anyone comparing this skill against the org's shared standards later.

**Version history:**
- 1.1.0 — Adopted the `ai-enablement` `SKILL.md` house style. Reframed the TypeScript bundled standard around the org's actual Vitest/feature-folder/`ApiResponse<T>` baseline (previously defaulted Next.js to Jest and assumed a domain/repository/service DI layering that isn't the org's real architecture). Added `Result<T>`/`ApiResponse<T>` assertion guidance per ADR-IDP001.
- 1.0.0 — Initial release, C# xUnit/Moq only.
- 1.0.1 — Extended to TypeScript (Jest or Vitest + Testing Library).

**Maintainer:** Jose Murillo Castro
