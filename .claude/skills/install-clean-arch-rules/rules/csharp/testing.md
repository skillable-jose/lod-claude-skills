# C# Testing Requirements

> This file extends common/testing.md with C# specific content.

---
paths: ["**/*.cs", "**/*.csx"]
---

## Minimum Test Coverage: 80%

Same requirement as common/testing.md. All C# projects must maintain 80%+ code coverage.

## Test Framework & Tools

- **Testing Framework**: MSTest
- **Mocking Framework**: Moq (with `MockBehavior.Strict`)
- **Assertion Library**: MSTest Assert + FluentAssertions
- **Test Organization**: One test class per service/class being tested

Install packages:
```powershell
dotnet add package Microsoft.NET.Test.Sdk
dotnet add package MSTest.TestAdapter
dotnet add package MSTest.TestFramework
dotnet add package Moq
dotnet add package FluentAssertions
```

## Test Class Structure

### Class Variable Declaration

Declare and initialize strict dependency mocks as **class variables inline**:

```csharp
[TestClass]
public class ClaimServiceTests
{
    // Dependency mocks - camelCase, no underscores, no abbreviations
    private Mock<IClaimRepository> claimRepositoryMock = new(MockBehavior.Strict);
    private Mock<IClaimValidator> claimValidatorMock = new(MockBehavior.Strict);
    private Mock<ILogger<ClaimService>> loggerMock = new(MockBehavior.Strict);
    private Mock<IDateTimeProvider> dateTimeProviderMock = new(MockBehavior.Strict);

    // Service under test mock - initialized in Setup
    private Mock<ClaimService> claimServiceMock = null!;

    // Time provider for testable time
    private FakeTimeProvider timeProvider = null!;
}
```

**Variable Naming Rules**:
- Use **camelCase** for private variables
- **No underscores** (e.g., use `claimRepositoryMock`, NOT `_claimRepositoryMock`)
- **No abbreviations** (e.g., use `claimRepositoryMock`, NOT `claimRepoMock`)
- Suffix all mocks with `Mock`

### Setup Method

Every test class MUST have a `[TestInitialize]` setup method that creates the mock of the service under test using **factory with constructor syntax**:

```csharp
[TestInitialize]
public void Setup()
{
    // Reset time provider for each test
    timeProvider = new FakeTimeProvider();

    // Create mock using factory syntax to allow mocking virtual methods
    claimServiceMock = new Mock<ClaimService>(
        () => new ClaimService(
            claimRepositoryMock.Object,
            claimValidatorMock.Object,
            loggerMock.Object,
            dateTimeProviderMock.Object
        ),
        MockBehavior.Strict);
}
```

**Why mock the service under test?**
- Tests go **one method deep** — the method under test runs its real logic, but any other method it calls on the same class is intercepted and controlled by the mock
- Interface dependencies (constructor-injected) are mocked as usual
- Internal virtual method calls are mocked via `Mock<SUT>.Setup()`, preventing execution of sibling method logic
- This is NOT testing the mock — the method under test executes real code via `CallBase` (virtual) or directly (non-virtual)

## Test Method Naming Convention

C# is the authoritative owner of this naming syntax. Test method names MUST follow one of these patterns:

1. **`<MethodName>_Should<Result>_When<Condition>`**
2. **`<MethodName>_Should<Result>_Given<Condition>`**

The three-part structure maps directly to the intent rule in [common/testing.md](../common/testing.md):
- `<MethodName>` — what is under test
- `Should<Result>` — expected outcome
- `When/Given<Condition>` — the triggering condition

```csharp
[TestMethod]
public async Task GetClaimAsync_ShouldReturnClaim_WhenClaimExists()

[TestMethod]
public async Task CreateClaimAsync_ShouldThrowValidationException_WhenClaimNumberIsEmpty()

[TestMethod]
public async Task ProcessClaimAsync_ShouldUpdateStatus_GivenValidStatusTransition()
```

