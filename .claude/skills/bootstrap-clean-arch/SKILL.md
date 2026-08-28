---
name: bootstrap-clean-arch
description: Scaffold one or more clean architecture projects and generate per-module CLAUDE.md files. Reads modularization rules from the deployed /rules folder to determine project structure, dependency boundaries, and applicable rules. Use when starting a new project, adding modules to an existing solution, or regenerating CLAUDE.md files.
argument-hint: "[ProjectNamespace] [--modules Module1,Module2,...]"
---

# Bootstrap Clean Architecture Project

Scaffold a clean architecture .NET solution (or add modules to an existing one) and generate per-module CLAUDE.md files. All decisions are driven by the rules deployed in `<repo-root>/rules/`.

**Core behavior — every operation is upsert:**
- **Scaffolding** is conditional: only runs if the project does not already exist. Existing source code is never touched.
- **CLAUDE.md** is unconditional: always created or updated for every requested module, whether new or existing.

## Prerequisites

The `rules/` directory must exist at the repo root with at least `rules/csharp/modularization.md`. If it does not exist, tell the user to run `/install-clean-arch-rules` first and stop.

## Step 1 — Parse Arguments

Parse `$ARGUMENTS`:

- **`ProjectNamespace`** (optional): The root namespace for the solution (e.g., `Skillable.ToDos`, `MyCompany.Billing`). Drives all assembly naming via `[ProjectNamespace].[AssemblyType]`. If omitted, discover it from existing `.csproj`/`.sqlproj` files.
- **`--modules`** (optional): Comma-separated list of modules to operate on. Valid source module names: `Common`, `Abstractions`, `Implementation`, `Repository`, `Database`, `Client`, `Web.Core`, `Web.Server`, `Web.Api`, `Angular`, `React`, `Nextjs`, `Cli`. Test modules follow the pattern `[SourceModule].Tests` (e.g., `Common.Tests`, `Implementation.Tests`). If omitted, discover existing state first (Step 3), then present a review table and ask the user (Step 4).

If arguments are missing or ambiguous, proceed to discovery.

## Step 2 — Read the Rules

1. Read `<repo-root>/rules/` recursively to catalog all available rule files.
2. Read the **full content** of every rule file, paying special attention to:
   - `csharp/modularization.md` — assembly structure, dependency flow, folder layout, naming conventions. **Scaffolding only — do NOT include in CLAUDE.md `@` references.**
   - `csharp/scaffolding.md` — solution file setup, Central Package Management, `.csproj`/`.sqlproj` templates, NuGet package assignments, Angular `.esproj`, test project wiring, starter code conventions, and build verification. **Scaffolding only — do NOT include in CLAUDE.md `@` references.**
   - `common/database.md` — relational database schema conventions (table/column naming, indexes, constraints, relationships).
   - All other rule files — read for CLAUDE.md applicability.
3. Build a catalog of rules organized by:
   - **Language**: common (applies to all), csharp, typescript
   - **Concern**: coding-style, testing, security, domain, services, persistence, presentation, hosting, logging, database
   - **Applicability criteria**: What kind of project/module does each rule apply to?

## Step 3 — Discover Existing State

Scan the repository to determine what already exists:

1. Find all `.csproj`, `.sqlproj`, and `.esproj` files under `src/` **and** `tests/`.
2. For each found project, note: project name, type, path, and whether a `CLAUDE.md` exists alongside it.
3. Pair each test project to its source counterpart by name (e.g., `Skillable.ToDo.Implementation.Tests` pairs with `Skillable.ToDo.Implementation`).
4. Read the root `CLAUDE.md` if it exists.
5. Derive `ProjectNamespace` from existing project names if not provided in arguments.

## Step 4 — Review and Select Modules

If `--modules` was not provided:

### 4a. Pre-compute proposed changes

Before asking anything, compute what the skill would do to each discovered module:

- **Source modules with existing CLAUDE.md**: read the current `## Rules` section; diff it against the applicable rules from the Step 5b table. Identify rules to add and rules to remove.
- **Source modules without CLAUDE.md**: mark as "Create".
- **Test modules with existing CLAUDE.md**: same diff computation using the Tests column.
- **Test modules without CLAUDE.md**: mark as "Create".

### 4b. Present the review table

Show a unified table covering both source and test modules:

```
Source modules:
┌──────────────────┬───────────┬─────────────────────────────────────────────┐
│ Module           │ CLAUDE.md │ Proposed change                             │
├──────────────────┼───────────┼─────────────────────────────────────────────┤
│ Common           │ ✓         │ No changes                                  │
│ Repository       │ ✓         │ Remove common/logging.md                    │
│ Web.Server       │ ✓         │ Remove common/patterns.md                   │
│ Implementation   │ Missing   │ Create                                      │
└──────────────────┴───────────┴─────────────────────────────────────────────┘

Test projects:
┌──────────────────────┬───────────┬─────────────────────────────────────────┐
│ Module               │ CLAUDE.md │ Proposed change                         │
├──────────────────────┼───────────┼─────────────────────────────────────────┤
│ Common.Tests         │ Missing   │ Create (.csproj + CLAUDE.md)            │
│ Repository.Tests     │ ✓         │ No changes                              │
└──────────────────────┴───────────┴─────────────────────────────────────────┘
```

For modules with rule changes, list the specific rules being added (`+ rule`) or removed (`- rule`).

### 4c. Ask for confirmation

After the table, ask which modules to operate on. Offer batch options appropriate to the discovered state, for example:

```
Which modules should I operate on?
1. All — [N source + M test modules]
2. Source modules only
3. Test projects only
4. A specific subset — list which ones
```

Do NOT default to all modules. Do NOT write anything until the user confirms. If the user selects a subset, list exactly which modules will be affected before proceeding.

If `--modules` was provided, skip 4a–4c and proceed directly to Step 5 with those modules.

## Step 5 — Upsert Modules

For each selected module, apply upsert logic independently:

### 5a. Scaffold (conditional)

**Source modules** — if the project file (`.csproj` / `.sqlproj`) **does not exist**:
- Follow `csharp/scaffolding.md` and `csharp/modularization.md` as the joint source of truth.
- Create the project file, folder structure, and starter code.
- Add the project to the `.sln` file under the `src` solution folder.

If the project file **already exists**:
- Skip all scaffolding. Do not modify any source files.
- Log: `[Module] — project exists, scaffolding skipped.`

**Test modules** — if the test project does not exist:
- Follow the same language-appropriate scaffolding rules as for source modules (e.g., `csharp/scaffolding.md` for .NET). Those rules define the test project file format, dependencies, starter code, and solution wiring under the `tests/` folder.

If the test project **already exists**:
- Skip all scaffolding. Do not modify any test source files.
- Log: `[Module].Tests — project exists, scaffolding skipped.`

**Nextjs module — special case**: Next.js is not a .NET project. There is no `.csproj`/`.esproj` and it is never added to the `.sln` or included in `dotnet build`. Scaffold it as a standalone app per `csharp/scaffolding.md`'s "Next.js Frontend (standalone)" section, and pair it with `Web.Api` (not `Web.Server`) — see `csharp/modularization.md`'s "Next.js Frontend" section for why. No test-module `.csproj` counterpart applies either; Next.js tests live inside the Next.js app per `typescript/testing.md`.

### 5b. CLAUDE.md (always)

For every selected module, regardless of whether it was just scaffolded or already existed, create or update its CLAUDE.md.

**Determine applicable rules** using the minimum applicable set principle. Only include rules where the module genuinely needs that guidance:

