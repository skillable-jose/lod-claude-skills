# C# Service Layer Rules

> Rules for Implementation assemblies: service implementations, validation, business logic, Options pattern, and DI registration.

---
paths: ["**/*.cs", "**/*.csx"]
---

## Service Implementations

Implement interfaces defined in Abstractions. Services operate on domain models, not DTOs. Use primary constructors for dependency injection.

### TimeProvider (CRITICAL)

**Never use `DateTime.Now`, `DateTime.UtcNow`, or `DateTimeOffset.UtcNow` directly in services.** Inject `TimeProvider` (.NET 8+) via the primary constructor and call `timeProvider.GetUtcNow()` instead. This makes time-dependent logic deterministic and testable.

```csharp
// CORRECT: Inject TimeProvider
public class UserService(
    IUserRepository userRepository,
    TimeProvider timeProvider) : IUserService
{
    public virtual async Task<User> CreateUserAsync(CreateUserRequest request, CancellationToken cancellationToken)
    {
        var user = new User
        {
            Name = request.Name,
            Email = request.Email,
            CreatedAtUtc = timeProvider.GetUtcNow().UtcDateTime
        };
        return await userRepository.UserAddAsync(user, cancellationToken);
    }
}

// WRONG: Direct DateTime usage — untestable, non-deterministic
var user = new User { CreatedAtUtc = DateTime.UtcNow };  // ❌
```

