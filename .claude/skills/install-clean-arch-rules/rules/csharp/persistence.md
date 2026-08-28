# C# Persistence Layer Rules

> Rules for Repository assemblies: repository implementations, EF Core entities, DbContext, migrations, specifications, and data access patterns.

---
paths: ["**/*.cs", "**/*.csx"]
---

## Repository Pattern

Repositories implement interfaces defined in Abstractions (see `csharp/domain.md`). They encapsulate all data access behind a consistent interface, operating on **domain models** externally while using ORM entities internally.

### Domain vs Entity Separation (CRITICAL)

ORM entity classes (with navigation properties, `[Table]` attributes, etc.) are defined in the Repository project as an implementation detail and MUST NOT leak outside it. Repository implementations map between domain models and ORM entities internally.

```csharp
// EF Core entity — defined in Repository project, NOT in Abstractions
// Contains ORM concerns (navigation properties, EF attributes)
public class UserEntity
{
    public int Id { get; set; }
    public string Email { get; set; } = string.Empty;
    public string PasswordHash { get; set; } = string.Empty;
    public string FirstName { get; set; } = string.Empty;
    public string LastName { get; set; } = string.Empty;
    public UserRole Role { get; set; }
    public bool IsActive { get; set; }
    public DateTime CreatedAtUtc { get; set; }
    public DateTime? UpdatedAtUtc { get; set; }
    public ICollection<TodoItemEntity> Todos { get; set; } = [];  // Navigation property — ORM only
}
```

### Thread Safety (CRITICAL)

Repositories derive from `RepositoryBase<TContext>`, which accepts `IDbContextFactory<TContext>` (not `TContext` directly) and exposes a `CreateContextAsync` method. Each repository method creates a short-lived `await using` DbContext via `CreateContextAsync`. This ensures each operation gets its own DbContext, making repositories safe to use from concurrent scopes, background services, and parallel calls. Never store a DbContext as a field.

```csharp
// Base class — centralizes IDbContextFactory injection and DbContext creation
public abstract class RepositoryBase<TContext>(
    IDbContextFactory<TContext> contextFactory) where TContext : DbContext
{
    protected virtual async Task<TContext> CreateContextAsync(CancellationToken cancellationToken)
    {
        return await contextFactory.CreateDbContextAsync(cancellationToken);
    }
}
```

### Repository Implementation

Read methods use `AsNoTracking()` and return new domain model instances. Write methods accept domain models, map to EF Core entities, persist via change tracking internally, and return new domain model instances reflecting the saved state.

```csharp
public class UserRepository(
    IDbContextFactory<ApplicationDbContext> contextFactory)
    : RepositoryBase<ApplicationDbContext>(contextFactory), IUserRepository
{
    // Single — throws NotFoundException when not found
    public async Task<User> UserSingleByIdAsync(int id, CancellationToken cancellationToken)
    {
        return await UserSingleOrDefaultByIdAsync(id, cancellationToken)
            ?? throw new NotFoundException($"User not found (UserId: {id}).");
    }

    public async Task<User> UserSingleByEmailAsync(string email, CancellationToken cancellationToken)
    {
        return await UserSingleOrDefaultByEmailAsync(email, cancellationToken)
            ?? throw new NotFoundException($"User not found (Email: {email}).");
    }

    // SingleOrDefault — returns null when not found
    public async Task<User?> UserSingleOrDefaultByIdAsync(int id, CancellationToken cancellationToken)
    {
        await using var dbContext = await CreateContextAsync(cancellationToken);

        var entity = await dbContext.Users
            .AsNoTracking()
            .FirstOrDefaultAsync(u => u.Id == id, cancellationToken);

        return entity is null ? null : MapToDomain(entity);
    }

    public async Task<User> UserAddAsync(User user, CancellationToken cancellationToken)
    {
        await using var dbContext = await CreateContextAsync(cancellationToken);

        var entity = MapToEntity(user);
        var entry = await dbContext.Users.AddAsync(entity, cancellationToken);
        await dbContext.SaveChangesAsync(cancellationToken);
        return MapToDomain(entry.Entity);
    }

    public async Task<User> UserUpdateAsync(User user, CancellationToken cancellationToken)
    {
        await using var dbContext = await CreateContextAsync(cancellationToken);

        var entity = MapToEntity(user);
        dbContext.Users.Update(entity);
        await dbContext.SaveChangesAsync(cancellationToken);
        return MapToDomain(entity);
    }

    // ... other methods follow the same pattern

    private static User MapToDomain(UserEntity entity)
    {
        return new User
        {
            Id = entity.Id,
            Email = entity.Email,
            PasswordHash = entity.PasswordHash,
            FirstName = entity.FirstName,
            LastName = entity.LastName,
            Role = entity.Role,
            IsActive = entity.IsActive,
            CreatedAtUtc = entity.CreatedAtUtc,
            UpdatedAtUtc = entity.UpdatedAtUtc
        };
    }

    private static UserEntity MapToEntity(User user)
    {
        return new UserEntity
        {
            Id = user.Id,
            Email = user.Email,
            PasswordHash = user.PasswordHash,
            FirstName = user.FirstName,
            LastName = user.LastName,
            Role = user.Role,
            IsActive = user.IsActive,
            CreatedAtUtc = user.CreatedAtUtc,
            UpdatedAtUtc = user.UpdatedAtUtc
        };
    }
}
```

