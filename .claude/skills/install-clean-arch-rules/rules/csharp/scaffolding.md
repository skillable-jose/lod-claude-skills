# C# Scaffolding

Defines the mechanics for scaffolding a .NET clean architecture solution: file formats, tooling commands, NuGet package assignments, and starter code conventions. Used by the `/bootstrap-clean-arch` skill — do not reference this file from module CLAUDE.md files.

---

## Solution Setup

### Directory Structure

```
<solution-root>/
├── src/                     ← source projects
├── tests/                   ← test projects
├── <SolutionName>.sln
└── Directory.Packages.props
```

### Solution File

Create `<SolutionName>.sln` at the repo root if one does not exist. Add projects with solution folders:

```bash
dotnet new sln -n <SolutionName>
dotnet sln add src/<ProjectName>/<ProjectName>.csproj --solution-folder src
dotnet sln add tests/<ProjectName>.Tests/<ProjectName>.Tests.csproj --solution-folder tests
```

If a `.sln` already exists, add only the new projects to it.

---

## Central Package Management (CRITICAL)

Every solution MUST have a `Directory.Packages.props` at the solution root. Individual `.csproj` files declare `<PackageReference>` **without a `Version` attribute** — versions are managed exclusively in this file.

### Directory.Packages.props Template

```xml
<Project>
  <PropertyGroup>
    <ManagePackageVersionsCentrally>true</ManagePackageVersionsCentrally>
  </PropertyGroup>
  <ItemGroup>
    <!-- DI / Extensions -->
    <PackageVersion Include="Microsoft.Extensions.DependencyInjection.Abstractions" Version="9.0.0" />
    <PackageVersion Include="Microsoft.Extensions.Hosting" Version="9.0.0" />
    <PackageVersion Include="Microsoft.Extensions.Logging.Abstractions" Version="9.0.0" />
    <PackageVersion Include="Microsoft.Extensions.Configuration.Abstractions" Version="9.0.0" />

    <!-- Entity Framework Core -->
    <PackageVersion Include="Microsoft.EntityFrameworkCore" Version="9.0.0" />
    <PackageVersion Include="Microsoft.EntityFrameworkCore.SqlServer" Version="9.0.0" />
    <PackageVersion Include="Microsoft.EntityFrameworkCore.Tools" Version="9.0.0" />
    <PackageVersion Include="Microsoft.EntityFrameworkCore.Design" Version="9.0.0" />

    <!-- Web / API -->
    <PackageVersion Include="Microsoft.AspNetCore.OpenApi" Version="9.0.0" />
    <PackageVersion Include="Swashbuckle.AspNetCore" Version="7.0.0" />

    <!-- CLI -->
    <PackageVersion Include="System.CommandLine" Version="2.0.0-beta4.22272.1" />

    <!-- Validation -->
    <PackageVersion Include="FluentValidation" Version="11.0.0" />
    <PackageVersion Include="FluentValidation.DependencyInjectionExtensions" Version="11.0.0" />

    <!-- Testing -->
    <PackageVersion Include="xunit" Version="2.9.0" />
    <PackageVersion Include="xunit.runner.visualstudio" Version="2.8.2" />
    <PackageVersion Include="Microsoft.NET.Test.Sdk" Version="17.11.0" />
    <PackageVersion Include="Moq" Version="4.20.72" />
    <PackageVersion Include="coverlet.collector" Version="6.0.2" />
  </ItemGroup>
</Project>
```

Update package versions to latest stable at time of scaffolding. Only declare packages that are actually used — add entries as modules are added.

---

## Project File (.csproj) Templates

### Base Properties (all .NET projects)

```xml
<PropertyGroup>
  <TargetFramework>net9.0</TargetFramework>
  <Nullable>enable</Nullable>
  <ImplicitUsings>enable</ImplicitUsings>
</PropertyGroup>
```

Match the `<TargetFramework>` of existing projects in the repo if any exist.

### Per-Module .csproj

**Common** — class library, no project or package dependencies:
```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net9.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
  </PropertyGroup>
</Project>
```

**Abstractions** — class library:
```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>...</PropertyGroup>
  <ItemGroup>
    <ProjectReference Include="..\[ProjectNamespace].Common\[ProjectNamespace].Common.csproj" />
  </ItemGroup>
</Project>
```

**Implementation** — class library:
```xml
<ItemGroup>
  <ProjectReference Include="..\[ProjectNamespace].Abstractions\[ProjectNamespace].Abstractions.csproj" />
  <ProjectReference Include="..\[ProjectNamespace].Common\[ProjectNamespace].Common.csproj" />
  <PackageReference Include="FluentValidation" />
  <PackageReference Include="FluentValidation.DependencyInjectionExtensions" />
  <PackageReference Include="Microsoft.Extensions.DependencyInjection.Abstractions" />
</ItemGroup>
```

