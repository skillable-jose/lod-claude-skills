# C# Testing Requirements

> This file extends common/testing.md with C# specific content.

---
paths: ["**/*.cs", "**/*.csx"]
---

## Minimum Test Coverage: 80%

Same requirement as common/testing.md. All C# projects must maintain 80%+ code coverage.

## Test Framework & Tools

- **Testing Framework**: xUnit
- **Mocking Framework**: Moq (with `MockBehavior.Strict`)
- **Assertion Library**: xUnit `Assert`
- **Test Organization**: One test class per service/class being tested

> **Recorded deviation from the org's generic default.** `ai-enablement`'s generic C# guidance (`csharp-coding-standards.md` / `review-dotnet-best-practices`) names **MSTest + FluentAssertions** with no AAA comments, copying the nearby file's style. This file deliberately deviates for repos in this family: the actual, already-shipping convention (see the real `.csproj` package references and the repo's own `unit-tests.md`/`CLAUDE.md`) is **xUnit + Moq(Strict)** with explicit AAA comments and `VerifyAll()`. Per the host-repo-wins adjudication rule (see `claude-md-brownfield.md`'s convention table), an established, working test suite is a legitimate reason to keep the host's framework rather than retrofit MSTest — this is not an oversight. If you are instead standing up a **new** C# project with no existing test convention, the org's generic MSTest+FluentAssertions default is the one to reach for; only use this file's xUnit+Moq bundle when the target repo already has it.

Install packages:
```powershell
dotnet add package Microsoft.NET.Test.Sdk
dotnet add package xunit
dotnet add package xunit.runner.visualstudio
dotnet add package Moq
dotnet add package coverlet.collector
```

## Test Class Naming

C# is the authoritative owner of this naming syntax (see the class-naming note in [common/testing.md](../common/testing.md#test-class-naming)). Test class names MUST follow the pattern **`<ClassUnderTest>Test`** — singular, no trailing `s`:

```
UserService          -> UserServiceTest
OrderValidator       -> OrderValidatorTest
LabProfileDataMapper -> LabProfileDataMapperTest
```

## Test Class Structure

### Class Variable Declaration

Declare and initialize strict dependency mocks as **class variables inline**:

```csharp
public class LabProfileServiceTest
{
    // Dependency mocks - underscore-prefixed camelCase, suffixed with Mock
    private readonly Mock<ILabProfileRepository> _labProfileRepositoryMock = new(MockBehavior.Strict);
    private readonly Mock<ILabProfileValidator> _labProfileValidatorMock = new(MockBehavior.Strict);
    private readonly Mock<ILogger<LabProfileService>> _loggerMock = new(MockBehavior.Strict);
    private readonly Mock<IDateTimeProvider> _dateTimeProviderMock = new(MockBehavior.Strict);

    // Service under test mock - initialized in the constructor
    private readonly Mock<LabProfileService> _labProfileServiceMock;

    // Time provider for testable time
    private readonly FakeTimeProvider _timeProvider;
}
```

**Variable Naming Rules**:
- Use **camelCase prefixed with an underscore** (e.g., `_labProfileRepositoryMock`, NOT `labProfileRepositoryMock`)
- **No abbreviations** (e.g., use `_labProfileRepositoryMock`, NOT `_labProfileRepoMock`)
- Suffix all mocks with `Mock`

### Constructor Setup

xUnit re-instantiates the test class for every test, so setup lives in the **constructor** — there is no `[TestInitialize]`/`[SetUp]` attribute. Create the mock of the service under test using **factory with constructor syntax**:

```csharp
public LabProfileServiceTest()
{
    // Reset time provider per test instance
    _timeProvider = new FakeTimeProvider();

    // Create mock using factory syntax to allow mocking virtual methods
    _labProfileServiceMock = new Mock<LabProfileService>(
        () => new LabProfileService(
            _labProfileRepositoryMock.Object,
            _labProfileValidatorMock.Object,
            _loggerMock.Object,
            _dateTimeProviderMock.Object
        ),
        MockBehavior.Strict);
}
```

If the test class needs teardown, implement `IDisposable` and dispose in `Dispose()` — there is no `[TestCleanup]` attribute in xUnit.

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
[Fact]
public async Task GetLabProfileAsync_ShouldReturnLabProfile_WhenLabProfileExists()

