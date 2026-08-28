# C# Presentation Layer Rules

> Rules for Web.Core assemblies: controllers, minimal APIs, middleware, API versioning, response caching, and Swagger.

---
paths: ["**/*.cs", "**/*.csx"]
---

## Controller-Based APIs

Use the `[ApiController]` attribute to enable automatic model validation and binding behaviors.

**Automatically enabled features:**
- Model validation errors return 400 Bad Request automatically
- Binding source parameter inference (`[FromBody]`, `[FromQuery]`, etc.)
- Multipart/form-data request inference
- Problem details for error responses

```csharp
[ApiController]
[Route("api/[controller]")]
public class UsersController(
    IUserService userService,
    ILogger<UsersController> logger) : ControllerBase
{
    [HttpGet]
    [ProducesResponseType(typeof(ApiResponse<IReadOnlyList<UserDto>>), StatusCodes.Status200OK)]
    public async Task<ActionResult<ApiResponse<IReadOnlyList<UserDto>>>> GetUsers(
        [FromQuery] int page,
        [FromQuery] int limit,
        CancellationToken cancellationToken)
    {
        var users = await userService.GetUsersAsync(page, limit, cancellationToken);
        return Ok(ApiResponse<IReadOnlyList<UserDto>>.Ok(users));
    }

    [HttpGet("{id}")]
    [ProducesResponseType(typeof(ApiResponse<UserDto>), StatusCodes.Status200OK)]
    [ProducesResponseType(typeof(ApiResponse<UserDto>), StatusCodes.Status404NotFound)]
    public async Task<ActionResult<ApiResponse<UserDto>>> GetUser(
        int id,
        CancellationToken cancellationToken)
    {
        var user = await userService.GetUserByIdAsync(id, cancellationToken);

        if (user is null)
            return NotFound(ApiResponse<UserDto>.Fail($"User not found (UserId: {id}).", HttpStatusCode.NotFound));

        return Ok(ApiResponse<UserDto>.Ok(user));
    }

    [HttpPost]
    [ProducesResponseType(typeof(ApiResponse<UserDto>), StatusCodes.Status201Created)]
    [ProducesResponseType(StatusCodes.Status400BadRequest)]
    public async Task<ActionResult<ApiResponse<UserDto>>> CreateUser(
        [FromBody] CreateUserRequest request,
        CancellationToken cancellationToken)
    {
        var user = await userService.CreateUserAsync(request, cancellationToken);

        return CreatedAtAction(
            nameof(GetUser),
            new { id = user.Id },
            ApiResponse<UserDto>.Ok(user));
    }

    [HttpPut("{id}")]
    [ProducesResponseType(typeof(ApiResponse<UserDto>), StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<ActionResult<ApiResponse<UserDto>>> UpdateUser(
        int id,
        [FromBody] UpdateUserRequest request,
        CancellationToken cancellationToken)
    {
        var user = await userService.UpdateUserAsync(id, request, cancellationToken);
        return Ok(ApiResponse<UserDto>.Ok(user));
    }

    [HttpDelete("{id}")]
    [ProducesResponseType(StatusCodes.Status204NoContent)]
    public async Task<ActionResult> DeleteUser(int id, CancellationToken cancellationToken)
    {
        await userService.DeleteUserAsync(id, cancellationToken);
        return NoContent();
    }
}
```

### Result Pattern in Controllers

When services return `Result<T>` (see `csharp/domain.md`), map to `ApiResponse`:

```csharp
[HttpPost]
public async Task<ActionResult<ApiResponse<UserDto>>> CreateUser(CreateUserRequest request, CancellationToken cancellationToken)
{
    var result = await _userService.CreateUserAsync(request, cancellationToken);

    return result.IsSuccess
        ? Ok(ApiResponse<UserDto>.Ok(MapToDto(result.Value!)))
        : BadRequest(ApiResponse<UserDto>.Fail(result.Error!, HttpStatusCode.BadRequest));
}
```