| Rule File | Common | Abstractions | Implementation | Repository | Database | Client | Web.Core | Web.Server / Web.Api | Cli | Angular | React | Nextjs | Tests |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| common/coding-style.md | Y | Y | Y | Y | — | Y | Y | Y | Y | Y | Y | Y | Y |
| common/database.md | — | — | — | Y | Y | — | — | — | — | — | — | — | — |
| common/logging.md | — | — | Y | — | — | Y | Y | Y | Y | — | — | — | — |
| common/patterns.md | Y | Y | Y | Y | — | Y | Y | — | Y | Y | Y | Y | — |
| common/security.md | — | — | — | Y | — | Y | Y | Y | Y | Y | Y | Y | — |
| common/testing.md | — | — | — | — | — | — | — | — | — | — | — | — | Y |
| common/command-line.md | — | — | — | — | — | — | — | — | Y | — | — | — | — |
| csharp/coding-style.md | Y | Y | Y | Y | — | Y | Y | Y | Y | — | — | — | Y (C#) |
| csharp/domain.md | — | Y | — | — | — | — | — | — | — | — | — | — | — |
| csharp/services.md | — | — | Y | — | — | — | Y | — | Y | — | — | — | — |
| csharp/persistence.md | — | — | — | Y | — | — | — | Y | — | — | — | — | — |
| csharp/presentation.md | — | — | — | — | — | — | Y | Y | — | — | — | — | — |
| csharp/hosting.md | — | — | — | — | — | — | — | Y | Y | — | — | — | — |
| csharp/command-line.md | — | — | — | — | — | — | — | — | Y | — | — | — | — |
| csharp/security.md | — | — | — | Y | — | Y | Y | Y | Y | — | — | — | — |
| csharp/testing.md | — | — | — | — | — | — | — | — | — | — | — | — | Y (C#) |
| typescript/coding-style.md | — | — | — | — | — | — | — | — | — | Y | Y | Y | Y (TS) |
| typescript/css.md | — | — | — | — | — | — | — | — | — | Y | Y | Y | — |
| typescript/frontend-arch.md | — | — | — | — | — | — | — | — | — | Y | Y | Y | — |
| typescript/angular.md | — | — | — | — | — | — | — | — | — | Y | — | — | — |
| typescript/react.md | — | — | — | — | — | — | — | — | — | — | Y | — | — |
| typescript/nextjs.md | — | — | — | — | — | — | — | — | — | — | — | Y | — |
| typescript/patterns.md | — | — | — | — | — | — | — | — | — | Y | Y | Y | — |
| typescript/security.md | — | — | — | — | — | — | — | — | — | Y | Y | Y | — |
| typescript/testing.md | — | — | — | — | — | — | — | — | — | Y | Y | Y | Y (TS) |

**Generate CLAUDE.md content:**

```markdown
# [Project Name]

[Brief description of the module's purpose]

## Rules

@[relative path to rule file]
@[relative path to next rule file]
...

## Module Purpose

[1-3 sentences describing what this module does]

## Key Contents

[Bullet list of main classes, services, or components]

## Dependency Constraints

[Allowed and forbidden dependencies per modularization rules]
```

**Handle existing CLAUDE.md:**
- Existing file: preserve all content except `## Rules`. Apply only the diff confirmed in Step 4 — add the rules marked `+`, remove the rules marked `-`. Do not touch Module Purpose, Key Contents, or Dependency Constraints.
- No file: create from template above.

**Test module CLAUDE.md:** Use the same template. Module Purpose should name the source module under test and list what kinds of tests it contains (unit, integration). Dependency Constraints should list the tested assembly and the language-appropriate test frameworks. Apply rules from the **Tests** column of the table above, selecting `Y (C#)` rules for C# test projects and `Y (TS)` rules for TypeScript test projects.

### 5c. Root CLAUDE.md

- If no root `CLAUDE.md` exists: generate one describing the solution architecture, listing all modules, and documenting the deployed rules structure.
- If root `CLAUDE.md` exists: update the module listing to include any newly added modules. Preserve all existing content.

## Step 6 — Verify

1. If any .NET projects were newly scaffolded, run `dotnet build` on the solution to verify compilation.
2. If the build fails, fix before reporting success.
3. If only CLAUDE.md files were updated (all modules pre-existed), verify that all `@` reference paths resolve to actual rule files.
4. Database projects (`.sqlproj`) are excluded from `dotnet build` — see `csharp/scaffolding.md` for database build verification.
5. A scaffolded Nextjs module is excluded from `dotnet build` (it is not a .NET project). Verify it instead with `npm run build` inside the Next.js app directory.

## Step 7 — Report

### Module Status

| Module | Scaffolded | CLAUDE.md |
|---|---|---|
| [ProjectNamespace].Abstractions | Created | Created |
| [ProjectNamespace].Repository | Already existed — skipped | Updated |
| [ProjectNamespace].Database | Created | Created |
| ... | ... | ... |

### Dependency Graph

Show the dependency flow as a simple text diagram matching the modularization rules.

### Next Steps

Suggest what the user should do next (e.g., "Add your domain models to Abstractions", "Configure your DbContext in Repository", "Add tables to Database").

## Constraints

- **Rules MUST be deployed first** — if `/rules` does not exist, stop and tell the user to run `/install-clean-arch-rules`.
- **Follow modularization and scaffolding rules exactly** — do not invent structure beyond what they specify.
- **Never touch existing source code** — only CLAUDE.md files are always written (Rules section only).
- **Only operate on selected modules** — always confirm which modules to operate on before doing anything.
- **All generated code must follow the deployed coding rules** — read them before generating any code.
- **Relative paths in CLAUDE.md must be accurate** — double-check directory depth.
- **Keep generated code minimal** — just enough to compile and demonstrate the pattern. Do not over-engineer starter code.
