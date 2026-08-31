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

**This is a repository-internal calling convention, not a violation of the ADR-IDP001 reservation rule.** `NotFoundException` thrown here is caught and translated by the calling **domain service** method into `Result.Failure<T>(new ResultError(ResultErrorKind.NotFound, ...))` before it can reach a controller — "not found" is an expected outcome at the service boundary even though the repository expresses it as an exception internally. A `NotFoundException` must never propagate past the service layer unhandled.

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

> This shape is normative per **ADR-IDP001** (Error Handling and Result Pattern for Use Case Returns) — it is not a locally-invented convention. Do not alter field names or factory signatures without a corresponding ADR change.

Domain service methods that can fail in **expected** ways (validation, not-found, conflict, forbidden — see the reservation rule below) return `Result<T>` (or the non-generic `Result` for void use cases) instead of throwing:

```csharp
public class Result<T>
{
    public bool IsSuccess { get; }
    public bool IsFailure => !IsSuccess;
    public T? Value { get; }
    public ResultError? Error { get; }
}

public class Result
{
    // Same shape as Result<T>, without Value — for void use cases
    public bool IsSuccess { get; }
    public bool IsFailure => !IsSuccess;
    public ResultError? Error { get; }
}

public record ResultError(ResultErrorKind Kind, string Code, string Message, object? Details = null);

public enum ResultErrorKind
{
    NotFound,
    Validation,
    Conflict,
    Forbidden,
    Unauthorized,
    BadRequest,
    Internal,
}
```

Construct results via the static factories, never a public constructor:

```csharp
Result.Success(value)                 // Result<T>, success
Result.Failure<T>(resultError)         // Result<T>, failure
```

### The reservation rule (ADR-IDP001)

If a domain expert would describe an outcome as **"the system did its job and the answer was no"** — return `Result.Failure(...)`:

- Name already exists → `Result.Failure<T>(new ResultError(ResultErrorKind.Conflict, "user.duplicate_email", "Email already exists"))`
- Entity not found → `ResultErrorKind.NotFound`
- Validation error → `ResultErrorKind.Validation`
- Insufficient permission → `ResultErrorKind.Forbidden`

If a domain expert would describe an outcome as **"something went wrong"** — it is not a `Result.Failure`. Integration failures (DB unavailable, downstream timeout) are caught and translated/re-raised at the layer that owns the integration (see `csharp/persistence.md`); programmer errors are exceptions that propagate to the last-resort handler. **Never mix the two mechanisms for the same kind of outcome** — a method that returns `Result<T>` for "not found" must not also throw for "not found" on a different code path.

Controllers translate `Result<T>` to HTTP via `result.ToActionResult(this)` — see `csharp/presentation.md`. Failure codes (e.g. `user.duplicate_email`) are stable API contract elements; clients match on `Code`, never on `Message`.

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

Reserve exceptions for the two cases ADR-IDP001 assigns to them — **never** for an expected outcome a `Result<T>` should carry instead:

```csharp
// CORRECT — repository-internal signal, translated to Result.Failure by the calling service
public class NotFoundException : Exception
{
    public NotFoundException(string message) : base(message) { }
}

// CORRECT — genuine integration failure (DB unreachable, downstream timeout),
// caught and re-raised at the layer that owns the integration (see csharp/persistence.md)
public class IntegrationException : Exception
{
    public IntegrationException(string message, Exception? inner = null) : base(message, inner) { }
}
```

**Do not** define an exception for a business outcome that has a `ResultErrorKind` — e.g. no `DuplicateEmailException`. "Email already exists" is `Result.Failure<T>(new ResultError(ResultErrorKind.Conflict, "user.duplicate_email", "Email already exists"))` per the reservation rule, not a thrown exception.