## Minimal APIs

Lightweight alternative to controllers for simple APIs and microservices.

```csharp
// Route groups for organization
var usersApi = app.MapGroup("/api/users")
    .WithTags("Users")
    .RequireAuthorization();

usersApi.MapGet("/", async (IUserService userService, int page, int limit) =>
{
    var users = await userService.GetUsersAsync(page, limit);
    return Results.Ok(ApiResponse<IReadOnlyList<UserDto>>.Ok(users));
})
.WithName("GetUsers")
.Produces<ApiResponse<IReadOnlyList<UserDto>>>(StatusCodes.Status200OK);

usersApi.MapGet("/{id}", async (int id, IUserService userService) =>
{
    var user = await userService.GetUserByIdAsync(id);

    return user is null
        ? Results.NotFound(ApiResponse<UserDto>.Fail($"User not found (UserId: {id}).", HttpStatusCode.NotFound))
        : Results.Ok(ApiResponse<UserDto>.Ok(user));
})
.WithName("GetUserById")
.Produces<ApiResponse<UserDto>>(StatusCodes.Status200OK)
.Produces<ApiResponse<UserDto>>(StatusCodes.Status404NotFound);

usersApi.MapPost("/", async (CreateUserRequest request, IUserService userService) =>
{
    var user = await userService.CreateUserAsync(request);

    return Results.CreatedAtRoute(
        "GetUserById",
        new { id = user.Id },
        ApiResponse<UserDto>.Ok(user));
})
.WithName("CreateUser")
.Produces<ApiResponse<UserDto>>(StatusCodes.Status201Created)
.Produces(StatusCodes.Status400BadRequest);
```

### Request Validation with Filters

```csharp
public class ValidationFilter<T>(IValidator<T> validator) : IEndpointFilter where T : class
{
    public async ValueTask<object?> InvokeAsync(EndpointFilterInvocationContext context, EndpointFilterDelegate next)
    {
        var request = context.Arguments.OfType<T>().FirstOrDefault();

        if (request is null)
            return Results.BadRequest("Invalid request");

        var validationResult = await validator.ValidateAsync(request);

        if (!validationResult.IsValid)
        {
            return Results.BadRequest(new
            {
                Errors = validationResult.Errors.Select(e => new { e.PropertyName, e.ErrorMessage })
            });
        }

        return await next(context);
    }
}

// Usage
usersApi.MapPost("/", async (CreateUserRequest request, IUserService userService) =>
{
    var user = await userService.CreateUserAsync(request);
    return Results.Created($"/api/users/{user.Id}", user);
})
.AddEndpointFilter<ValidationFilter<CreateUserRequest>>();
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
            await context.Response.WriteAsJsonAsync(
                ApiResponse<object>.Fail("X-Tenant-Id header is required.", HttpStatusCode.BadRequest));
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

## API Versioning

```csharp
// Install: Asp.Versioning.Http
builder.Services.AddApiVersioning(options =>
{
    options.DefaultApiVersion = new ApiVersion(1, 0);
    options.AssumeDefaultVersionWhenUnspecified = true;
    options.ReportApiVersions = true;
    options.ApiVersionReader = new UrlSegmentApiVersionReader();
});

[ApiController]
[Route("api/v{version:apiVersion}/[controller]")]
[ApiVersion("1.0")]
public class UsersV1Controller : ControllerBase
{
    [HttpGet]
    public IActionResult GetUsers() => Ok(new[] { "User1", "User2" });
}

[ApiController]
[Route("api/v{version:apiVersion}/[controller]")]
[ApiVersion("2.0")]
public class UsersV2Controller : ControllerBase
{
    [HttpGet]
    public IActionResult GetUsers() => Ok(new[] { new { Id = 1, Name = "User1" } });
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
app.MapGet("/api/users", async (IUserService userService) =>
{
    var users = await userService.GetUsersAsync();
    return Results.Ok(users);
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