**Method Signature Rules**:
- Use `async Task` for async methods being tested
- Use descriptive, specific condition descriptions — never vague words like `Works`, `Success`, `Valid`
- The full intent (unit + outcome + condition) must be readable from the method name alone without opening the test body

## Mock Usage Standards

### Always Use Strict Mocks

All mocks MUST use `MockBehavior.Strict`:

```csharp
// CORRECT
private Mock<IClaimRepository> claimRepositoryMock = new(MockBehavior.Strict);

// WRONG - never use Loose
private Mock<IClaimRepository> claimRepositoryMock = new(MockBehavior.Loose);
```

### Setup with Verifiable - MANDATORY

Every Setup call MUST be chained with `.Verifiable(Times.X)`:

```csharp
claimRepositoryMock
    .Setup(repo => repo.GetByIdAsync(claimId, cancellationToken))
    .ReturnsAsync(expectedClaim)
    .Verifiable(Times.Once());
```

**Formatting Rules**:
- Each chained method starts on a new line
- Setup, Returns, and Verifiable each on separate lines

### Parameter Matching

Maximize argument checking. Avoid `It.IsAny()` when possible:

```csharp
// AVOID: Using It.IsAny when specific values can be checked
claimRepositoryMock
    .Setup(repo => repo.GetByIdAsync(It.IsAny<int>(), It.IsAny<CancellationToken>()))
    .ReturnsAsync(expectedClaim);

// PREFER: Specific value checking
var claimId = 123;
claimRepositoryMock
    .Setup(repo => repo.GetByIdAsync(claimId, cancellationToken))
    .ReturnsAsync(expectedClaim)
    .Verifiable(Times.Once());

// BEST: Use It.Is<T> for complex matching
claimRepositoryMock
    .Setup(repo => repo.CreateAsync(
        It.Is<Claim>(c => c.ClaimNumber == "CLM-123" && c.Priority > 0),
        cancellationToken))
    .ReturnsAsync(savedClaim)
    .Verifiable(Times.Once());
```

## Test Implementation Structure

Every test MUST follow Arrange/Act/Assert with clearly marked sections:

```csharp
[TestMethod]
public async Task CreateClaimAsync_ShouldReturnClaimDto_WhenRequestIsValid()
{
    // Arrange
    var request = new CreateClaimRequest { ClaimNumber = "CLM-001" };
    var expectedClaim = new ClaimDto { Id = 1, ClaimNumber = "CLM-001" };

    claimValidatorMock
        .Setup(v => v.ValidateAsync(request, cancellationToken))
        .ReturnsAsync(ValidationResult.Success)
        .Verifiable(Times.Once());

    claimRepositoryMock
        .Setup(r => r.CreateAsync(
            It.Is<Claim>(c => c.ClaimNumber == request.ClaimNumber),
            cancellationToken))
        .ReturnsAsync(expectedClaim)
        .Verifiable(Times.Once());

    // Act
    var result = await claimServiceMock.Object.CreateClaimAsync(request, cancellationToken);

    // Assert
    Assert.IsNotNull(result);
    Assert.AreEqual(expectedClaim.ClaimNumber, result.ClaimNumber);

    claimServiceMock.VerifyAll();
    claimValidatorMock.VerifyAll();
    claimRepositoryMock.VerifyAll();
}
```

## Virtual Method Testing - CRITICAL

Virtual methods require special handling with Moq:

### Testing the virtual method itself - use CallBase()

```csharp
claimServiceMock
    .Setup(service => service.GetClaimAsync(claimId, cancellationToken))
    .CallBase()
    .Verifiable(Times.Once());

var result = await claimServiceMock.Object.GetClaimAsync(claimId, cancellationToken);
```

### Testing a non-virtual method that calls a virtual method - mock the virtual method

