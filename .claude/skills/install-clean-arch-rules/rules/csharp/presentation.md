# C# Presentation Layer Rules

> Rules for Web.Core assemblies: controllers, minimal APIs, middleware, API versioning, response shape, and Swagger.
>
> The API-shape rules in this file are normative per a chain of ADRs, not locally invented: **ADR-API0001** (RESTful API Design Standards) is the base standard; **ADR-API0002** supersedes its Standard 4 (versioning); **ADR-API0003** supersedes its Standards 5 & 5a (response envelope + pagination); **ADR-API0004** supersedes its Standard 5b (`include` parameter); **ADR-API0005** supersedes part of its Standard 8 (403-vs-404 authorization disclosure). API0001's other standards (HTTP verb semantics, resource naming, URL structure, date format, sub-resources, HATEOAS) are unaffected by any of the four supersessions.

---
paths: ["**/*.cs", "**/*.csx"]
---

## Resource Naming & URL Structure (ADR-API0001, Standards 1–3, 5c)

- **Nouns, not verbs. Always plural.** `GET /v{n}/labprofiles`, not `GET /v{n}/GetLabProfiles`. The HTTP method conveys the action.
- **Action endpoints are the sanctioned exception** for operations that don't map to CRUD: `POST /v{n}/labprofiles/{id}/launch`.
- **Identifiers immediately follow the resource noun they identify**: `GET /v{n}/labprofiles/{id}/instructionsets`.
- **Sub-resource endpoints** return only the requested related data, not the parent payload, and inherit the parent's authorization rules: `GET /v{n}/labprofiles/{id}/skills`.
- **Dates**: ISO 8601, UTC, `YYYY-MM-DDTHH:mm:ssZ`.

| Method | Usage | Success status |
|---|---|---|
| `GET` | Read a resource or list a collection | `200 OK` |
| `POST` | Create a resource (ID assigned by service) | `201 Created` + `Location` header |
| `POST` | Perform a non-CRUD action | `200 OK` |
| `PUT` | Create or fully replace a resource | `200 OK` or `201 Created` |
| `PATCH` | Partially modify (JSON Merge Patch) | `200 OK` or `201 Created` |
| `DELETE` | Remove a resource | `204 No Content` (avoid `404`) |

## API Versioning (ADR-API0002 — supersedes API0001 Standard 4)

```
https://api.skillable.com/{domain}/v{MAJOR}/{resource}[/{id}…]?api-version=YYYY-MM-DD
```

- **Path major — per-domain, required, structural-only.** Each business domain owns its own `v{MAJOR}` segment and bumps it independently, only for a structural break. `/v1` is expected to be near-permanent. A request missing `/v{MAJOR}` is rejected with `400`.
- **Dated contract — query string, optional, the rolling boundary.** `?api-version=YYYY-MM-DD` selects a dated release *within* the current major; this is where ordinary breaking changes land. When omitted, it defaults to the dated release closest to when the calling account's credentials were minted, and is overridable per request.
- **Additive changes** (new optional fields, new endpoints) ship with **no** version change at all.
- The ADR's own implementation notes describe the APIM/ASP.NET wiring as **deliberately deferred** — the shape and the required-major/optional-date rule are the ratified decision; the exact `Asp.Versioning` configuration below is illustrative, not itself ratified:

```csharp
// Illustrative — confirm against the service's actual APIM/versioning wiring before copying verbatim
builder.Services.AddApiVersioning(options =>
{
    options.ApiVersionReader = new UrlSegmentApiVersionReader(); // path major
    options.AssumeDefaultVersionWhenUnspecified = false;         // major is REQUIRED — no silent default
    options.ReportApiVersions = true;
});

[ApiController]
[Route("labauthoring/v{version:apiVersion}/labprofiles")] // {domain} segment is fixed per service, not per-request
[ApiVersion("1.0")]
public class LabProfilesController : ControllerBase { /* ... */ }
```

## Response Shape (ADR-API0003 — supersedes API0001 Standards 5 & 5a)