**Repository** — class library:
```xml
<ItemGroup>
  <ProjectReference Include="..\[ProjectNamespace].Abstractions\[ProjectNamespace].Abstractions.csproj" />
  <ProjectReference Include="..\[ProjectNamespace].Common\[ProjectNamespace].Common.csproj" />
  <PackageReference Include="Microsoft.EntityFrameworkCore" />
  <PackageReference Include="Microsoft.EntityFrameworkCore.SqlServer" />
  <PackageReference Include="Microsoft.EntityFrameworkCore.Tools" />
  <PackageReference Include="Microsoft.Extensions.DependencyInjection.Abstractions" />
</ItemGroup>
```

**Client** — class library:
```xml
<ItemGroup>
  <ProjectReference Include="..\[ProjectNamespace].Abstractions\[ProjectNamespace].Abstractions.csproj" />
  <ProjectReference Include="..\[ProjectNamespace].Common\[ProjectNamespace].Common.csproj" />
  <PackageReference Include="Microsoft.Extensions.DependencyInjection.Abstractions" />
</ItemGroup>
```

**Web.Core** — class library (not a web app SDK; references ASP.NET Core packages implicitly via transitive deps from Web.Server/Web.Api):
```xml
<Project Sdk="Microsoft.NET.Sdk">
  <ItemGroup>
    <ProjectReference Include="..\[ProjectNamespace].Abstractions\[ProjectNamespace].Abstractions.csproj" />
    <ProjectReference Include="..\[ProjectNamespace].Implementation\[ProjectNamespace].Implementation.csproj" />
    <ProjectReference Include="..\[ProjectNamespace].Common\[ProjectNamespace].Common.csproj" />
    <PackageReference Include="Microsoft.AspNetCore.OpenApi" />
  </ItemGroup>
</Project>
```

**Web.Server** — ASP.NET Core web application:
```xml
<Project Sdk="Microsoft.NET.Sdk.Web">
  <ItemGroup>
    <ProjectReference Include="..\[ProjectNamespace].Web.Core\[ProjectNamespace].Web.Core.csproj" />
    <ProjectReference Include="..\[ProjectNamespace].Implementation\[ProjectNamespace].Implementation.csproj" />
    <ProjectReference Include="..\[ProjectNamespace].Repository\[ProjectNamespace].Repository.csproj" />
    <!-- Angular or React SPA — build-only reference, no assembly output -->
    <ProjectReference Include="..\[projectnamespace].client\[projectnamespace].client.esproj"
                      ReferenceOutputAssembly="false" />
  </ItemGroup>
</Project>
```

Omit the `.esproj` reference if neither an Angular nor a React module was requested (a project has at most one of the two — Angular and React are alternative choices for the same `.client` slot). Never add this reference for a Next.js module — Next.js is not an `.esproj`; see "Next.js Frontend (standalone)" below.

**Web.Api** — ASP.NET Core web application:
```xml
<Project Sdk="Microsoft.NET.Sdk.Web">
  <ItemGroup>
    <ProjectReference Include="..\[ProjectNamespace].Web.Core\[ProjectNamespace].Web.Core.csproj" />
    <ProjectReference Include="..\[ProjectNamespace].Implementation\[ProjectNamespace].Implementation.csproj" />
    <ProjectReference Include="..\[ProjectNamespace].Repository\[ProjectNamespace].Repository.csproj" />
    <PackageReference Include="Swashbuckle.AspNetCore" />
  </ItemGroup>
</Project>
```

**Cli** — console application:
```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>net9.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
  </PropertyGroup>
  <ItemGroup>
    <ProjectReference Include="..\[ProjectNamespace].Abstractions\[ProjectNamespace].Abstractions.csproj" />
    <ProjectReference Include="..\[ProjectNamespace].Implementation\[ProjectNamespace].Implementation.csproj" />
    <ProjectReference Include="..\[ProjectNamespace].Repository\[ProjectNamespace].Repository.csproj" />
    <ProjectReference Include="..\[ProjectNamespace].Common\[ProjectNamespace].Common.csproj" />
    <PackageReference Include="System.CommandLine" />
    <PackageReference Include="Microsoft.Extensions.Hosting" />
  </ItemGroup>
</Project>
```

---

## Angular .esproj

Project name is lowercase: `[projectnamespace].client`. Place it under `src/`.

```xml
<!-- [projectnamespace].client.esproj -->
<Project Sdk="Microsoft.VisualStudio.JavaScript.SDK/1.0">
  <PropertyGroup>
    <StartupCommand>npm start</StartupCommand>
    <JavaScriptTestRoot>src/</JavaScriptTestRoot>
    <JavaScriptTestFramework>Jasmine</JavaScriptTestFramework>
    <GeneratePkgDefFile>false</GeneratePkgDefFile>
    <ShouldRunBuildScript>false</ShouldRunBuildScript>
    <TypeScriptCompileBlocked>true</TypeScriptCompileBlocked>
  </PropertyGroup>
</Project>
```