[Fact]
public async Task CreateLabProfileAsync_ShouldThrowValidationException_WhenNameIsEmpty()

[Fact]
public async Task ProcessLabProfileAsync_ShouldUpdateStatus_GivenValidStatusTransition()
```

**Method Signature Rules**:
- Use `async Task` for async methods being tested — never `async void`
- Use descriptive, specific condition descriptions — never vague words like `Works`, `Success`, `Valid`
- The full intent (unit + outcome + condition) must be readable from the method name alone without opening the test body

## Mock Usage Standards

### Always Use Strict Mocks

All mocks MUST use `MockBehavior.Strict`:

```csharp
// CORRECT
private readonly Mock<ILabProfileRepository> _labProfileRepositoryMock = new(MockBehavior.Strict);

// WRONG - never use Loose
private readonly Mock<ILabProfileRepository> _labProfileRepositoryMock = new(MockBehavior.Loose);
```

### Setup with Verifiable - MANDATORY

Every Setup call MUST be chained with `.Verifiable(Times.X)`. This replaces ad-hoc `Verify()` calls scattered through the Assert section — prefer `VerifyAll()` instead.

```csharp
_labProfileRepositoryMock
    .Setup(repo => repo.GetByIdAsync(labProfileId, cancellationToken))
    .ReturnsAsync(expectedLabProfile)
    .Verifiable(Times.Once);
```

**Formatting Rules**:
- Each chained method starts on a new line
- `Setup`, `Returns`/`ReturnsAsync`, and `Verifiable` each on separate lines

### Parameter Matching

Maximize argument checking. Avoid `It.IsAny()` when possible. If method parameters are propagated to dependencies, declare local variables for them so they can be verified:

```csharp
// AVOID: Using It.IsAny when specific values can be checked
_labProfileRepositoryMock
    .Setup(repo => repo.GetByIdAsync(It.IsAny<int>(), It.IsAny<CancellationToken>()))
    .ReturnsAsync(expectedLabProfile);

// PREFER: Specific value checking
var labProfileId = 123;
_labProfileRepositoryMock
    .Setup(repo => repo.GetByIdAsync(labProfileId, cancellationToken))
    .ReturnsAsync(expectedLabProfile)
    .Verifiable(Times.Once);

// BEST: Use It.Is<T> for complex matching
_labProfileRepositoryMock
    .Setup(repo => repo.CreateAsync(
        It.Is<LabProfile>(p => p.Name == "Advanced-Kubernetes-303" && p.MaxSeats > 0),
        cancellationToken))
    .ReturnsAsync(savedLabProfile)
    .Verifiable(Times.Once);
```

## Test Implementation Structure

Every test MUST follow Arrange/Act/Assert with clearly marked sections:

```csharp
[Fact]
public async Task CreateLabProfileAsync_ShouldReturnLabProfileDto_WhenRequestIsValid()
{
    // Arrange
    var request = new CreateLabProfileRequest { Name = "Intro-to-Networking-101" };
    var expectedLabProfile = new LabProfileDto { Id = 1, Name = "Intro-to-Networking-101" };

    _labProfileValidatorMock
        .Setup(v => v.ValidateAsync(request, cancellationToken))
        .ReturnsAsync(ValidationResult.Success)
        .Verifiable(Times.Once);

    _labProfileRepositoryMock
        .Setup(r => r.CreateAsync(
            It.Is<LabProfile>(p => p.Name == request.Name),
            cancellationToken))
        .ReturnsAsync(expectedLabProfile)
        .Verifiable(Times.Once);

    // Act
    var result = await _labProfileServiceMock.Object.CreateLabProfileAsync(request, cancellationToken);

    // Assert
    Assert.NotNull(result);
    Assert.Equal(expectedLabProfile.Name, result.Name);

    _labProfileServiceMock.VerifyAll();
    _labProfileValidatorMock.VerifyAll();
    _labProfileRepositoryMock.VerifyAll();
}
```

## Virtual Method Testing - CRITICAL

Virtual methods require special handling with Moq:

### Testing the virtual method itself - use CallBase()

```csharp
_labProfileServiceMock
    .Setup(service => service.GetLabProfileAsync(labProfileId, cancellationToken))
    .CallBase()
    .Verifiable(Times.Once);

