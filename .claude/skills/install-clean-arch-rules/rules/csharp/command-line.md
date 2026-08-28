# C# CLI Implementation (System.CommandLine)

## Package & Project Setup

- Package: `System.CommandLine` (NuGet) — currently in prerelease (2.x beta) but production-ready and actively maintained by Microsoft
- Project type: `<OutputType>Exe</OutputType>`, target `net9.0`
- Enable nullable reference types and implicit usings

## Structure: RootCommand + Subcommands

```csharp
// Program.cs
var rootCommand = new RootCommand("MyApp - description of the tool");
rootCommand.AddGlobalOption(verboseOption);
rootCommand.AddGlobalOption(jsonOption);

var dbCommand = new Command("db", "Database operations");
dbCommand.AddCommand(new MigrateCommand(services));
rootCommand.AddCommand(dbCommand);

return await rootCommand.InvokeAsync(args);
```

- `RootCommand`: one per app, holds global options (`--verbose`, `--json`, `--version`)
- `Command`: subcommands, organized by topic (`db`, `user`, `report`)
- `Option<T>`: typed options — always provide long name + short alias where appropriate
- `Argument<T>`: typed positional arguments — use sparingly

## Typed Options and Arguments

```csharp
var outputOption = new Option<OutputFormat>(
    new[] { "--output", "-o" },
    getDefaultValue: () => OutputFormat.Text,
    description: "output format");

var fileArgument = new Argument<FileInfo>(
    "file",
    description: "path to the input file");
```

- Use enums for constrained choices (System.CommandLine auto-generates completion)
- Use `FileInfo` / `DirectoryInfo` for file/path arguments — System.CommandLine validates existence
- Always provide `description` — this appears in auto-generated help

## Handler Pattern: Context-Based, Thin Handlers

Commands have **zero service dependencies in their constructors** — they are pure routing/parsing constructs. Services are resolved inside `SetHandler` at invocation time via `context.BindingContext`, which has access to the `IHost` wired by `UseHost()`.

```csharp
// Command — no service dependencies in constructor
public class MigrateCommand : Command
{
    private readonly Option<bool> _dryRunOption =
        new("--dry-run", "Show pending migrations without applying");

    public MigrateCommand() : base("migrate", "Apply pending migrations")
    {
        AddOption(_dryRunOption);

        this.SetHandler(async (context) =>
        {
            // Resolve services at invocation time — not at construction time
            var host = context.BindingContext.GetRequiredService<IHost>();
            var migrationService = host.Services.GetRequiredService<IMigrationService>();

            var dryRun = context.ParseResult.GetValueForOption(_dryRunOption);
            var ct = context.GetCancellationToken();

            await migrationService.MigrateAsync(dryRun, ct);
        });
    }
}
```

Do NOT inject `IServiceProvider` or services into command constructors — they are not needed during parsing, only during handler execution.

## DI Integration via IHost + UseHost()

Use `CommandLineBuilder.UseHost()` to wire the `IHost` (and therefore all DI services) into the `BindingContext` available inside every handler:

```csharp
// Program.cs
var rootCommand = new RootCommand("MyApp — description of the tool");
rootCommand.AddGlobalOption(verboseOption);
rootCommand.AddGlobalOption(jsonOption);

var dbCommand = new Command("db", "Database operations");
dbCommand.AddCommand(new MigrateCommand());   // no services passed — zero deps
rootCommand.AddCommand(dbCommand);

return await new CommandLineBuilder(rootCommand)
    .UseDefaults()
    .UseHost(Host.CreateDefaultBuilder, host =>
    {
        host.ConfigureAppConfiguration((ctx, config) =>
        {
            config.AddJsonFile("appsettings.json", optional: true);
            config.AddEnvironmentVariables(prefix: "MYAPP_");
        });
        host.ConfigureServices((ctx, services) =>
        {
            services.AddSingleton(TimeProvider.System);
            services.AddScoped<IMigrationService, MigrationService>();
        });
    })
    .Build()
    .InvokeAsync(args);
```

## CancellationToken

Always pass `CancellationToken` to async operations. Get it from the invocation context — `UseHost()` wires Ctrl+C automatically:

```csharp
this.SetHandler(async (context) =>
{
    var cancellationToken = context.GetCancellationToken();
    await service.DoWorkAsync(cancellationToken);
});
```

## Exit Codes

Return an `int` from handlers (or set `context.ExitCode`):

```csharp
this.SetHandler(async (context) =>
{
    var cancellationToken = context.GetCancellationToken();
    try
    {
        await service.MigrateAsync(cancellationToken);
        // exit code 0 by default on success
    }
    catch (OperationCanceledException)
    {
        context.Console.Error.WriteLine("Cancelled.");
        context.ExitCode = 1;
    }
});
```

- Do NOT throw unhandled exceptions for business errors — set `context.ExitCode = 1` and write to `context.Console.Error`
- Reserve `ExitCode = 2` for argument/usage errors (System.CommandLine sets this automatically on parse failures)

## Console Abstraction (Testability)

Use `context.Console` (type `IConsole`) — never `System.Console` directly:

```csharp
// CORRECT: testable
context.Console.Out.WriteLine("Operation completed.");
context.Console.Error.WriteLine("error: something went wrong.");

// WRONG: untestable
Console.WriteLine("Operation completed.");
```

In tests, pass a `TestConsole` instance to `InvokeAsync` and assert on `.Out.ToString()`.

## Output Formatting Strategy

Support human-readable (default) and JSON (`--json`) output via a formatter:

```csharp
public interface IOutputFormatter
{
    void Write<T>(IConsole console, T value);
}

public class TextOutputFormatter : IOutputFormatter { ... }
public class JsonOutputFormatter : IOutputFormatter { ... }
```

Register the correct formatter based on the `--json` global option in the DI setup.

## Logging

- Use `ILogger<T>` — never `Console.WriteLine` for diagnostics
- Wire `Microsoft.Extensions.Logging` via `IHost` (same as service hosting)
- Default log level: `Warning` in normal mode; `Information` in `--verbose` mode
- Suppress all log output in `--quiet` mode
- Logs go to stderr (configure console logging to stderr, not stdout)

## Testing CLI Commands

Invoke the full command pipeline in tests using string args:

```csharp
var console = new TestConsole();
var exitCode = await rootCommand.InvokeAsync("db migrate --dry-run", console);

Assert.Equal(0, exitCode);
Assert.Contains("2 pending migrations", console.Out.ToString());
```

## csproj Configuration

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>net9.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <RootNamespace>MyProject.Cli</RootNamespace>
    <AssemblyName>MyProject.Cli</AssemblyName>
  </PropertyGroup>
</Project>
```

## CLI Development Checklist

- [ ] `System.CommandLine` used for all option/argument parsing (no manual `args[]` parsing)
- [ ] All options have `--long-form`; common ones have `-s` short form
- [ ] `--help`, `--version`, `--verbose`, `--quiet`, `--json` wired on root command
- [ ] `context.Console` used (not `System.Console`)
- [ ] CancellationToken passed to all async calls (from `context.GetCancellationToken()`)
- [ ] Exit codes: 0 success, 1 failure, 2 bad args
- [ ] Errors and warnings written to `context.Console.Error`
- [ ] `IHost` + `UseHost()` used for DI and configuration
- [ ] Handlers are thin — business logic in `.Implementation` services
- [ ] Commands are each in their own class file (`Commands/` folder)
- [ ] `--dry-run` supported for any destructive operation