```csharp
// Mock the virtual method that will be called internally
claimServiceMock
    .Setup(service => service.GetClaimAsync(claimId, cancellationToken))
    .ReturnsAsync(claim)
    .Verifiable(Times.Once());

// NO CallBase() for non-virtual method under test
await claimServiceMock.Object.ProcessClaimAsync(claimId, cancellationToken);
```

**Rules Summary**:
- **Virtual method under test**: `.Setup().CallBase().Verifiable()`
- **Non-virtual method under test**: NO setup needed — real code executes directly
- **Virtual method called by method under test**: `.Setup().ReturnsAsync().Verifiable()` — intercepted by mock, real code does NOT execute

## Virtual Methods Prerequisite

All public and internal methods on service/repository classes MUST be `virtual`. See [csharp/coding-style.md](coding-style.md#virtual-methods-on-service-classes--critical) for the full rule, rationale, and examples.

When generating or reviewing tests, if a non-virtual or static method on the SUT is encountered:

1. **Flag as a warning** before writing any test code
2. **Recommend** adding the `virtual` keyword (or converting static to virtual instance method)
3. **Do not silently write tests** that allow non-virtual internal calls to execute

## Assert Section Standards

### VerifyAll() - MANDATORY

Every test MUST call `VerifyAll()` on:
1. The service mock itself
2. All dependency mocks that have Setup calls

```csharp
// Assert
Assert.IsNotNull(result);
Assert.AreEqual(expectedClaimNumber, result.ClaimNumber);

// MANDATORY: Verify all mocks
claimServiceMock.VerifyAll();
claimValidatorMock.VerifyAll();
claimRepositoryMock.VerifyAll();
loggerMock.VerifyAll();
```

### Exception Testing

Use `Assert.ThrowsExceptionAsync<T>`. Do NOT use `[ExpectedException]` attribute:

```csharp
[TestMethod]
public async Task CreateClaimAsync_ShouldThrowValidationException_WhenClaimNumberIsEmpty()
{
    // Arrange
    var invalidRequest = new CreateClaimRequest { ClaimNumber = "" };

    claimValidatorMock
        .Setup(v => v.ValidateAsync(invalidRequest, cancellationToken))
        .ThrowsAsync(new ValidationException("Claim number is required"))
        .Verifiable(Times.Once());

    // Act & Assert
    var exception = await Assert.ThrowsExceptionAsync<ValidationException>(
        () => claimServiceMock.Object.CreateClaimAsync(invalidRequest, cancellationToken));

    Assert.AreEqual("Claim number is required", exception.Message);

    claimServiceMock.VerifyAll();
    claimValidatorMock.VerifyAll();
}
```

## Time Provider

Use `FakeTimeProvider` for testable time:

```csharp
[TestClass]
public class ClaimServiceTests
{
    private FakeTimeProvider timeProvider = null!;
    private Mock<ClaimService> claimServiceMock = null!;

    [TestInitialize]
    public void Setup()
    {
        timeProvider = new FakeTimeProvider();
        timeProvider.SetUtcNow(new DateTime(2024, 1, 15, 10, 30, 0, DateTimeKind.Utc));

        claimServiceMock = new Mock<ClaimService>(
            () => new ClaimService(timeProvider),
            MockBehavior.Strict);
    }

    [TestMethod]
    public async Task CreateClaim_ShouldSetCreatedDate_WhenClaimIsValid()
    {
        // Arrange
        var expectedDate = timeProvider.GetUtcNow();

        // Act
        var result = await claimServiceMock.Object.CreateClaimAsync(request, cancellationToken);

        // Assert
        Assert.AreEqual(expectedDate, result.CreatedDate);
    }
}
```

## Parameterized Tests with DataRow

Use `[DataRow]` for testing multiple scenarios:

```csharp
[TestMethod]
[DataRow(0, false)]
[DataRow(-1, false)]
[DataRow(1, true)]
[DataRow(100, true)]
public void IsValidUserId_ShouldReturnExpectedResult_GivenVariousInputs(
    int userId,
    bool expected)
{
    // Act
    var result = userServiceMock.Object.IsValidUserId(userId);

    // Assert
    Assert.AreEqual(expected, result);
}
```

## EF Core Testing

### DbContext Mock Testing

```csharp
[TestClass]
public class ClaimDataMapperTests
{
    private Mock<CareMCContext> contextMock = new(MockBehavior.Strict);
    private Mock<DbSet<Claim>> claimDbSetMock = new(MockBehavior.Strict);
    private Mock<ClaimDataMapper> dataMapperMock = null!;

    [TestInitialize]
    public void Setup()
    {
        dataMapperMock = new Mock<ClaimDataMapper>(
            () => new ClaimDataMapper(contextMock.Object),
            MockBehavior.Strict);
    }

    [TestMethod]
    public async Task GetClaimByIdAsync_ShouldReturnClaim_WhenClaimExists()
    {
        // Arrange
        var claimId = 123;
        var expectedClaim = new Claim { Id = claimId };

        contextMock
            .Setup(ctx => ctx.Claims)
            .Returns(claimDbSetMock.Object)
            .Verifiable(Times.Once());

        claimDbSetMock
            .Setup(set => set.FindAsync(claimId))
            .ReturnsAsync(expectedClaim)
            .Verifiable(Times.Once());

        // Act
        var result = await dataMapperMock.Object.GetClaimByIdAsync(claimId, cancellationToken);

        // Assert
        Assert.IsNotNull(result);
        Assert.AreEqual(claimId, result.Id);
        dataMapperMock.VerifyAll();
        contextMock.VerifyAll();
        claimDbSetMock.VerifyAll();
    }
}
```

### Stored Procedure Testing

```csharp
[TestMethod]
public async Task GetClaimSummaryAsync_ShouldReturnSummary_WhenClaimExists()
{
    // Arrange
    var claimId = 123;
    var expectedResults = new List<ClaimSummaryDto>
    {
        new ClaimSummaryDto { ClaimId = claimId, Status = "Open" }
    };

    contextMock
        .Setup(ctx => ctx.Database.SqlQuery<ClaimSummaryDto>(
            $"EXEC GetClaimSummary {claimId}"))
        .Returns(expectedResults.AsAsyncEnumerable())
        .Verifiable(Times.Once());

    // Act
    var result = await dataMapperMock.Object.GetClaimSummaryAsync(claimId, cancellationToken);

    // Assert
    Assert.IsNotNull(result);
    Assert.AreEqual(claimId, result.ClaimId);
    dataMapperMock.VerifyAll();
    contextMock.VerifyAll();
}
```

## Integration Tests with WebApplicationFactory

Test entire HTTP pipeline including routing, model binding, validation, and filters:

```csharp
[TestClass]
public class UsersControllerIntegrationTests
{
    private WebApplicationFactory<Program> factory = null!;
    private HttpClient client = null!;

    [TestInitialize]
    public void Setup()
    {
        factory = new WebApplicationFactory<Program>()
            .WithWebHostBuilder(builder =>
            {
                builder.ConfigureServices(services =>
                {
                    var descriptor = services.SingleOrDefault(
                        d => d.ServiceType == typeof(DbContextOptions<ApplicationDbContext>));

                    if (descriptor != null)
                        services.Remove(descriptor);

                    services.AddDbContext<ApplicationDbContext>(options =>
                    {
                        options.UseInMemoryDatabase("TestDb");
                    });
                });
            });

        client = factory.CreateClient();
    }

    [TestCleanup]
    public void Cleanup()
    {
        client.Dispose();
        factory.Dispose();
    }

    [TestMethod]
    public async Task GetUsers_ShouldReturnSuccess_WhenUsersExist()
    {
        // Act
        var response = await client.GetAsync("/api/users");

        // Assert
        response.EnsureSuccessStatusCode();
        Assert.AreEqual(
            "application/json; charset=utf-8",
            response.Content.Headers.ContentType?.ToString());
    }
}
```

## FluentAssertions

More expressive assertions (used alongside MSTest Assert):

```csharp
using FluentAssertions;

[TestMethod]
public async Task GetUserByIdAsync_ShouldReturnUserWithCorrectProperties_WhenUserExists()
{
    // Arrange
    var userId = 1;
    var user = new User { Id = userId, Name = "John Doe", Email = "john@example.com" };
    userRepositoryMock
        .Setup(r => r.GetByIdAsync(userId, cancellationToken))
        .ReturnsAsync(user)
        .Verifiable(Times.Once());

    // Act
    var result = await userServiceMock.Object.GetUserByIdAsync(userId, cancellationToken);

    // Assert
    result.Should().NotBeNull();
    result!.Id.Should().Be(userId);
    result.Name.Should().Be("John Doe");
    result.Email.Should().Be("john@example.com");

    userServiceMock.VerifyAll();
    userRepositoryMock.VerifyAll();
}
```

## Test Project Organization

Structure tests to mirror source code:

```
Solution/
├── MyApp/
│   ├── Controllers/
│   │   └── UsersController.cs
│   ├── Services/
│   │   └── UserService.cs
│   └── Repositories/
│       └── UserRepository.cs
└── MyApp.Tests/
    ├── Controllers/
    │   └── UsersControllerTests.cs
    ├── Services/
    │   └── UserServiceTests.cs
    ├── Repositories/
    │   └── UserRepositoryTests.cs
    └── Integration/
        └── UsersControllerIntegrationTests.cs
```

## Running Tests and Coverage

```powershell
# Run all tests
dotnet test

# Run tests with coverage
dotnet test /p:CollectCoverage=true /p:CoverletOutputFormat=opencover

# With coverage threshold
dotnet test /p:CollectCoverage=true /p:Threshold=80

# Generate HTML coverage report (requires ReportGenerator)
dotnet tool install -g dotnet-reportgenerator-globaltool
dotnet test /p:CollectCoverage=true /p:CoverletOutputFormat=cobertura
reportgenerator -reports:"coverage.cobertura.xml" -targetdir:"coveragereport" -reporttypes:Html
```

Add to `.csproj`:
```xml
<ItemGroup>
  <PackageReference Include="coverlet.collector" Version="6.0.0" />
  <PackageReference Include="coverlet.msbuild" Version="6.0.0" />
</ItemGroup>
```

## C# Testing Checklist

Before submitting unit tests:

- [ ] Test class named `<ClassUnderTest>Tests`
- [ ] All mocks declared inline with `MockBehavior.Strict`
- [ ] Mock variables use camelCase, no underscores, no abbreviations, suffixed with `Mock`
- [ ] `[TestInitialize]` Setup method creates service mock with factory constructor syntax
- [ ] `FakeTimeProvider` used for testable time
- [ ] Test methods follow `MethodName_ShouldResult_WhenCondition` naming convention
- [ ] All public/internal methods on SUT are `virtual` (no non-virtual, no static on service classes)
- [ ] Non-virtual or static methods flagged as warnings if encountered
- [ ] Virtual methods tested with `.CallBase()` when under test
- [ ] All Setup calls chained with `.Verifiable(Times.X)`
- [ ] Every test calls `.VerifyAll()` on all mocks
- [ ] Exception tests use `Assert.ThrowsExceptionAsync<T>` (NOT `[ExpectedException]`)
- [ ] Parameter matching maximized (avoid `It.IsAny` when specific values can be checked)
- [ ] AAA pattern used (Arrange-Act-Assert with comments)
- [ ] `async Task` used for async test methods
- [ ] 80%+ code coverage achieved
- [ ] All tests pass (`dotnet test` succeeds)
- [ ] TDD workflow followed (tests written before implementation)

These testing standards are mandatory for all C# projects.