Register `TimeProvider.System` as a singleton in the DI container (see [Extension Method Patterns](#extension-method-patterns-for-service-registration)):

```csharp
services.AddSingleton(TimeProvider.System);
```

In tests, use `Microsoft.Extensions.Time.Testing.FakeTimeProvider` to control time:

```csharp
var fakeTime = new FakeTimeProvider(new DateTimeOffset(2025, 1, 15, 10, 0, 0, TimeSpan.Zero));
var service = new UserService(mockRepository, fakeTime);
// fakeTime.Advance(TimeSpan.FromHours(1)); — to simulate time passing
```

### Example Service — Result<T> per the reservation rule (ADR-IDP001)

All of these outcomes — duplicate email, not-found — are "the system did its job and the answer was no," so they return `Result<T>`/`Result`, never throw. See `csharp/domain.md` for the `Result<T>`/`ResultError`/`ResultErrorKind` shape and the full reservation rule. **There is one canonical style for expected outcomes — do not mix a `Result`-returning method with a throwing one for the same kind of failure.**

```csharp
public class UserService(
    IUserRepository userRepository,
    TimeProvider timeProvider,
    ILogger<UserService> logger) : IUserService
{
    public virtual async Task<Result<User>> GetUserByIdAsync(int id, CancellationToken cancellationToken)
    {
        var user = await userRepository.UserSingleOrDefaultByIdAsync(id, cancellationToken);
        if (user is null)
            return Result.Failure<User>(new ResultError(
                ResultErrorKind.NotFound, "user.not_found", $"User {id} was not found"));

        return Result.Success(user);
    }

    public virtual async Task<Result<User>> CreateUserAsync(CreateUserRequest request, CancellationToken cancellationToken)
    {
        // Business logic: Check for duplicate email — SingleOrDefault since absence is expected
        var existingUser = await userRepository.UserSingleOrDefaultByEmailAsync(request.Email, cancellationToken);
        if (existingUser is not null)
        {
            logger.LogInformation(
                "User creation rejected: email conflict. Email={Email}", request.Email);
            return Result.Failure<User>(new ResultError(
                ResultErrorKind.Conflict, "user.duplicate_email", $"Email {request.Email} already exists"));
        }

        var user = new User
        {
            Name = request.Name,
            Email = request.Email,
            CreatedAtUtc = timeProvider.GetUtcNow().UtcDateTime
        };
        var created = await userRepository.UserAddAsync(user, cancellationToken);
        return Result.Success(created);
    }

    public virtual async Task<Result<User>> UpdateUserAsync(int id, UpdateUserRequest request, CancellationToken cancellationToken)
    {
        // NotFoundException from Single is a repository-internal signal — caught and
        // translated here into Result.Failure(NotFound), never left to propagate.
        User user;
        try
        {
            user = await userRepository.UserSingleByIdAsync(id, cancellationToken);
        }
        catch (NotFoundException)
        {
            return Result.Failure<User>(new ResultError(
                ResultErrorKind.NotFound, "user.not_found", $"User {id} was not found"));
        }

        var updated = user with
        {
            Name = request.Name,
            Email = request.Email,
            UpdatedAtUtc = timeProvider.GetUtcNow().UtcDateTime
        };
        var saved = await userRepository.UserUpdateAsync(updated, cancellationToken);
        return Result.Success(saved);
    }

    public virtual async Task<Result> DeleteUserAsync(int id, CancellationToken cancellationToken)
    {
        await userRepository.UserDeleteAsync(id, cancellationToken);
        return Result.Success();
    }
}
```

Every rejection is **logged once, at this layer, at info/warn level** (per ADR-IDP001 Part 1) — the controller and any central handler must not re-log it. See `csharp/presentation.md` for how a controller translates `Result<T>` to HTTP.

## FluentValidation

Use FluentValidation to keep request records clean. Validation lives in dedicated validator classes — never as Data Annotation attributes on records.

```csharp
// Validators — one per request type
public class CreateUserRequestValidator : AbstractValidator<CreateUserRequest>
{
    public CreateUserRequestValidator()
    {
        RuleFor(x => x.Name).NotEmpty().MaximumLength(100);
        RuleFor(x => x.Email).NotEmpty().EmailAddress().MaximumLength(255);
        RuleFor(x => x.Age).InclusiveBetween(18, 120);
    }
}

public class UpdateUserRequestValidator : AbstractValidator<UpdateUserRequest>
{
    public UpdateUserRequestValidator()
    {
        RuleFor(x => x.Name).NotEmpty().MaximumLength(100);
        RuleFor(x => x.Email).NotEmpty().EmailAddress().MaximumLength(255);
    }
}
```

### Dependent/Async Validation Rules

FluentValidation validators keep traditional constructors — rules must be defined in the constructor body, which primary constructors do not provide.

```csharp
public class CreateUserRequestValidator : AbstractValidator<CreateUserRequest>
{
    public CreateUserRequestValidator(IUserRepository userRepository)
    {
        RuleFor(x => x.Email)
            .NotEmpty()
            .EmailAddress()
            .MustAsync(async (email, cancellation) =>
            {
                var existing = await userRepository.UserSingleOrDefaultByEmailAsync(email, cancellation);
                return existing is null;
            })
            .WithMessage("Email already exists");
    }
}
```

### DI Registration for Validators

Wire inside the assembly's `Add{Feature}` extension method:

```csharp
services.AddValidatorsFromAssemblyContaining<CreateUserRequestValidator>();
```

For Minimal APIs, use the `ValidationFilter<T>` endpoint filter (see `csharp/presentation.md`). For controller-based APIs, wire FluentValidation into the MVC pipeline:

```csharp
services.AddFluentValidationAutoValidation();
```

## Options Pattern

Use the Options pattern for strongly-typed configuration:

```csharp
// Configuration class
public class DatabaseOptions
{
    public const string SectionName = "Database";

    public string ConnectionString { get; set; } = string.Empty;
    public int MaxRetryCount { get; set; } = 3;
    public int CommandTimeout { get; set; } = 30;
}

public class EmailOptions
{
    public const string SectionName = "Email";

    public string SmtpServer { get; set; } = string.Empty;
    public int Port { get; set; } = 587;
    public string FromAddress { get; set; } = string.Empty;
}

// appsettings.json
{
  "Database": {
    "ConnectionString": "Server=localhost;Database=MyApp;",
    "MaxRetryCount": 3,
    "CommandTimeout": 30
  },
  "Email": {
    "SmtpServer": "smtp.example.com",
    "Port": 587,
    "FromAddress": "noreply@example.com"
  }
}

// Inside Add{Feature} extension method
services.Configure<DatabaseOptions>(
    configuration.GetSection(DatabaseOptions.SectionName));

services.Configure<EmailOptions>(
    configuration.GetSection(EmailOptions.SectionName));

// Usage in service with primary constructor
public class EmailService(
    IOptions<EmailOptions> emailOptions,
    ILogger<EmailService> logger)
{
    public async Task SendEmailAsync(string to, string subject, string body, CancellationToken cancellationToken)
    {
        logger.LogInformation(
            "Sending email (To: {To}, SmtpServer: {SmtpServer}, Port: {Port}).",
            to,
            emailOptions.Value.SmtpServer,
            emailOptions.Value.Port);

        // Send email using emailOptions.Value
    }
}
```

For validation, use `IValidateOptions<T>`:

```csharp
public class DatabaseOptionsValidator : IValidateOptions<DatabaseOptions>
{
    public ValidateOptionsResult Validate(string? name, DatabaseOptions options)
    {
        if (string.IsNullOrWhiteSpace(options.ConnectionString))
            return ValidateOptionsResult.Fail("ConnectionString is required");

        if (options.MaxRetryCount < 0)
            return ValidateOptionsResult.Fail("MaxRetryCount must be >= 0");

        return ValidateOptionsResult.Success;
    }
}

// Inside Add{Feature} extension method
services.AddSingleton<IValidateOptions<DatabaseOptions>, DatabaseOptionsValidator>();
```

## Extension Method Patterns for Service Registration

Each assembly exposes its own DI wiring via an `IServiceCollection` extension method.

**Conventions**:
- **Namespace**: Always `Microsoft.Extensions.DependencyInjection` — this is the standard .NET convention so callers discover the method without extra `using` statements
- **Method name**: `Add{Feature}(...)` — describes what capability the assembly adds (e.g., `AddPersistence`, `AddUserServices`, `AddWebCore`)
- **File location**: `Infrastructure/{Feature}ServiceCollectionExtensions.cs` inside the assembly
- **Class name**: `{Feature}ServiceCollectionExtensions`

```csharp
// TodoApp.Implementation/Infrastructure/UserServiceCollectionExtensions.cs
namespace Microsoft.Extensions.DependencyInjection;

public static class UserServiceCollectionExtensions
{
    public static IServiceCollection AddUserServices(this IServiceCollection services)
    {
        services.AddScoped<IUserService, UserService>();

        return services;
    }
}

// Program.cs — callers compose features without extra using statements
builder.Services
    .AddUserServices()
    .AddPersistence(builder.Configuration);
```