### DI Registration

Wire inside the assembly's `Add{Feature}` extension method. Use `AddDbContextFactory` (NOT `AddDbContext`) — repositories create their own DbContext per operation.

```csharp
// TodoApp.Repository/Infrastructure/PersistenceServiceCollectionExtensions.cs
using Microsoft.EntityFrameworkCore;

namespace Microsoft.Extensions.DependencyInjection;

public static class PersistenceServiceCollectionExtensions
{
    public static IServiceCollection AddPersistence(
        this IServiceCollection services,
        IConfiguration configuration)
    {
        services.AddDbContextFactory<ApplicationDbContext>(options =>
            options.UseSqlServer(
                configuration.GetConnectionString("DefaultConnection")));

        services.AddScoped<IUserRepository, UserRepository>();
        services.AddScoped<IOrderRepository, OrderRepository>();

        return services;
    }
}
```

## DbContext Configuration

```csharp
public class ApplicationDbContext(DbContextOptions<ApplicationDbContext> options)
    : DbContext(options)
{
    public DbSet<User> Users => Set<User>();
    public DbSet<Order> Orders => Set<Order>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        base.OnModelCreating(modelBuilder);

        modelBuilder.Entity<User>(entity =>
        {
            entity.HasKey(e => e.Id);
            entity.Property(e => e.Email).IsRequired().HasMaxLength(255);
            entity.HasIndex(e => e.Email).IsUnique();

            entity.HasMany(e => e.Orders)
                .WithOne(e => e.User)
                .HasForeignKey(e => e.UserId)
                .OnDelete(DeleteBehavior.Cascade);
        });

        modelBuilder.ApplyConfigurationsFromAssembly(typeof(ApplicationDbContext).Assembly);
    }
}

// Registration — use AddDbContextFactory for thread-safe repository access
builder.Services.AddDbContextFactory<ApplicationDbContext>(options =>
    options.UseSqlServer(
        builder.Configuration.GetConnectionString("DefaultConnection"),
        sqlOptions =>
        {
            sqlOptions.EnableRetryOnFailure(
                maxRetryCount: 3,
                maxRetryDelay: TimeSpan.FromSeconds(5),
                errorNumbersToAdd: null);
            sqlOptions.CommandTimeout(30);
        }));
```

## Migration Workflow

```powershell
# Create migration
dotnet ef migrations add AddUserEmail

# Update database
dotnet ef database update

# Rollback to specific migration
dotnet ef database update PreviousMigration

# Remove last migration (if not applied)
dotnet ef migrations remove

# Generate SQL script
dotnet ef migrations script
```

## Query Performance Patterns

All examples create a short-lived DbContext from `IDbContextFactory`:

