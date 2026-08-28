# C# Domain Layer Rules

> Rules for Abstractions assemblies: domain models, interfaces, contracts, value objects, and exceptions.

---
paths: ["**/*.cs", "**/*.csx"]
---

## Domain Models

Domain models are persistence-ignorant classes defined in Abstractions. They MUST NOT contain ORM attributes (`[Table]`, `[Column]`), navigation properties, or framework-specific code. ORM entity classes belong in the Repository project (see `csharp/persistence.md`).

```csharp
// CORRECT: Clean domain model in Abstractions
public class User
{
    public int Id { get; set; }
    public string Email { get; set; } = string.Empty;
    public string FirstName { get; set; } = string.Empty;
    public string LastName { get; set; } = string.Empty;
    public UserRole Role { get; set; }
    public bool IsActive { get; set; }
    public DateTime CreatedAtUtc { get; set; }
    public DateTime? UpdatedAtUtc { get; set; }
}

// WRONG: ORM entity leaking into Abstractions
public class User
{
    [Table("Users")]
    public int Id { get; set; }
    public ICollection<Order> Orders { get; set; } = [];  // Navigation property — belongs in Repository
}
```

## Repository Interfaces

Repository interfaces operate on **domain models**, NOT on EF Core entity classes. Use standalone interfaces (no generic `IRepository<T>` base — mapping makes that impractical).

**Naming Convention** (Repositories only — does NOT apply to Services): Methods MUST start with entity type for IDE grouping (e.g., `UserSingleByIdAsync`, `UserAddAsync`).

**Single vs SingleOrDefault**: `Single` methods throw `NotFoundException` when the entity is not found. `SingleOrDefault` methods return `null`. Use `Single` when absence is exceptional; use `SingleOrDefault` when absence is a valid outcome.

```csharp
public interface IUserRepository
{
    // Single — throws NotFoundException when not found
    Task<User> UserSingleByIdAsync(int id, CancellationToken cancellationToken);
    Task<User> UserSingleByEmailAsync(string email, CancellationToken cancellationToken);

    // SingleOrDefault — returns null when not found
    Task<User?> UserSingleOrDefaultByIdAsync(int id, CancellationToken cancellationToken);
    Task<User?> UserSingleOrDefaultByEmailAsync(string email, CancellationToken cancellationToken);

    Task<IReadOnlyList<User>> UserGetAllAsync(CancellationToken cancellationToken);
    Task<IReadOnlyList<User>> UserGetActiveAsync(CancellationToken cancellationToken);
    Task<User> UserAddAsync(User user, CancellationToken cancellationToken);
    Task<User> UserUpdateAsync(User user, CancellationToken cancellationToken);
    Task UserDeleteAsync(int id, CancellationToken cancellationToken);
}
```

## Service Interfaces

Service interfaces use natural application-level naming (e.g., `GetUserByIdAsync`, `CreateUserAsync`) — NOT entity-first naming like repositories. Service interfaces operate on domain models, not DTOs.

```csharp
public interface IUserService
{
    Task<User?> GetUserByIdAsync(int id, CancellationToken cancellationToken);
    Task<User> CreateUserAsync(CreateUserRequest request, CancellationToken cancellationToken);
    Task<User> UpdateUserAsync(int id, UpdateUserRequest request, CancellationToken cancellationToken);
    Task DeleteUserAsync(int id, CancellationToken cancellationToken);
}
```

## Request/Response Records

Keep request DTOs clean and positional. Never use Data Annotation attributes on records — validation lives in dedicated FluentValidation validator classes (see `csharp/services.md`).

```csharp
public record CreateUserRequest(string Name, string Email, int Age);
public record UpdateUserRequest(string Name, string Email);
```

## Result Pattern

For operations where failure is expected, use a `Result<T>` type instead of exceptions:

```csharp
public record Result<T>
{
    public bool IsSuccess { get; init; }
    public T? Value { get; init; }
    public string? Error { get; init; }

    public static Result<T> Success(T value) =>
        new() { IsSuccess = true, Value = value };

    public static Result<T> Failure(string error) =>
        new() { IsSuccess = false, Error = error };
}
```

## API Response Envelope

Use a consistent envelope for all API responses. `StatusCode` is ALWAYS required.

```csharp
public record ApiResponse<T>
{
    public bool Success { get; init; }
    public T? Data { get; init; }
    public string? Error { get; init; }
    public HttpStatusCode StatusCode { get; init; }

    public static ApiResponse<T> Ok(T data) =>
        new() { Success = true, Data = data, StatusCode = HttpStatusCode.OK };

    public static ApiResponse<T> Fail(string error, HttpStatusCode statusCode) =>
        new() { Success = false, Error = error, StatusCode = statusCode };
}
```

## Specification Interface

For complex query encapsulation (implementation in `csharp/persistence.md`):

```csharp
public interface ISpecification<T>
{
    Expression<Func<T, bool>> Criteria { get; }
    List<Expression<Func<T, object>>> Includes { get; }
    Expression<Func<T, object>>? OrderBy { get; }
}
```

## Custom Exceptions

Define domain-specific exceptions in Abstractions:

```csharp
public class NotFoundException : Exception
{
    public NotFoundException(string message) : base(message) { }
}

public class DuplicateEmailException : Exception
{
    public DuplicateEmailException(string message) : base(message) { }
}
```
