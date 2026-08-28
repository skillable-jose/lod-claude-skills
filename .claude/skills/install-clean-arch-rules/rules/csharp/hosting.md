# C# Hosting Layer Rules

> Rules for Web.Server and Web.Api assemblies: Program.cs pipeline, background services, health checks, containerization, and global exception handling.

---
paths: ["**/*.cs", "**/*.csx"]
---

## Middleware Pipeline

**CRITICAL:** Middleware order matters. Use this sequence:

```csharp
var app = builder.Build();

// 1. Exception handling (first to catch all errors)
app.UseExceptionHandler("/error");
app.UseHsts();  // Production only

// 2. HTTPS redirection
app.UseHttpsRedirection();

// 3. Static files (before routing)
app.UseStaticFiles();

// 4. Routing
app.UseRouting();

// 5. CORS (after routing, before auth)
app.UseCors("AllowSpecificOrigin");

// 6. Authentication (before authorization)
app.UseAuthentication();

// 7. Authorization
app.UseAuthorization();

// 8. Response caching
app.UseResponseCaching();

// 9. Endpoints (last)
app.MapControllers();

app.Run();
```

## Global Exception Handler

All unhandled exceptions are caught here, logged, and returned as a consistent `ApiResponse.Fail(...)` envelope. Controllers should **not** use try/catch for response shaping — let exceptions bubble up to this handler (see `common/logging.md`).

```csharp
app.UseExceptionHandler(errorApp =>
{
    errorApp.Run(async context =>
    {
        var exceptionHandlerFeature = context.Features.Get<IExceptionHandlerFeature>();
        var exception = exceptionHandlerFeature?.Error;

        var logger = context.RequestServices.GetRequiredService<ILogger<Program>>();
        logger.LogError(exception, "Unhandled exception.");

        var statusCode = exception switch
        {
            NotFoundException => StatusCodes.Status404NotFound,
            UnauthorizedAccessException => StatusCodes.Status401Unauthorized,
            _ => StatusCodes.Status500InternalServerError
        };

        context.Response.StatusCode = statusCode;
        context.Response.ContentType = "application/json";

        var message = statusCode == StatusCodes.Status500InternalServerError
            ? "Internal server error"
            : exception?.Message ?? "An error occurred";

        await context.Response.WriteAsJsonAsync(
            ApiResponse<object>.Fail(message, (HttpStatusCode)statusCode));
    });
});
```

## Background Services

```csharp
public class DataCleanupService(
    IServiceProvider serviceProvider,
    ILogger<DataCleanupService> logger) : BackgroundService
{
    private readonly TimeSpan _interval = TimeSpan.FromHours(24);

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        logger.LogInformation("Data cleanup service started.");

        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                await CleanupExpiredDataAsync(stoppingToken);
                await Task.Delay(_interval, stoppingToken);
            }
            catch (OperationCanceledException)
            {
                break;
            }
            catch (Exception ex)
            {
                logger.LogError(ex, "Error in data cleanup service.");
                await Task.Delay(TimeSpan.FromMinutes(5), stoppingToken);
            }
        }

        logger.LogInformation("Data cleanup service stopped.");
    }

    private async Task CleanupExpiredDataAsync(CancellationToken cancellationToken)
    {
        // Create scope to resolve scoped services (repositories, services)
        using var scope = serviceProvider.CreateScope();
        var tempDataRepository = scope.ServiceProvider.GetRequiredService<ITempDataRepository>();

        var cutoffDate = DateTime.UtcNow.AddDays(-30);
        var deletedCount = await tempDataRepository.TempDataDeleteExpiredAsync(cutoffDate, cancellationToken);

        logger.LogInformation("Expired records cleaned up (Count: {Count}).", deletedCount);
    }
}

// Registration
builder.Services.AddHostedService<DataCleanupService>();
```

## Health Checks

```csharp
builder.Services.AddHealthChecks()
    .AddDbContextCheck<ApplicationDbContext>()
    .AddUrlGroup(new Uri("https://api.example.com/health"), "External API")
    .AddCheck<CustomHealthCheck>("Custom Check");

// Custom health check
public class CustomHealthCheck(IUserRepository userRepository) : IHealthCheck
{
    public async Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context,
        CancellationToken cancellationToken)
    {
        try
        {
            var userCount = await userRepository.GetCountAsync(cancellationToken);

            return userCount > 0
                ? HealthCheckResult.Healthy($"Database has {userCount} users")
                : HealthCheckResult.Degraded("Database is empty");
        }
        catch (Exception ex)
        {
            return HealthCheckResult.Unhealthy("Database check failed", ex);
        }
    }
}

// Endpoint mapping
app.MapHealthChecks("/health");
app.MapHealthChecks("/health/ready", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("ready")
});
```

## Containerization

**Multi-stage Dockerfile:**

```dockerfile
# Build stage
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src

COPY ["MyApi/MyApi.csproj", "MyApi/"]
RUN dotnet restore "MyApi/MyApi.csproj"

COPY . .
WORKDIR "/src/MyApi"
RUN dotnet build "MyApi.csproj" -c Release -o /app/build

# Publish stage
FROM build AS publish
RUN dotnet publish "MyApi.csproj" -c Release -o /app/publish /p:UseAppHost=false

# Runtime stage
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS final
WORKDIR /app

RUN adduser --disabled-password --gecos "" appuser && chown -R appuser /app
USER appuser

COPY --from=publish /app/publish .

EXPOSE 8080
ENV ASPNETCORE_URLS=http://+:8080

ENTRYPOINT ["dotnet", "MyApi.dll"]
```

**docker-compose.yml:**

```yaml
version: '3.8'

services:
  api:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "5000:8080"
    environment:
      - ASPNETCORE_ENVIRONMENT=Development
      - ConnectionStrings__DefaultConnection=Server=db;Database=MyDb;User=sa;Password=YourStrong@Passw0rd;
    depends_on:
      - db

  db:
    image: mcr.microsoft.com/mssql/server:2022-latest
    environment:
      - ACCEPT_EULA=Y
      - SA_PASSWORD=YourStrong@Passw0rd
    ports:
      - "1433:1433"
    volumes:
      - sqldata:/var/opt/mssql

volumes:
  sqldata:
```

## Backend Development Checklist

Before committing ASP.NET Core backend code:
- [ ] Controllers use `[ApiController]` attribute or minimal APIs use route groups
- [ ] All endpoints have proper `[ProducesResponseType]` attributes or `.Produces()` calls
- [ ] Database queries use `AsNoTracking()` for read-only operations
- [ ] Eager loading used to avoid N+1 queries
- [ ] Middleware pipeline in correct order
- [ ] Background services properly inject scoped services via `IServiceProvider`
- [ ] Health checks configured for database and external dependencies
- [ ] Swagger/OpenAPI documentation enabled in Development
- [ ] Multi-stage Dockerfile optimized for production
- [ ] No hardcoded connection strings (use `IConfiguration`)
- [ ] Global exception handler configured
- [ ] CancellationToken passed to all async methods