Additional files to create (see `typescript/angular.md` for package and config details):

| File | Notes |
|---|---|
| `angular.json` | Set `outputPath` to `../[ProjectNamespace].Web.Server/wwwroot` |
| `package.json` | Angular dependencies per `typescript/angular.md` |
| `tsconfig.json` | TypeScript config per `typescript/angular.md` |
| `proxy.conf.js` | Dev proxy: route `/api` to `https://localhost:<port>` |
| `src/app/domain/` | Types, interfaces, domain errors |
| `src/app/repositories/` | HTTP services (`*-api.service.ts`) |
| `src/app/services/` | Business logic services |
| `src/app/state/` | Reactive state (signals, `resource()`) |
| `src/app/components/` | Presentational components + `shared/` + `layout/` |
| `src/app/pages/` | Smart page components with lazy routes |
| `src/app/core/` | Providers, guards, interceptors |
| `src/main.ts` | Bootstrap entry point |
| `src/app/app.component.ts` | Root component |
| `src/app/app.config.ts` | Application configuration |
| `src/app/app.routes.ts` | Root route definitions |

---

## React SPA (Vite) .esproj

Project name is lowercase: `[projectnamespace].client`. Place it under `src/`. Omit the reference from `Web.Server.csproj` (see above) if no React module was requested.

```xml
<!-- [projectnamespace].client.esproj -->
<Project Sdk="Microsoft.VisualStudio.JavaScript.SDK/1.0">
  <PropertyGroup>
    <StartupCommand>npm run dev</StartupCommand>
    <JavaScriptTestRoot>src/</JavaScriptTestRoot>
    <JavaScriptTestFramework>Vitest</JavaScriptTestFramework>
    <GeneratePkgDefFile>false</GeneratePkgDefFile>
    <ShouldRunBuildScript>false</ShouldRunBuildScript>
    <TypeScriptCompileBlocked>true</TypeScriptCompileBlocked>
  </PropertyGroup>
</Project>
```

Additional files to create (see `typescript/react.md` for the layer implementation):

| File | Notes |
|---|---|
| `vite.config.ts` | Set `build.outDir` to `../[ProjectNamespace].Web.Server/wwwroot` |
| `package.json` | React + Vite dependencies (`react`, `react-dom`, `@tanstack/react-query`, `zustand`, `vite`) |
| `tsconfig.json` | TypeScript config |
| `.env` / `.env.development` | `VITE_API_BASE_URL` — dev proxy target |
| `vite.config.ts` server.proxy | Route `/api` to `https://localhost:<port>` |
| `src/domain/` | Types, interfaces, domain errors |
| `src/repositories/` | HTTP implementations |
| `src/services/` | Business logic services |
| `src/state/` | React Query hooks + Zustand stores |
| `src/components/` | Presentational components + `shared/` + `layout/` |
| `src/pages/` | Smart page components |
| `src/core/` | `providers.tsx` composition root, `api-client.ts` |
| `src/main.tsx` | Bootstrap entry point (renders `<ServicesProvider>`) |

---

## Next.js Frontend (standalone)

Next.js is **not** an `.esproj` and is **not** added to the `.sln`. It is a sibling Node.js application scaffolded and run independently, wired to call `[ProjectNamespace].Web.Api` over HTTP — see `csharp/modularization.md` → "Next.js Frontend" for the architectural rationale and `typescript/nextjs.md` for the layer implementation.

**Scaffold command** (run from the repo root, outside `src/` and outside the `.sln`):

```bash
npx create-next-app@latest [projectnamespace].client --typescript --app --src-dir --eslint
```

Additional setup:

| File | Notes |
|---|---|
| `.env.local` | `API_BASE_URL` (server-side, internal) and `NEXT_PUBLIC_API_BASE_URL` (client-side, public) pointing at `Web.Api`'s dev URL |
| `src/domain/` | Types, interfaces, domain errors |
| `src/repositories/` | HTTP implementations |
| `src/services/` | Business logic services |
| `src/state/` | React Query hooks + Zustand stores (Client Components only) |
| `src/components/` | `'use client'` presentational components needing interactivity |
| `src/core/services.server.ts` | Server Component / Route Handler composition root |
| `src/core/providers.tsx` | Client Component composition root |
| `src/app/` | App Router routes, layouts, and optional Route Handlers |

**Backend wiring**: `Web.Api`'s `Program.cs` must add a CORS policy allowing the Next.js app's origin — see `csharp/hosting.md` for where CORS fits in the middleware pipeline order.

**Build verification**: `dotnet build` does not cover this module. Verify with `npm run build` inside `[projectnamespace].client/`.

---

