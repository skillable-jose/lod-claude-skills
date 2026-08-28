# Coding Style

## Immutability (CRITICAL)

ALWAYS create new objects, NEVER mutate existing ones:

```csharp
// Example
WRONG:   modify(original, field, value) → changes original in-place
CORRECT: update(original, field, value) → returns new copy with change
```

Rationale: Immutable data prevents hidden side effects, makes debugging easier, and enables safe concurrency. Use record types and with expressions where possible.

## DRY, KISS, YAGNI (MANDATORY)

### DRY (Don't Repeat Yourself)
**BEFORE creating any new utility or helper**: ALWAYS search the codebase first.

Steps:
1. Search for similar functionality in existing code
2. If found, reuse or refactor into shared utility
3. If NOT found, create new utility in appropriate location
4. Document where utilities are located

**Anti-pattern**: Duplicating logic across files
**Correct**: Single source of truth for each piece of logic

### KISS (Keep It Simple, Stupid)
Prefer simple, readable solutions over clever, complex ones.

**Questions to ask**:
- Can this be understood by a junior developer?
- Is there a simpler way to achieve the same result?
- Am I over-engineering this?

### YAGNI (You Aren't Gonna Need It)
Don't build features or abstractions until they're actually needed.

**Anti-pattern**: Building extensibility points "just in case"
**Correct**: Add abstractions when you have concrete need for multiple implementations

## File Organization

MANY SMALL FILES > FEW LARGE FILES:

- **One class per file:** Each class, interface, enum, or record MUST be implemented in its own dedicated file. Nested private classes are the only exception.
- **Cohesion:** High cohesion, low coupling.
- **Size:** 200-400 lines typical, 800 lines max.
- **Extraction:** Extract utilities from large modules into focused, internal classes.
- **Structure:** Organize by feature/domain, not by technical type (e.g., Vertical Slices).

## Error Handling & Observability

### Fail Fast
Validate inputs and fail immediately at system boundaries. Don't let invalid data propagate through the system.

PRIORITIZE GLOBAL TELEMETRY OVER LOCAL CATCH BLOCKS:

- **Bubble Up:** Do not catch exceptions unless you are specifically transforming them into domain-specific errors or performing mandatory cleanup.
- **No Defensive Rethrowing:** Avoid `catch(ex) { throw ex; }`. It destroys the stack trace. Let the exception bubble to the global handler.
- **No Silent Failures:** NEVER silently swallow errors. Handle at every level or propagate explicitly.
- **Cross-Cutting Logging:** Rely on global middleware and telemetry (e.g., Azure Application Insights, OpenTelemetry) to capture request metadata and unhandled exceptions automatically.
- **No "Log-Vomit":** Do not log success paths, entering/exiting methods, or obvious state changes. Log only failures or critical state transitions at the system boundary.
- **Structured Logging:** Use message templates for searchable telemetry (e.g., `LogInformation("Order failed (OrderId: {OrderId}).", id)`).

### User-Friendly Error Messages
**UI layer**: Show friendly, actionable messages
**Server logs**: Include detailed context, stack traces, correlation IDs

**Example**:
- User sees: "Unable to process payment. Please try again."
- Log contains: "Payment gateway timeout (OrderId: 12345, GatewayResponse: 504, CorrelationId: abc-123)."

## Input Validation

ALWAYS validate at system boundaries:

- **Fail Fast:** Validate all user input before processing using schema-based validation (e.g., FluentValidation).
- **User Feedback:** Provide user-friendly messages in UI-facing code; log detailed context only on the server side via telemetry.
- **Zero Trust:** Never trust external data (API responses, user input, file content).

## Cyclomatic Complexity (CRITICAL)

**Maximum cyclomatic complexity per function: 6.** Functions exceeding 6 MUST be split.

Each decision point adds 1 (starting from 1): `if`, `else if`, `case`/pattern arm, `&&`, `||`, `catch`, `?.`, `??`, loops, ternary `? :`.

**Reduction techniques**:
- Guard clauses (early returns)
- Extract method (move conditional blocks into named helpers)
- Lookup tables / dictionaries (replace switch/if chains)
- Strategy pattern (replace conditional behavior with polymorphism)

## Code Quality Checklist

Before marking work complete:

- [ ] **Complexity:** Every function has cyclomatic complexity ≤ 6.
- [ ] **Readability:** Code is readable and well-named.
- [ ] **Sizing:** Functions are small (<50 lines) and files are focused (<800 lines).
- [ ] **Nesting:** No deep nesting (>4 levels).
- [ ] **Resilience:** Logic relies on global exception handling; minimal/no manual try/catch.
- [ ] **Observability:** Telemetry handles logging; no redundant manual log statements.
- [ ] **Cleanliness:** No hardcoded values (use constants or config).
- [ ] **Immutability:** No mutation (immutable patterns used).