**Single-resource responses are bare** — the resource object *is* the body. No `{data, status, timestamp}` wrapper; the HTTP status line and `Date`/`ETag` headers carry that metadata.

**Collection responses carry data + a cursor only** — no `totalCount`, `totalPages`, `status`, or `timestamp`:

```json
{
  "data": [ /* bare resource objects */ ],
  "pagination": { "nextCursor": "eyJpZCI6Li4uXX0=", "hasMore": true }
}
```

- Request pagination via `?limit=&cursor=`. Default `limit` = 25, max = 100; `limit > 100` → `400`.
- Cursors encode the ordered key + direction, remain valid ≥ 24 hours, and there is **no `previousCursor` in v1** (forward-only).
- Requesting past the end returns an empty `data` array, never `404`.
- **Cursor pagination is the standard and the end goal.** You may encounter offset pagination (`pageIndex`/`pageSize`/`totalCount`) in code that predates this ADR — treat it as a legacy pattern to migrate away from, not a second sanctioned option, when touching that code.
- HATEOAS `_links` (API0001 Standard 9) still apply, now as a top-level field on the bare resource and per item in `data[]`. Always include `self`; include state-dependent links (e.g. `launch`) only when applicable.
- `?include=` (API0001 Standard 5b) and `?fields=` sparse fieldsets continue to compose with this shape.

```csharp
public record Page<T>(IReadOnlyList<T> Data, PaginationInfo Pagination);
public record PaginationInfo(string? NextCursor, bool HasMore);
```

## Include-Parameter Validation (ADR-API0004 — supersedes API0001 Standard 5b's silent-ignore rule)

An `include` token an endpoint does not recognize returns `400 Bad Request` as an RFC 9457 Problem Details response — **never** silently ignored. Derive the accepted-token set and the error message's enumerated list from **one shared source** (a flags enum), match case-insensitively by declared name only (not `Enum.Parse`, which would also accept numeric values), and validate before any downstream work begins.

```csharp
[Flags]
public enum LabProfileInclude { None = 0, InstructionSets = 1, Skills = 2 }

public class LabProfileIncludeParser
{
    public virtual Result<LabProfileInclude> Parse(string? includeParam)
    {
        if (string.IsNullOrEmpty(includeParam))
            return Result.Success(LabProfileInclude.None);

        var result = LabProfileInclude.None;
        foreach (var token in includeParam.Split(',', StringSplitOptions.TrimEntries))
        {
            var match = Enum.GetNames<LabProfileInclude>()
                .FirstOrDefault(name => string.Equals(name, token, StringComparison.OrdinalIgnoreCase));

            if (match is null)
                return Result.Failure<LabProfileInclude>(new ResultError(
                    ResultErrorKind.Validation,
                    "include.invalid_value",
                    $"Unsupported include value '{token}'. Supported values: {string.Join(", ", Enum.GetNames<LabProfileInclude>().Where(n => n != "None"))}."));

            result |= Enum.Parse<LabProfileInclude>(match);
        }

        return Result.Success(result);
    }
}
```

An omitted or empty `include` means "no expansion requested" — that leniency is unchanged.

## Object-Level Authorization Disclosure (ADR-API0005 — supersedes part of API0001 Standard 8)

- **Resource-scoped denial** (the outcome depends on *which* resource ID was requested — "can this caller view lab profile #123?") → `404 Not Found`, **identical** to the true-not-found response. A `403` here would itself disclose that the resource exists to a caller who isn't entitled to see it (OWASP API1:2023, Broken Object Level Authorization).
- **Endpoint/action-scoped denial** (the outcome is the same regardless of resource ID — "does this caller hold the required role at all?") → `403 Forbidden`, unchanged. It discloses nothing resource-specific.
- Log resource-level denials server-side (warn level) for audit purposes — the response body itself carries nothing distinguishing "doesn't exist" from "exists, not yours."
- Implementation pattern: both the not-found path and the access-denied path throw the **same** `NotFoundException`, differing only in a server-side log line that never reaches the HTTP response:

```csharp
public virtual async Task<Result<LabProfileSummary>> GetLabProfileSummaryAsync(int id, CancellationToken cancellationToken)
{
    var profile = await _repo.LabProfileSingleOrDefaultByIdAsync(id, cancellationToken);
    if (profile is null)
        return Result.Failure<LabProfileSummary>(new ResultError(ResultErrorKind.NotFound, "labprofile.not_found", $"Lab profile {id} was not found"));

    if (!await _authz.UserCanViewAsync(profile, cancellationToken))
    {
        logger.LogWarning("Access denied for lab profile {LabProfileId} requested by user {UserId}.", id, _securityContext.UserId);
        // Same NotFound outcome as genuinely missing — never Forbidden here.
        return Result.Failure<LabProfileSummary>(new ResultError(ResultErrorKind.NotFound, "labprofile.not_found", $"Lab profile {id} was not found"));
    }

    return Result.Success(MapToSummary(profile));
}
```

## Error Responses — RFC 9457 Problem Details (ADR-API0001 Standard 8, unaffected by the supersessions above)

```json
{
  "type": "https://api.skillable.com/problems/validation-error",
  "title": "Bad Request",
  "status": 400,
  "detail": "Unsupported include value 'instructionSet'. Supported values: instructionsets, skills.",
  "instance": "/labauthoring/v1/labprofiles/12345"
}
```

- `400` invalid input/query params, `401` missing/invalid credentials, `404` not found **or** a resource-scoped authorization denial (see above), `500` unhandled server failure (generic message, no stack traces).
- `403` is reserved for endpoint/action-scoped denials only (see above) — never for a specific resource's visibility.
- Clients match on the failure `Code` from `Result<T>`/`ResultError`, never on `Message` or `detail` text.

## Controller-Based APIs

```csharp
[ApiController]
[Route("labauthoring/v{version:apiVersion}/labprofiles")]
[ApiVersion("1.0")]
public class LabProfilesController(
    ILabProfileService labProfileService) : ControllerBase
{
    [HttpGet("{id}")]
    public async Task<IActionResult> GetLabProfile(int id, [FromQuery] string? include, CancellationToken cancellationToken)
    {
        var result = await labProfileService.GetLabProfileAsync(id, include, cancellationToken);
        return result.ToActionResult(this); // bare resource on success; ResultErrorKind maps to the right status
    }

    [HttpGet]
    public async Task<IActionResult> GetLabProfiles(
        [FromQuery] string? cursor, [FromQuery] int limit = 25, CancellationToken cancellationToken = default)
    {
        var result = await labProfileService.GetLabProfilesAsync(cursor, limit, cancellationToken);
        return result.ToActionResult(this); // Page<LabProfileDto> -> { data, pagination }
    }

    [HttpPost]
    public async Task<IActionResult> CreateLabProfile([FromBody] CreateLabProfileRequest request, CancellationToken cancellationToken)
    {
        var result = await labProfileService.CreateLabProfileAsync(request, cancellationToken);
        return result.ToActionResult(this); // 201 + Location on success; Conflict/Validation mapped on failure
    }
}
```

`result.ToActionResult(this)` (the `ResultActionExtensions` anti-corruption layer, per ADR-IDP002) is the **only** way a controller produces a response from a `Result<T>`/`Result` — no manual `BadRequest(...)`/`NotFound(...)` calls on a service outcome. Controllers stay thin: no business logic, no direct DB calls, no manual status-code mapping.

## Minimal APIs

Lightweight alternative to controllers for simple APIs and microservices.

```csharp
var labProfilesApi = app.MapGroup("/labauthoring/v{version:apiVersion}/labprofiles")
    .WithTags("LabProfiles")
    .RequireAuthorization();

labProfilesApi.MapGet("/{id}", async (int id, string? include, ILabProfileService service, CancellationToken ct) =>
{
    var result = await service.GetLabProfileAsync(id, include, ct);
    return result.ToMinimalApiResult(); // same ResultErrorKind -> status mapping, IResult-flavored
})
.WithName("GetLabProfile");

labProfilesApi.MapPost("/", async (CreateLabProfileRequest request, ILabProfileService service, CancellationToken ct) =>
{
    var result = await service.CreateLabProfileAsync(request, ct);
    return result.ToMinimalApiResult();
})
.WithName("CreateLabProfile");
```

