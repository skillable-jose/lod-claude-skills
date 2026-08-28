# Logging Standards

## PRIORITIZE GLOBAL TELEMETRY OVER LOCAL LOGGING

**CRITICAL**: Rely on cross-cutting telemetry infrastructure (Application Insights, OpenTelemetry, etc.) to capture most observability data automatically. Do NOT manually log what telemetry already captures.

**Telemetry automatically captures**:
- HTTP requests/responses (status codes, duration, endpoints)
- Exceptions and stack traces
- Performance metrics
- Correlation IDs and distributed tracing
- Database queries and external service calls

**DO NOT log manually**:
- ❌ Request entry/exit ("Processing request", "Request completed")
- ❌ Success paths ("User created successfully", "Order processed")
- ❌ Method entry/exit ("Entering method X", "Exiting method Y")
- ❌ Obvious state changes already captured by telemetry
- ❌ Information already in exception stack traces

**Only log when**:
- ✅ Critical business decision point not visible in telemetry
- ✅ Contextual information needed for debugging that telemetry can't infer
- ✅ Non-exception failures that need investigation

## No Try/Catch Log Pattern

**DO NOT wrap every method in try/catch just to log exceptions**. Let exceptions bubble up to global exception handlers.

**WRONG**:
```csharp
public async Task<User> GetUserAsync(int id)
{
    try
    {
        return await _repository.GetByIdAsync(id);
    }
    catch (Exception ex)
    {
        _logger.LogError(ex, "Error getting user");  // ❌ Redundant - global handler captures this
        throw;
    }
}
```

**CORRECT**:
```csharp
public async Task<User> GetUserAsync(int id)
{
    // No try/catch - let global handler capture exceptions
    return await _repository.GetByIdAsync(id);
}
```

**Exception**: Only catch locally when transforming exceptions or performing mandatory cleanup:
```csharp
public async Task<Result<User>> GetUserAsync(int id)
{
    try
    {
        return Result.Success(await _repository.GetByIdAsync(id));
    }
    catch (NotFoundException ex)
    {
        // Transform to domain-specific result - logging handled globally
        return Result.Failure<User>("User not found");
    }
}
```

## Structured Logging (When You Must Log)

When logging is necessary, use structured logging with key-value pairs:

**Wrong**: `log("User 123 logged in")`
**Correct**: `log("User logged in", { userId: 123 })`

Benefits:
- Searchable and queryable in telemetry tools
- Enables filtering and aggregation
- Machine-readable for analytics

### Message Template Design

Design templates so the **action or event is immediately clear** without reading interpolated values. Push contextual/correlation values toward the end of the message. End log messages with a period -- they are finished sentences.

**Why**: Rendered log text should be scannable and greppable. A stable prefix lets you search for patterns (`grep "User not found"`) without variable data breaking the match.

**WRONG** -- variable data breaks the scannable pattern:
```csharp
_logger.LogWarning("User {UserId} was not found in tenant {TenantId}", userId, tenantId);
```
```typescript
logger.warn(`User ${userId} was not found in tenant ${tenantId}`);
```

**CORRECT** -- action first, context appended, finished sentence:
```csharp
_logger.LogWarning("User not found (UserId: {UserId}, TenantId: {TenantId}).", userId, tenantId);
```
```typescript
logger.warn({ userId, tenantId }, "User not found.");
```

**ALSO CORRECT** -- single identifying value that preserves a greppable prefix:
```csharp
_logger.LogWarning("Feature {FeatureName} unavailable, using fallback.", featureName);
```

**Guidance, not dogma**: If moving a value to the end creates awkward phrasing, prioritize readability. The goal is greppable, scannable messages -- not rigid syntax.

## Correlation IDs (Handled by Telemetry)

Modern telemetry systems (Application Insights, OpenTelemetry) automatically inject and track correlation IDs across distributed systems. **DO NOT manually add correlation IDs to log messages** - they're already part of the telemetry context.

## No Sensitive Data

NEVER log:
- Passwords
- API keys
- Security tokens
- Credit card numbers
- Social Security Numbers
- Personal identifying information (depending on policy)

**Wrong**: `log("Login attempt", { email: "user@example.com", password: "secret123" })`
**Correct**: Let telemetry capture auth attempts; only log if business context needed beyond standard telemetry

## Log Levels (Use Sparingly)

### ERROR
**Let global exception handlers log errors automatically.** Only log manually if:
- Business-critical failure not represented by exception
- Context needed beyond what's in exception details

**Example** (rare): `LogError("Payment gateway returned success but order was not created.", { orderId, gatewayResponse })`

### WARN
Recoverable issues where user action might be needed
- Feature degradation (fallback to default behavior)
- Configuration issues detected at runtime
- Resource limits approaching

**Example**: `LogWarning("Feature flag service unavailable, using cached flags.", { lastUpdate })`

### INFO
**Use extremely sparingly.** Only for business-critical events not captured by telemetry:
- Manual intervention required
- Compliance/audit events (financial transactions, data access)
- Critical state transitions not visible in telemetry

**Example**: `LogInformation("Refund approval required.", { orderId, amount, reason })`

### DEBUG
Development/troubleshooting only. Disabled in production.

**Example**: `LogDebug("Price calculation breakdown.", { basePrice, tax, discount, final })`

### TRACE
**Do not use.** Telemetry tracing handles this better.

## Language-Specific Implementations

### C#
Use `ILogger<T>` only when necessary (see guidelines above):
```csharp
_logger.LogWarning("Feature unavailable, using fallback.", new { feature, fallback });
```

**DO NOT**:
```csharp
_logger.LogInformation("User {UserId} logged in", userId);  // ❌ Telemetry captures this
_logger.LogInformation("Processing order {OrderId}", orderId);  // ❌ Redundant
```

### TypeScript
Use structured logger (Winston, Pino) sparingly:
```typescript
logger.warn({ feature, fallback }, "Feature unavailable, using fallback.");
```

## Logging Best Practices

- [ ] Rely on global telemetry for 95% of observability needs
- [ ] NO try/catch blocks just for logging - let exceptions bubble
- [ ] Only log when telemetry doesn't capture the information
- [ ] Use structured logging with key-value pairs when you do log
- [ ] Never log sensitive data (passwords, tokens, PII)
- [ ] Never log success paths or routine operations
- [ ] Configure telemetry tools (Application Insights, ELK, Splunk, OpenTelemetry) for central observability
- [ ] Use correlation IDs from telemetry context (automatic in modern frameworks)

## What Telemetry Already Captures (Don't Log These)

Modern telemetry platforms automatically capture:
- Request/response cycles with status codes, duration, paths
- Unhandled exceptions with full stack traces
- Database query performance and connection issues
- External API calls with latency and status codes
- User authentication events
- Resource usage (CPU, memory, I/O)
- Custom metrics and performance counters

**If telemetry already has it, don't log it manually.**