```csharp
// CORRECT: AsNoTracking for read-only queries
public async Task<IReadOnlyList<UserDto>> GetUsersAsync(CancellationToken cancellationToken)
{
    await using var dbContext = await _dbContextFactory.CreateDbContextAsync(cancellationToken);

    return await dbContext.Users
        .AsNoTracking()
        .Select(u => new UserDto(u.Id, u.Name, u.Email))
        .ToListAsync(cancellationToken);
}

// CORRECT: Eager loading to avoid N+1 queries
public async Task<User?> GetUserWithOrdersAsync(int userId, CancellationToken cancellationToken)
{
    await using var dbContext = await _dbContextFactory.CreateDbContextAsync(cancellationToken);

    return await dbContext.Users
        .Include(u => u.Orders)
            .ThenInclude(o => o.Items)
        .FirstOrDefaultAsync(u => u.Id == userId, cancellationToken);
}

// CORRECT: Projection to select only needed columns
public async Task<IReadOnlyList<UserSummaryDto>> GetUserSummariesAsync(CancellationToken cancellationToken)
{
    await using var dbContext = await _dbContextFactory.CreateDbContextAsync(cancellationToken);

    return await dbContext.Users
        .Select(u => new UserSummaryDto
        {
            Id = u.Id,
            Name = u.Name,
            OrderCount = u.Orders.Count
        })
        .ToListAsync(cancellationToken);
}

// WRONG: Loading all data then filtering in memory
public async Task<IReadOnlyList<User>> GetActiveUsersWrong()
{
    await using var dbContext = await _dbContextFactory.CreateDbContextAsync();
    var allUsers = await dbContext.Users.ToListAsync();  // Loads all users
    return allUsers.Where(u => u.IsActive).ToList();  // Filters in memory
}
```

## Transaction Handling

When multiple repository operations must be atomic, wrap them in a `TransactionScope`. Each repository method creates its own DbContext, but `TransactionScope` enlists them in a single ambient transaction:

```csharp
public async Task<Order> CreateOrderWithInventoryUpdateAsync(
    CreateOrderRequest request,
    CancellationToken cancellationToken)
{
    using var transactionScope = new TransactionScope(TransactionScopeAsyncFlowOption.Enabled);

    var order = await _orderRepository.OrderAddAsync(
        new Order { UserId = request.UserId, Total = request.Total },
        cancellationToken);

    await _productRepository.ProductDecrementStockAsync(
        request.ProductId,
        request.Quantity,
        cancellationToken);

    transactionScope.Complete();
    return order;
}
```

## Specification Pattern

Encapsulate complex query logic in reusable specifications (interface defined in `csharp/domain.md`):

```csharp
public class BaseSpecification<T>(Expression<Func<T, bool>> criteria) : ISpecification<T>
{
    public Expression<Func<T, bool>> Criteria { get; } = criteria;
    public List<Expression<Func<T, object>>> Includes { get; } = new();
    public Expression<Func<T, object>>? OrderBy { get; private set; }

    protected void AddInclude(Expression<Func<T, object>> includeExpression)
    {
        Includes.Add(includeExpression);
    }

    protected void ApplyOrderBy(Expression<Func<T, object>> orderByExpression)
    {
        OrderBy = orderByExpression;
    }
}

// Example specification
// Subclasses that need constructor body logic keep traditional constructors
// since primary constructors have no body.
public class ActiveUsersSpecification : BaseSpecification<User>
{
    public ActiveUsersSpecification() : base(u => u.IsActive)
    {
        AddInclude(u => u.Orders);
        ApplyOrderBy(u => u.LastName);
    }
}

// Repository method consuming specifications
public async Task<IReadOnlyList<T>> GetAsync(ISpecification<T> spec)
{
    var query = _dbSet.AsQueryable();

    query = query.Where(spec.Criteria);

    query = spec.Includes.Aggregate(query, (current, include) =>
        current.Include(include));

    if (spec.OrderBy is not null)
        query = query.OrderBy(spec.OrderBy);

    return await query.ToListAsync();
}
```
