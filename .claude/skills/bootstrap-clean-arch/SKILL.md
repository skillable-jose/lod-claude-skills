---
name: bootstrap-clean-arch
description: Scaffold one or more clean architecture projects and generate CLAUDE.md files (a root file in the org's brownfield contract shape, plus per-module files) from the deployed /rules folder, citing the specific ADRs those rules are grounded in. Use when starting a new project, adding modules to an existing solution, or regenerating CLAUDE.md files.
version: 1.1.0
applies_to: [dotnet, node]
references: [ADR-AA002, ADR-IDP001, ADR-API0001, ADR-API0002, ADR-API0003, ADR-API0004, ADR-API0005, ADR-AA015]
authors: Jose Murillo Castro
last_updated: 2026-08-28
argument-hint: "[ProjectNamespace] [--modules Module1,Module2,...]"
---

# Bootstrap Clean Architecture Project

## What this does

Scaffolds a clean architecture .NET solution (or adds modules to an existing one) and generates CLAUDE.md files — a root file plus one per module. All decisions are driven by the rules deployed in `<repo-root>/rules/` (see `install-clean-arch-rules`).

**Core behavior — every operation is upsert:**
- **Scaffolding** is conditional: only runs if the project does not already exist. Existing source code is never touched.
- **CLAUDE.md** is unconditional: always created or reconciled for every requested module (and the root), whether new or existing.

## When to use this

Use when starting a new .NET/frontend module, adding modules to an existing solution, or regenerating/reconciling CLAUDE.md files against the deployed rules.

**Do NOT use this for:**
- Deploying the rule files themselves — that's `install-clean-arch-rules`, a **prerequisite** for this skill.
- Writing or extending unit tests inside a module this skill scaffolded — that's `write-unit-tests`.
- A greenfield service cloned from the org's `skillable-template-dotnet-react` template — that's `ai-enablement`'s `bootstrap-dotnet-from-template` / `extend-dotnet-solution` / `scaffold-minimal`, a different, template-repo-centric path (if that plugin is installed). Use this skill instead when the target repo already exists with its own structure and isn't being bootstrapped from that template.

## Inputs

| Input | Type | Notes |
|---|---|---|
| `ProjectNamespace` | string, optional | Root namespace for the solution (e.g., `Skillable.ToDos`). Drives assembly naming via `[ProjectNamespace].[AssemblyType]`. If omitted, discover it from existing `.csproj`/`.sqlproj` files. |
| `--modules` | comma-separated list, optional | Valid source modules: `Common`, `Abstractions`, `Implementation`, `Repository`, `Database`, `Client`, `Web.Core`, `Web.Server`, `Web.Api`, `Angular`, `React`, `Cli`. Next.js is intentionally not offered — see `csharp/modularization.md`. Test modules follow `[SourceModule].Tests`. If omitted, discover existing state first (Step 3) and present a review table (Step 4). |

## Outputs

- Scaffolded project files/folders for newly requested modules that don't already exist (existing source is never touched).
- A **root CLAUDE.md**, in the org's brownfield-contract shape (see Step 5c) — created or additively reconciled.
- A **per-module CLAUDE.md** for every selected module — created or additively reconciled, pointing back to the root file's standards section.
- A module-status table, dependency graph, and next-steps report (Step 7).

## Prerequisites

The `rules/` directory must exist at the repo root with at least `rules/csharp/modularization.md`. If it does not exist, tell the user to run `/install-clean-arch-rules` first and stop.

## How it works

### Step 1 — Parse Arguments

Parse `$ARGUMENTS` per the Inputs table above. If arguments are missing or ambiguous, proceed to discovery.

### Step 2 — Read the Rules

1. Read `<repo-root>/rules/` recursively to catalog all available rule files.
2. Read the **full content** of every rule file, paying special attention to:
   - `csharp/modularization.md` — assembly structure, dependency flow, folder layout, naming conventions. **Scaffolding only — do NOT include in per-module CLAUDE.md `@` references.**
   - `csharp/scaffolding.md` — solution file setup, Central Package Management, `.csproj`/`.sqlproj` templates, NuGet package assignments, `.esproj`, test project wiring, starter code conventions, and build verification. **Scaffolding only — do NOT include in per-module CLAUDE.md `@` references.**
   - `common/database.md` — relational database schema conventions.
   - All other rule files — read for CLAUDE.md applicability, and note which cite a specific `engineering-decisions` ADR by ID (grep the deployed `rules/` tree for `ADR-`) — those citations feed Step 5c's root CLAUDE.md §3.
3. Build a catalog of rules organized by language (common/csharp/typescript), concern, and applicability criteria.

### Step 3 — Discover Existing State

1. Find all `.csproj`, `.sqlproj`, and `.esproj` files under `src/` and `tests/`.
2. For each, note: project name, type, path, whether a CLAUDE.md exists alongside it.
3. Pair each test project to its source counterpart by name.
4. Read the root `CLAUDE.md` if it exists — note whether it already has the 5-section shape from a prior run of this skill, or a different/no shape (first run on a brownfield repo).
5. Derive `ProjectNamespace` from existing project names if not provided.
6. If a pre-existing (non-empty) codebase is being operated on, note anything observable about its actual conventions (test framework/style, DI idiom, error handling, naming) that Step 5c's root CLAUDE.md §"Existing Conventions" will need — this skill has no dedicated survey step, so keep this to what Step 2/3 already surfaced, not a separate deep audit.

### Step 4 — Review and Select Modules

If `--modules` was not provided:

**4a. Pre-compute proposed changes.** For each discovered module: source modules with an existing CLAUDE.md — diff the current `## Rules` against Step 5b's applicable-rules table, listing rules to add/remove; without one — mark "Create." Same for test modules against the Tests column.

**4b. Present the review table:**

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

**4c. Ask for confirmation.** Offer batch options (All / Source only / Test only / A specific subset). Do NOT default to all modules. Do NOT write anything until the user confirms.

If `--modules` was provided, skip 4a–4c and proceed directly to Step 5.

### Step 5 — Upsert Modules

For each selected module, apply upsert logic independently:

#### 5a. Scaffold (conditional)

**Source modules** — if the project file does not exist: follow `csharp/scaffolding.md` and `csharp/modularization.md` as the joint source of truth; create the project file, folder structure, starter code; add it to the `.sln` under `src`. If it already exists: skip all scaffolding, log `[Module] — project exists, scaffolding skipped.`

**Test modules** — same conditional logic under `tests/`, following the language-appropriate scaffolding rules.

#### 5b. Per-module CLAUDE.md (always)

For every selected module, create or update its CLAUDE.md.

**Determine applicable rules** using the minimum applicable set principle. Only include rules where the module genuinely needs that guidance:

| Rule File | Common | Abstractions | Implementation | Repository | Database | Client | Web.Core | Web.Server / Web.Api | Cli | Angular | React | Tests |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| common/coding-style.md | Y | Y | Y | Y | — | Y | Y | Y | Y | Y | Y | Y |
| common/database.md | — | — | — | Y | Y | — | — | — | — | — | — | — |
| common/logging.md | — | — | Y | — | — | Y | Y | Y | Y | — | — | — |
| common/patterns.md | Y | Y | Y | Y | — | Y | Y | — | Y | Y | Y | — |
| common/security.md | — | — | — | Y | — | Y | Y | Y | Y | Y | Y | — |
| common/testing.md | — | — | — | — | — | — | — | — | — | — | — | Y |
| common/command-line.md | — | — | — | — | — | — | — | — | Y | — | — | — |
| csharp/coding-style.md | Y | Y | Y | Y | — | Y | Y | Y | Y | — | — | Y (C#) |
| csharp/domain.md | — | Y | — | — | — | — | — | — | — | — | — | — |
| csharp/services.md | — | — | Y | — | — | — | Y | — | Y | — | — | — |
| csharp/persistence.md | — | — | — | Y | — | — | — | Y | — | — | — | — |
| csharp/presentation.md | — | — | — | — | — | — | Y | Y | — | — | — | — |
| csharp/hosting.md | — | — | — | — | — | — | — | Y | Y | — | — | — |
| csharp/command-line.md | — | — | — | — | — | — | — | — | Y | — | — | — |
| csharp/security.md | — | — | — | Y | — | Y | Y | Y | Y | — | — | — |
| csharp/testing.md | — | — | — | — | — | — | — | — | — | — | — | Y (C#) |
| typescript/coding-style.md | — | — | — | — | — | — | — | — | — | Y | Y | Y (TS) |
| typescript/css.md | — | — | — | — | — | — | — | — | — | Y | Y | — |
| typescript/frontend-arch.md | — | — | — | — | — | — | — | — | — | — | Y | — |
| typescript/angular.md | — | — | — | — | — | — | — | — | — | Y | — | — |
| typescript/react.md | — | — | — | — | — | — | — | — | — | — | Y | — |
| typescript/patterns.md | — | — | — | — | — | — | — | — | — | Y | Y | — |
| typescript/security.md | — | — | — | — | — | — | — | — | — | Y | Y | — |
| typescript/testing.md | — | — | — | — | — | — | — | — | — | Y | Y | Y (TS) |

Next.js is not in this table — it's not offered as a module choice (see Step 1/Inputs). `typescript/frontend-arch.md` applies to React only; Angular's architecture is described independently in `typescript/angular.md` and isn't governed by the same ADR.

**Template:**

```markdown
# [Project Name]

> Standards for new code are cited by ADR in the root `CLAUDE.md`'s "Standards for New Code" section — this file lists only the rule files applicable to this module.

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

**Handle existing CLAUDE.md:** preserve all content except `## Rules`; apply only the diff confirmed in Step 4. **No file:** create from template above (with the pointer line at top).

**Test module CLAUDE.md:** same template; Module Purpose names the source module under test and the kinds of tests it contains; Dependency Constraints lists the tested assembly and test frameworks.

#### 5c. Root CLAUDE.md

Adapted from the org's brownfield CLAUDE.md contract (`ai-enablement`'s `_shared/claude-md-brownfield.md`) to what this skill can actually produce on its own — it has no codebase-survey or architect-mapping step, so §2 and §4 below are necessarily lighter than what that contract describes for the full `augment-orchestrator` pipeline. **Preserve all existing upsert/additive-merge behavior** — this changes the template's shape and citations, not the reconciliation rules.

**If no root CLAUDE.md exists, generate one with this structure:**

```markdown
# [Project Name]

[One-paragraph purpose. Solution/project map — one line per module. How to build/run/test — the actual commands from Step 3's discovery, plus any gotcha the discovery step actually found.]

## Existing Conventions (Observed)

> These describe the existing code. When modifying existing files, match them. They are NOT the bar for new code — see "Standards for New Code" below.

[Facts with file examples, from Step 3.6 — test framework/style, DI idiom, error handling, naming, module layout. Omit this whole section entirely on a from-scratch scaffold with nothing yet to observe — do not write a section with nothing in it.]

## Standards for New Code

All new code in this repo follows the rules deployed under `/rules`, each grounded in a specific `engineering-decisions` ADR where one exists:

| Rule file | Grounded in |
|---|---|
| `csharp/domain.md` (Result<T> pattern, reservation rule) | ADR-IDP001 |
| `csharp/modularization.md` (layering intent) | ADR-AA002 (advisory — no must/shall language; recognize the vocabulary mapping noted in the file rather than treating a name mismatch as a violation) |
| `csharp/presentation.md` (versioning, response shape, include validation, 403-vs-404 disclosure) | ADR-API0002, ADR-API0003, ADR-API0004, ADR-API0005 (each supersedes part of ADR-API0001, which still governs everything they don't) |
| `typescript/frontend-arch.md` (feature-folder architecture) | ADR-AA015 (see the file's own citation-bug note — the ID doesn't currently resolve in the `engineering-decisions` catalog) |
[... one row per other deployed rule file that cites an ADR — derive from Step 2's grep, don't invent citations for rule files that don't have one]

### Skillable engineering tools — skill-first workflow

If the `ai-enablement` `skillable-engineering-tools` plugin is installed in this environment (check the available-skills list for `skillable-engineering-tools:` entries):
1. **Preflight:** if none appear, stop and ask the user to install the plugin rather than concluding it doesn't exist — it ships in the plugin, not a repo-local `.claude/skills/` directory.
2. **Skill-first rule:** before any engineering task, scan those skills for one that covers it and use it proactively; proceed manually only when none does, and say so explicitly.
3. **Hard invariants:** commits/PRs via that plugin's `commit-and-pr-*` skill, never direct to the default branch; architectural decisions via `propose-adr`; new bounded contexts via the stack's `extend-*-solution`.

Independently of whether that plugin is installed, the `/rules` files deployed by `install-clean-arch-rules` and cited above are this repo's standards regardless — this skill and `write-unit-tests` are how they get applied.

## Convention Adjudication Table

One row per dimension where the observed host convention and a cited standard actually differ. Default: **the standard wins** — a row resolves to "host" only as a recorded deviation with a real rationale (not just "consistency with what's there").

| Dimension | Host does | Standard says | New code follows | Rationale |
|---|---|---|---|---|
| C# test framework | xUnit + Moq(Strict), AAA comments | `ai-enablement`'s generic default: MSTest + FluentAssertions, no AAA comments (`csharp-coding-standards.md`) | **Host** (xUnit + Moq) | Recorded deviation, per `csharp/testing.md` — an established, working test suite is a legitimate reason to keep the host's framework; see that file for the full adjudication. |
[... add further rows by hand when a human identifies another host-vs-standard conflict — this skill has no automated survey/architect step to discover them on its own]

## Deployed Rules

[List of every file under /rules, as deployed by install-clean-arch-rules]

<!-- Generated/updated by bootstrap-clean-arch on [date]. "Standards for New Code" lists the /rules files this run deployed and the ADRs they cite; the adjudication table above is not exhaustive — this skill has no automated convention-survey step, unlike the org's full augment-orchestrator pipeline. Reconcile additively; do not regenerate. -->
```

**If a root CLAUDE.md already exists:** additive merge only — add newly discovered modules to the project map, add any newly-cited ADR to the Standards table, add the C# test-framework adjudication row if absent and applicable. Never rewrite host-authored guidance; if something is factually wrong, correct it with an inline `<!-- corrected by bootstrap-clean-arch run on <date>: <why> -->` note rather than deleting it.

### Step 6 — Verify

1. If any .NET projects were newly scaffolded, run `dotnet build` on the solution to verify compilation. Fix before reporting success.
2. If only CLAUDE.md files were updated, verify every `@` reference path resolves to an actual rule file.
3. Database projects (`.sqlproj`) are excluded from `dotnet build` — see `csharp/scaffolding.md`.

### Step 7 — Report

**Module Status:**

| Module | Scaffolded | CLAUDE.md |
|---|---|---|
| [ProjectNamespace].Abstractions | Created | Created |
| [ProjectNamespace].Repository | Already existed — skipped | Updated |
| ... | ... | ... |

**Dependency Graph:** a simple text diagram matching the modularization rules.

**Next Steps:** what the user should do next (e.g., "Add your domain models to Abstractions").

## Anti-patterns

- ❌ Scaffolding or generating CLAUDE.md content before `/rules` is deployed — stop and tell the user to run `/install-clean-arch-rules` first.
- ❌ Inventing project structure beyond what `csharp/modularization.md`/`csharp/scaffolding.md` specify.
- ❌ Touching existing source code for any reason other than the conditional scaffold check — only CLAUDE.md is unconditionally written.
- ❌ Defaulting to "all modules" instead of confirming the selection with the user first.
- ❌ Citing an ADR in the root CLAUDE.md's Standards table that the deployed rule files don't actually reference — derive citations from what Step 2 found, never invent one.
- ❌ Writing an "Existing Conventions" section with nothing observed in it, or a Convention Adjudication Table row with no real host-vs-standard conflict behind it.
- ❌ Regenerating an existing root or per-module CLAUDE.md wholesale instead of additively reconciling it.

## Validation

- [ ] `/rules` existed before any scaffolding or CLAUDE.md generation started
- [ ] Only the user-confirmed set of modules was touched
- [ ] No existing source file was modified — scaffolding only ran where a project file was missing
- [ ] Root CLAUDE.md has all 5 elements (overview, existing-conventions-if-any, ADR-cited standards + skill-first note, adjudication table, provenance footer) or additively reconciled an existing one
- [ ] Every ADR cited in the root CLAUDE.md's Standards table is one a deployed rule file actually references
- [ ] Per-module CLAUDE.md files point back to the root file's Standards section
- [ ] `dotnet build` passes for any newly scaffolded .NET projects
- [ ] Every `@` reference in every CLAUDE.md resolves to an actual deployed rule file

## Related skills

- **install-clean-arch-rules** — prerequisite; deploys the `/rules` this skill reads.
- **write-unit-tests** — fills in tests for the modules this skill scaffolds.
- **`ai-enablement`'s `bootstrap-dotnet-from-template` / `extend-dotnet-solution` / `scaffold-minimal`** (if installed) — the template-repo-centric alternative for a greenfield service cloned from `skillable-template-dotnet-react`; use that path instead when starting from the template rather than an existing repo structure.

## Notes for skill authors

The root CLAUDE.md's "Standards for New Code" table is only as accurate as Step 2's ADR grep — when a rule file's inline ADR citation changes, this skill's output changes automatically on the next run; it does not hardcode ADR IDs of its own. Keep it that way: never hardcode an ADR ID in this SKILL.md that duplicates what a rule file already cites, since the two would drift.

**Version history:**
- 1.1.0 — Adopted the `ai-enablement` `SKILL.md` house style. Redesigned the root CLAUDE.md to the `claude-md-brownfield.md` 5-section contract (ADR-cited standards, skill-first workflow note, convention adjudication table, provenance footer), adapted to what this skill can produce without a full codebase-survey/architect-mode pipeline. Removed Next.js as a module choice (the org's frontend ADR rejects it — see `csharp/modularization.md`).
- 1.0.0 — Initial release.

**Maintainer:** Jose Murillo Castro