## Database Project (.sqlproj)

Use the SDK-style SQL project format (`Microsoft.Build.Sql`), which supports `dotnet build`.

### Project File

```xml
<!-- [ProjectNamespace].Database.sqlproj -->
<Project DefaultTargets="Build">
  <Sdk Name="Microsoft.Build.Sql" Version="0.2.0-preview" />
  <PropertyGroup>
    <Name>[ProjectNamespace].Database</Name>
    <DSP>Microsoft.Data.Tools.Schema.Sql.SqlAzureV12DatabaseSchemaProvider</DSP>
    <ModelCollation>1033, CI</ModelCollation>
  </PropertyGroup>
</Project>
```

### Adding to Solution

```bash
dotnet sln add src/[ProjectNamespace].Database/[ProjectNamespace].Database.sqlproj --solution-folder src
```

### DACPAC References

To reference another database project (e.g., shared utilities):

```xml
<ItemGroup>
  <ArtifactReference Include="path/to/Other.Database.dacpac">
    <HintPath>path/to/Other.Database.dacpac</HintPath>
    <SuppressMissingDependenciesErrors>false</SuppressMissingDependenciesErrors>
    <DatabaseVariableLiteralValue>OtherDb</DatabaseVariableLiteralValue>
  </ArtifactReference>
</ItemGroup>
```

### Starter Files

| File | Notes |
|---|---|
| `Security/[schema].sql` | `CREATE SCHEMA [schema] AUTHORIZATION [dbo]` |
| `[schema]/Tables/Example.sql` | One example table following `common/database.md` conventions |

### Build Verification

```bash
dotnet build src/[ProjectNamespace].Database/[ProjectNamespace].Database.sqlproj
```

Database projects are excluded from solution-wide `dotnet build` by default. Verify them independently.

---

## Test Projects

For each source module, create `tests/[ProjectNamespace].[AssemblyType].Tests/`.

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net9.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <IsPackable>false</IsPackable>
  </PropertyGroup>
  <ItemGroup>
    <ProjectReference Include="..\..\src\[ProjectNamespace].[AssemblyType]\[ProjectNamespace].[AssemblyType].csproj" />
    <PackageReference Include="xunit" />
    <PackageReference Include="xunit.runner.visualstudio" />
    <PackageReference Include="Microsoft.NET.Test.Sdk" />
    <PackageReference Include="Moq" />
    <PackageReference Include="coverlet.collector">
      <IncludeAssets>runtime; build; native; contentfiles; analyzers; buildtransitive</IncludeAssets>
      <PrivateAssets>all</PrivateAssets>
    </PackageReference>
  </ItemGroup>
</Project>
```

Include a placeholder test class so the project compiles:

```csharp
namespace [ProjectNamespace].[AssemblyType].Tests;

public class PlaceholderTests
{
    [Fact]
    public void Placeholder() => Assert.True(true);
}
```

---

## Starter Code

Generate minimal files per module — just enough to compile and demonstrate the module's role. All generated code MUST follow `csharp/coding-style.md` and `common/patterns.md`.

| Module | Files to Create |
|---|---|
| Common | README placeholder only |
| Abstractions | `Interfaces/IExampleService.cs`, `Models/ExampleModel.cs` |
| Implementation | `Services/ExampleService.cs` (implements IExampleService), `Extensions/ServiceCollectionExtensions.cs` |
| Repository | `Contexts/AppDbContext.cs`, `Entities/ExampleEntity.cs`, `Repositories/ExampleRepository.cs`, `Extensions/ServiceCollectionExtensions.cs` |
| Client | `Clients/ExampleApiClient.cs`, `Extensions/ServiceCollectionExtensions.cs` |
| Web.Core | `Controllers/ExampleController.cs` (thin — delegates to IExampleService) |
| Web.Server | `Program.cs` (HTTPS, static files, SPA fallback, DI wiring) |
| Web.Api | `Program.cs` (HTTPS, Swagger UI, DI wiring) |
| Cli | `Program.cs` (RootCommand + IHost via `CommandLineBuilder.UseHost()`), `Commands/ExampleCommand.cs` |
| Database | `Security/[schema].sql` (schema creation), `[schema]/Tables/Example.sql` (example table) |

### Starter Code Constraints

- No default parameters — use overloads
- All async methods accept `CancellationToken ct` as the last parameter
- `Program.cs` for Cli: follow patterns in `csharp/command-line.md`
- Keep files minimal — no over-engineering, no speculative abstractions

---

## Build Verification

After scaffolding, verify compilation:

```bash
dotnet build <SolutionName>.sln
```

If the build fails, diagnose and fix before reporting success. Common causes:
- Package not declared in `Directory.Packages.props`
- Project reference path incorrect (check relative depth `src/` vs `tests/`)
- Namespace mismatch between `.csproj` and generated code
