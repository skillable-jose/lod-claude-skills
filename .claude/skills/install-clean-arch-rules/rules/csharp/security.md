# C# Security Guidelines

> This file extends common/security.md with C# specific content.

---
paths: ["**/*.cs", "**/*.csx"]
---

## Secret Management

NEVER hardcode secrets. ALWAYS use `IConfiguration` with appropriate providers:

```csharp
// CORRECT: Using IConfiguration with primary constructor
public class EmailService(IConfiguration configuration)
{
    public async Task SendEmailAsync(string to, string subject, string body)
    {
        var apiKey = configuration["SendGrid:ApiKey"]
            ?? throw new InvalidOperationException("SendGrid API key not configured");

        // Use apiKey...
    }
}

// WRONG: Hardcoded secret
public class EmailService
{
    private const string API_KEY = "SG.abc123...";  // ❌ NEVER hardcode secrets
}
```

**Local Development**: Use User Secrets

```powershell
# Initialize user secrets
dotnet user-secrets init

# Add secret
dotnet user-secrets set "SendGrid:ApiKey" "your-secret-key"
```

```csharp
// Program.cs (automatically loaded in Development environment)
if (builder.Environment.IsDevelopment())
{
    builder.Configuration.AddUserSecrets<Program>();
}
```

**Production**: Use environment variables or Azure Key Vault

```csharp
// Environment variables
var connectionString = Environment.GetEnvironmentVariable("DATABASE_CONNECTION_STRING");

// Azure Key Vault (recommended for production)
builder.Configuration.AddAzureKeyVault(
    new Uri($"https://{keyVaultName}.vault.azure.net/"),
    new DefaultAzureCredential());
```

## SQL Injection Prevention

ALWAYS use parameterized queries. NEVER use string interpolation or concatenation in SQL:

```csharp
// CORRECT: EF Core (parameterized by default)
var users = await _dbContext.Users
    .Where(u => u.Email == email)
    .ToListAsync();

// CORRECT: Raw SQL with parameters
var users = await _dbContext.Users
    .FromSqlRaw("SELECT * FROM Users WHERE Email = {0}", email)
    .ToListAsync();

// CORRECT: Stored procedure with parameters
var users = await _dbContext.Users
    .FromSqlRaw("EXEC GetUserByEmail @Email = {0}", email)
    .ToListAsync();

// CORRECT: ADO.NET with parameters
using var command = new SqlCommand("SELECT * FROM Users WHERE Email = @Email", connection);
command.Parameters.AddWithValue("@Email", email);
var reader = await command.ExecuteReaderAsync();

// WRONG: String interpolation in raw SQL
var users = await _dbContext.Users
    .FromSqlRaw($"SELECT * FROM Users WHERE Email = '{email}'")  // ❌ SQL injection risk
    .ToListAsync();

// WRONG: String concatenation
var sql = "SELECT * FROM Users WHERE Email = '" + email + "'";  // ❌ SQL injection risk
var users = await _dbContext.Users.FromSqlRaw(sql).ToListAsync();
```

## Authentication & Authorization

Use ASP.NET Core's built-in authentication and authorization:

```csharp
// Program.cs - Configure authentication
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidateAudience = true,
            ValidateLifetime = true,
            ValidateIssuerSigningKey = true,
            ValidIssuer = builder.Configuration["Jwt:Issuer"],
            ValidAudience = builder.Configuration["Jwt:Audience"],
            IssuerSigningKey = new SymmetricSecurityKey(
                Encoding.UTF8.GetBytes(builder.Configuration["Jwt:Key"]!))
        };
    });

// Configure authorization policies
builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("RequireAdminRole", policy => 
        policy.RequireRole("Admin"));
    
    options.AddPolicy("RequireEmailVerified", policy =>
        policy.RequireClaim("email_verified", "true"));
});

var app = builder.Build();

app.UseAuthentication();
app.UseAuthorization();

// Controller with authorization
[ApiController]
[Route("api/[controller]")]
[Authorize]  // Require authentication for all actions
public class UsersController : ControllerBase
{
    [HttpGet]
    [AllowAnonymous]  // Override: allow anonymous access
    public async Task<ActionResult<IEnumerable<UserDto>>> GetUsers()
    {
        // Public endpoint
    }

    [HttpPost]
    [Authorize(Roles = "Admin")]  // Require Admin role
    public async Task<ActionResult<UserDto>> CreateUser(CreateUserRequest request)
    {
        // Admin-only endpoint
    }

    [HttpDelete("{id}")]
    [Authorize(Policy = "RequireAdminRole")]  // Require policy
    public async Task<ActionResult> DeleteUser(int id)
    {
        // Admin-only endpoint
    }
}
```