### Request Validation with Filters

```csharp
public class ValidationFilter<T>(IValidator<T> validator) : IEndpointFilter where T : class
{
    public async ValueTask<object?> InvokeAsync(EndpointFilterInvocationContext context, EndpointFilterDelegate next)
    {
        var request = context.Arguments.OfType<T>().FirstOrDefault();

        if (request is null)
            return Results.Problem(statusCode: StatusCodes.Status400BadRequest, title: "Invalid request");

        var validationResult = await validator.ValidateAsync(request);

        if (!validationResult.IsValid)
        {
            return Results.ValidationProblem(
                validationResult.ToDictionary()); // RFC 9457-compatible validation problem response
        }

        return await next(context);
    }
}

// Usage
labProfilesApi.MapPost("/", async (CreateLabProfileRequest request, ILabProfileService service, CancellationToken ct) =>
{
    var result = await service.CreateLabProfileAsync(request, ct);
    return result.ToMinimalApiResult();
})
.AddEndpointFilter<ValidationFilter<CreateLabProfileRequest>>();
```

### When to Use Each

**Controllers** — Use for: large APIs (10+ endpoints), complex routing, action filters/custom attributes, traditional MVC.

**Minimal APIs** — Use for: microservices, simple CRUD (5-10 endpoints), performance-critical scenarios, serverless/containerized deployments.

## Custom Middleware Pattern

Do NOT create request-logging middleware — telemetry (Application Insights, OpenTelemetry) captures request method, path, status code, and duration automatically (see `common/logging.md`). Use custom middleware only for cross-cutting concerns not covered by telemetry:

```csharp
public class TenantResolutionMiddleware(RequestDelegate next)
{
    public async Task InvokeAsync(HttpContext context, ITenantProvider tenantProvider)
    {
        var tenantId = context.Request.Headers["X-Tenant-Id"].FirstOrDefault();

        if (string.IsNullOrEmpty(tenantId))
        {
            context.Response.StatusCode = StatusCodes.Status400BadRequest;
            await context.Response.WriteAsJsonAsync(new
            {
                type = "https://api.skillable.com/problems/missing-header",
                title = "Bad Request",
                status = 400,
                detail = "X-Tenant-Id header is required.",
                instance = context.Request.Path.Value,
            });
            return;
        }

        tenantProvider.SetTenant(tenantId);
        await next(context);
    }
}

// Extension method for registration
public static class TenantResolutionMiddlewareExtensions
{
    public static IApplicationBuilder UseTenantResolution(this IApplicationBuilder builder)
    {
        return builder.UseMiddleware<TenantResolutionMiddleware>();
    }
}
```

## Response Caching

```csharp
// Output caching (.NET 7+)
builder.Services.AddOutputCache(options =>
{
    options.AddBasePolicy(builder => builder.Cache());
    options.AddPolicy("Expire30s", builder => builder.Expire(TimeSpan.FromSeconds(30)));
});

app.UseOutputCache();

// Usage
labProfilesApi.MapGet("/", async (ILabProfileService service, CancellationToken ct) =>
{
    var result = await service.GetLabProfilesAsync(null, 25, ct);
    return result.ToMinimalApiResult();
})
.CacheOutput("Expire30s");
```

## Swagger/OpenAPI

```csharp
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen(options =>
{
    options.SwaggerDoc("v1", new OpenApiInfo
    {
        Title = "My API",
        Version = "v1",
        Description = "A sample ASP.NET Core Web API"
    });

    var xmlFile = $"{Assembly.GetExecutingAssembly().GetName().Name}.xml";
    var xmlPath = Path.Combine(AppContext.BaseDirectory, xmlFile);
    options.IncludeXmlComments(xmlPath);
});

if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}
```

Enable XML documentation in `.csproj`:
```xml
<PropertyGroup>
  <GenerateDocumentationFile>true</GenerateDocumentationFile>
  <NoWarn>$(NoWarn);1591</NoWarn>
</PropertyGroup>
```