var result = await _labProfileServiceMock.Object.GetLabProfileAsync(labProfileId, cancellationToken);
```

### Testing a non-virtual method that calls a virtual method - mock the virtual method

```csharp
// Mock the virtual method that will be called internally
_labProfileServiceMock
    .Setup(service => service.GetLabProfileAsync(labProfileId, cancellationToken))
    .ReturnsAsync(labProfile)
    .Verifiable(Times.Once);

// NO CallBase() for non-virtual method under test
await _labProfileServiceMock.Object.ProcessLabProfileAsync(labProfileId, cancellationToken);
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
Assert.NotNull(result);
Assert.Equal(expectedName, result.Name);

// MANDATORY: Verify all mocks
_labProfileServiceMock.VerifyAll();
_labProfileValidatorMock.VerifyAll();
_labProfileRepositoryMock.VerifyAll();
_loggerMock.VerifyAll();
```

### Exception Testing

Use `Assert.ThrowsAsync<T>` (or `Assert.Throws<T>` for synchronous methods). Do NOT use `[ExpectedException]`-style attributes — xUnit doesn't have one, but the same reasoning applies to any declarative exception attribute encountered in ported code:

```csharp
[Fact]
public async Task CreateLabProfileAsync_ShouldThrowValidationException_WhenNameIsEmpty()
{
    // Arrange
    var invalidRequest = new CreateLabProfileRequest { Name = "" };

    _labProfileValidatorMock
        .Setup(v => v.ValidateAsync(invalidRequest, cancellationToken))
        .ThrowsAsync(new ValidationException("Lab profile name is required"))
        .Verifiable(Times.Once);

    // Act & Assert
    var exception = await Assert.ThrowsAsync<ValidationException>(
        () => _labProfileServiceMock.Object.CreateLabProfileAsync(invalidRequest, cancellationToken));

    Assert.Equal("Lab profile name is required", exception.Message);

    _labProfileServiceMock.VerifyAll();
    _labProfileValidatorMock.VerifyAll();
}
```

## Time Provider

Use `FakeTimeProvider` (from `Microsoft.Extensions.TimeProvider.Testing`) for testable time:

```csharp
public class LabProfileServiceTest
{
    private readonly FakeTimeProvider _timeProvider;
    private readonly Mock<LabProfileService> _labProfileServiceMock;

    public LabProfileServiceTest()
    {
        _timeProvider = new FakeTimeProvider();
        _timeProvider.SetUtcNow(new DateTime(2024, 1, 15, 10, 30, 0, DateTimeKind.Utc));

        _labProfileServiceMock = new Mock<LabProfileService>(
            () => new LabProfileService(_timeProvider),
            MockBehavior.Strict);
    }

    [Fact]
    public async Task CreateLabProfile_ShouldSetCreatedDate_WhenLabProfileIsValid()
    {
        // Arrange
        var expectedDate = _timeProvider.GetUtcNow();

        // Act
        var result = await _labProfileServiceMock.Object.CreateLabProfileAsync(request, cancellationToken);

        // Assert
        Assert.Equal(expectedDate, result.CreatedDate);
    }
}
```

## Parameterized Tests with InlineData

Use `[Theory]` + `[InlineData]` for testing multiple scenarios:

```csharp
[Theory]
[InlineData(0, false)]
[InlineData(-1, false)]
[InlineData(1, true)]
[InlineData(100, true)]
public void IsValidUserId_ShouldReturnExpectedResult_GivenVariousInputs(
    int userId,
    bool expected)
{
    // Act
    var result = _userServiceMock.Object.IsValidUserId(userId);

    // Assert
    Assert.Equal(expected, result);
}
```

## EF Core Testing

### DbContext Mock Testing

```csharp
public class LabProfileDataMapperTest
{
    private readonly Mock<LabOnDemandContext> _contextMock = new(MockBehavior.Strict);
    private readonly Mock<DbSet<LabProfile>> _labProfileDbSetMock = new(MockBehavior.Strict);
    private readonly Mock<LabProfileDataMapper> _dataMapperMock;

    public LabProfileDataMapperTest()
    {
        _dataMapperMock = new Mock<LabProfileDataMapper>(
            () => new LabProfileDataMapper(_contextMock.Object),
            MockBehavior.Strict);
    }