**Policy-based authorization** (recommended for complex rules):

```csharp
// Custom requirement
public class MinimumAgeRequirement(int minimumAge) : IAuthorizationRequirement
{
    public int MinimumAge { get; } = minimumAge;
}

// Handler
public class MinimumAgeHandler : AuthorizationHandler<MinimumAgeRequirement>
{
    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext context,
        MinimumAgeRequirement requirement)
    {
        var birthDateClaim = context.User.FindFirst(c => c.Type == ClaimTypes.DateOfBirth);
        
        if (birthDateClaim is null)
            return Task.CompletedTask;
        
        var birthDate = DateTime.Parse(birthDateClaim.Value);
        var age = DateTime.Today.Year - birthDate.Year;
        
        if (age >= requirement.MinimumAge)
            context.Succeed(requirement);
        
        return Task.CompletedTask;
    }
}

// Registration
builder.Services.AddSingleton<IAuthorizationHandler, MinimumAgeHandler>();
builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("AtLeast18", policy =>
        policy.Requirements.Add(new MinimumAgeRequirement(18)));
});

// Usage
[Authorize(Policy = "AtLeast18")]
public async Task<ActionResult> AdultOnlyAction() { }
```

## CORS Configuration

Configure CORS properly to prevent unauthorized cross-origin requests:

```csharp
// Program.cs
builder.Services.AddCors(options =>
{
    options.AddPolicy("AllowSpecificOrigin", policy =>
    {
        policy.WithOrigins(
                "https://example.com",
                "https://app.example.com")
            .AllowAnyMethod()
            .AllowAnyHeader()
            .AllowCredentials();
    });
    
    // Development only - NEVER use in production
    if (builder.Environment.IsDevelopment())
    {
        options.AddPolicy("DevelopmentCors", policy =>
        {
            policy.AllowAnyOrigin()
                .AllowAnyMethod()
                .AllowAnyHeader();
        });
    }
});

var app = builder.Build();

app.UseCors(builder.Environment.IsDevelopment() 
    ? "DevelopmentCors" 
    : "AllowSpecificOrigin");

// WRONG: Allow all origins in production
builder.Services.AddCors(options =>
{
    options.AddDefaultPolicy(policy =>
    {
        policy.AllowAnyOrigin()  // ❌ Security risk in production
            .AllowAnyMethod()
            .AllowAnyHeader();
    });
});
```

## Anti-Forgery Token Validation

Enable anti-forgery tokens for state-changing operations:

```csharp
// Program.cs
builder.Services.AddAntiforgery(options =>
{
    options.HeaderName = "X-CSRF-TOKEN";
    options.Cookie.SecurePolicy = CookieSecurePolicy.Always;
    options.Cookie.SameSite = SameSiteMode.Strict;
});

// Controller action
[HttpPost]
[ValidateAntiForgeryToken]
public async Task<ActionResult<UserDto>> CreateUser(CreateUserRequest request)
{
    // Action protected by anti-forgery token
}

// For APIs, use custom middleware
public class AntiForgeryMiddleware(RequestDelegate next, IAntiforgery antiforgery)
{
    public async Task InvokeAsync(HttpContext context)
    {
        if (HttpMethods.IsPost(context.Request.Method) ||
            HttpMethods.IsPut(context.Request.Method) ||
            HttpMethods.IsDelete(context.Request.Method))
        {
            await antiforgery.ValidateRequestAsync(context);
        }

        await next(context);
    }
}
```

## Input Validation