    [Fact]
    public async Task GetLabProfileByIdAsync_ShouldReturnLabProfile_WhenLabProfileExists()
    {
        // Arrange
        var labProfileId = 123;
        var expectedLabProfile = new LabProfile { Id = labProfileId };

        _contextMock
            .Setup(ctx => ctx.LabProfiles)
            .Returns(_labProfileDbSetMock.Object)
            .Verifiable(Times.Once);

        _labProfileDbSetMock
            .Setup(set => set.FindAsync(labProfileId))
            .ReturnsAsync(expectedLabProfile)
            .Verifiable(Times.Once);

        // Act
        var result = await _dataMapperMock.Object.GetLabProfileByIdAsync(labProfileId, cancellationToken);

        // Assert
        Assert.NotNull(result);
        Assert.Equal(labProfileId, result.Id);
        _dataMapperMock.VerifyAll();
        _contextMock.VerifyAll();
        _labProfileDbSetMock.VerifyAll();
    }
}
```

## Integration Tests with WebApplicationFactory

Test entire HTTP pipeline including routing, model binding, validation, and filters. In xUnit, share the factory across tests with `IClassFixture<T>` rather than per-test `[TestInitialize]`/`[TestCleanup]`:

```csharp
public class UsersControllerIntegrationTest : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly HttpClient _client;

    public UsersControllerIntegrationTest(WebApplicationFactory<Program> factory)
    {
        var configuredFactory = factory.WithWebHostBuilder(builder =>
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

        _client = configuredFactory.CreateClient();
    }

    [Fact]
    public async Task GetUsers_ShouldReturnSuccess_WhenUsersExist()
    {
        // Act
        var response = await _client.GetAsync("/api/users");

        // Assert
        response.EnsureSuccessStatusCode();
        Assert.Equal(
            "application/json; charset=utf-8",
            response.Content.Headers.ContentType?.ToString());
    }
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
    │   └── UsersControllerTest.cs
    ├── Services/
    │   └── UserServiceTest.cs
    ├── Repositories/
    │   └── UserRepositoryTest.cs
    └── Integration/
        └── UsersControllerIntegrationTest.cs
```

## Running Tests and Coverage

```powershell
# Run all tests
dotnet test

# Run tests with coverage
dotnet test --collect:"XPlat Code Coverage"

# Generate HTML coverage report (requires ReportGenerator)
dotnet tool install -g dotnet-reportgenerator-globaltool
reportgenerator -reports:"**/coverage.cobertura.xml" -targetdir:"coveragereport" -reporttypes:Html
```

Add to `.csproj`:
```xml
<ItemGroup>
  <PackageReference Include="coverlet.collector" Version="6.0.0" />
</ItemGroup>
```

## C# Testing Checklist

Before submitting unit tests:

- [ ] Test class named `<ClassUnderTest>Test` (singular)
- [ ] All mocks declared inline with `MockBehavior.Strict`
- [ ] Mock variables use underscore-prefixed camelCase, no abbreviations, suffixed with `Mock`
- [ ] Constructor creates the service mock with factory constructor syntax (no `[TestInitialize]` — that's MSTest/NUnit, not xUnit)
- [ ] `FakeTimeProvider` used for testable time
- [ ] Test methods follow `MethodName_ShouldResult_WhenCondition` naming convention
- [ ] All public/internal methods on SUT are `virtual` (no non-virtual, no static on service classes)
- [ ] Non-virtual or static methods flagged as warnings if encountered
- [ ] Virtual methods tested with `.CallBase()` when under test
- [ ] All Setup calls chained with `.Verifiable(Times.X)`
- [ ] Every test calls `.VerifyAll()` on all mocks
- [ ] Exception tests use `Assert.ThrowsAsync<T>` / `Assert.Throws<T>`
- [ ] Parameter matching maximized (avoid `It.IsAny` when specific values can be checked)
- [ ] AAA pattern used (Arrange-Act-Assert with comments)
- [ ] `async Task` used for async test methods
- [ ] 80%+ code coverage achieved
- [ ] All tests pass (`dotnet test` succeeds)
- [ ] TDD workflow followed (tests written before implementation)

These testing standards are mandatory for all C# projects.