ALWAYS validate all inputs at API boundaries using **FluentValidation**. NEVER use Data Annotation attributes on records — they clutter positional syntax and violate separation of concerns (see [csharp/coding-style.md](../csharp/coding-style.md#validation-strategy-critical)).

```csharp
// Clean positional record — no validation attributes
public record CreateUserRequest(string Email, string Name, int Age);

// FluentValidation validator — testable, composable, separated from the model
public class CreateUserRequestValidator : AbstractValidator<CreateUserRequest>
{
    public CreateUserRequestValidator()
    {
        RuleFor(x => x.Email)
            .NotEmpty().WithMessage("Email is required")
            .EmailAddress().WithMessage("Invalid email format");

        RuleFor(x => x.Name)
            .NotEmpty()
            .Length(2, 100).WithMessage("Name must be 2-100 characters");

        RuleFor(x => x.Age)
            .InclusiveBetween(18, 120).WithMessage("Age must be between 18 and 120");
    }
}

// Async/dependent validation (e.g., uniqueness checks)
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

// Registration — wire inside Add{Feature} extension method
builder.Services.AddValidatorsFromAssemblyContaining<CreateUserRequestValidator>();
builder.Services.AddFluentValidationAutoValidation();
```

## Secure Password Hashing

NEVER store passwords in plain text. Use ASP.NET Core Identity or a proper hashing library:

```csharp
// Using ASP.NET Core Identity (recommended)
public class UserService(UserManager<IdentityUser> userManager)
{
    public async Task<IdentityResult> CreateUserAsync(string email, string password)
    {
        var user = new IdentityUser { UserName = email, Email = email };
        return await userManager.CreateAsync(user, password);  // Automatically hashed
    }

    public async Task<bool> ValidatePasswordAsync(string email, string password)
    {
        var user = await userManager.FindByEmailAsync(email);
        if (user is null) return false;

        return await userManager.CheckPasswordAsync(user, password);
    }
}

// Using BCrypt.Net (alternative)
public class PasswordService
{
    public string HashPassword(string password)
    {
        return BCrypt.Net.BCrypt.HashPassword(password, workFactor: 12);
    }

    public bool VerifyPassword(string password, string hash)
    {
        return BCrypt.Net.BCrypt.Verify(password, hash);
    }
}

// WRONG: Plain text or weak hashing
public class UserService
{
    public async Task CreateUserAsync(string email, string password)
    {
        var user = new User
        {
            Email = email,
            Password = password  // ❌ NEVER store plain text
        };
        
        // Also wrong: MD5, SHA1, SHA256 without salt
        user.Password = ComputeMD5Hash(password);  // ❌ Not secure for passwords
    }
}
```

## Sensitive Data in Logs

NEVER log sensitive information:

```csharp
// CORRECT: Log only safe data
logger.LogInformation(
    "User login attempt (Email: {Email}).",
    email);  // Email may be acceptable depending on policy

logger.LogInformation(
    "Processing payment (UserId: {UserId}).",
    userId);  // Use IDs, not PII

// WRONG: Logging sensitive data
logger.LogInformation(
    "User login with password {Password}",
    password);  // ❌ NEVER log passwords

logger.LogInformation(
    "Processing credit card {CardNumber}",
    cardNumber);  // ❌ NEVER log card numbers

// Redact sensitive data in exceptions
try
{
    await ProcessPayment(cardNumber);
}
catch (Exception ex)
{
    logger.LogError(ex, "Payment processing failed (UserId: {UserId}).", userId);
    // Don't include card number in log
}
```

## Security Checklist for C#

Before committing code:
- [ ] No hardcoded secrets (use `IConfiguration` + User Secrets/Key Vault)
- [ ] All user inputs validated (Data Annotations or FluentValidation)
- [ ] SQL injection prevented (EF Core or parameterized queries)
- [ ] XSS prevention (use Razor encoding, never `@Html.Raw` with user input)
- [ ] Authentication/authorization implemented (`[Authorize]` attributes)
- [ ] CORS configured correctly (specific origins in production)
- [ ] Anti-forgery tokens enabled for state-changing operations
- [ ] Passwords hashed (Identity or BCrypt, not plain text/MD5/SHA)
- [ ] Sensitive data not logged (no passwords, tokens, card numbers)
- [ ] HTTPS enforced in production (`app.UseHttpsRedirection()`)

These security measures are mandatory for all ASP.NET Core applications.
