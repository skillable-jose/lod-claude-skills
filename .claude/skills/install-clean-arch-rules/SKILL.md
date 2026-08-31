---
name: install-clean-arch-rules
description: Deploy the bundled clean-architecture coding/testing rule files (common, csharp, typescript) into a target repo's /rules folder, so bootstrap-clean-arch and per-module CLAUDE.md files have something to reference.
version: 1.1.0
applies_to: [dotnet, node]
references: []
authors: Jose Murillo Castro
last_updated: 2026-08-28
argument-hint: "[overwrite | skip]"
---

# Install Clean Architecture Rules

## What this does

Copies this skill's bundled `rules/` directory (common, csharp, typescript coding/testing conventions) to `<repo-root>/rules/`, via a plain shell copy — never a read-and-rewrite. It only deploys files; it does not generate `CLAUDE.md` or scaffold any project.

## When to use this

Run this first, before `bootstrap-clean-arch` or `write-unit-tests`, in any repo that doesn't yet have a `/rules` directory populated by this bundle.

**Do NOT use this for:**
- Generating or updating `CLAUDE.md` files — that's `bootstrap-clean-arch`.
- Scaffolding project structure — also `bootstrap-clean-arch`.
- Editing rule file *contents* — this skill copies them verbatim; edit the source files under this skill's own `rules/` directory instead, then re-run with `overwrite`.

## Inputs

| Input | Type | Notes |
|---|---|---|
| `$ARGUMENTS` | `"overwrite"` \| `"skip"` \| empty | Controls behavior when `<repo-root>/rules/` already has `common/`, `csharp/`, or `typescript/` content. If empty and rules already exist, ask the user rather than guessing. |

## Outputs

- `<repo-root>/rules/common/`, `<repo-root>/rules/csharp/`, `<repo-root>/rules/typescript/` populated with the bundled rule files.
- `<repo-root>/rules/skills/` (if present) is never touched — it's managed separately from this bundle.
- A summary table of files deployed per directory, with Created/Overwritten/Skipped status.

## How it works

### Step 1 — Locate Bundled Rules

Find the `rules/` subdirectory that is a sibling of this `SKILL.md` file (inside the skill's own directory). It contains:

```
rules/
├── common/
│   ├── coding-style.md
│   ├── command-line.md
│   ├── database.md
│   ├── logging.md
│   ├── patterns.md
│   ├── security.md
│   └── testing.md
├── csharp/
│   ├── coding-style.md
│   ├── command-line.md
│   ├── domain.md
│   ├── hosting.md
│   ├── modularization.md
│   ├── persistence.md
│   ├── presentation.md
│   ├── scaffolding.md
│   ├── security.md
│   ├── services.md
│   └── testing.md
└── typescript/
    ├── angular.md
    ├── coding-style.md
    ├── css.md
    ├── frontend-arch.md
    ├── patterns.md
    ├── react.md
    ├── security.md
    └── testing.md
```

### Step 2 — Check Target Location

Check whether a `rules/` directory already exists at the repository root (excluding `rules/skills/`).

- **If `$ARGUMENTS` is `overwrite`**: Replace all rule files in `<repo-root>/rules/` with the bundled versions. Preserve the `rules/skills/` directory — only overwrite `rules/common/`, `rules/csharp/`, and `rules/typescript/`.
- **If `$ARGUMENTS` is `skip`**: Keep existing rules as-is. Report what was skipped.
- **If `$ARGUMENTS` is empty or unrecognized and rules already exist**: Ask the user whether to **overwrite** or **skip**.
- **If no rules exist at the repo root**: Copy everything without prompting.

### Step 3 — Deploy Rules via Shell Copy

**CRITICAL: Use the Bash tool with `cp -rf`. Do NOT read files and re-write them one by one — that is slow and error-prone.**

`cp -rf` works on all platforms Claude Code runs on: macOS, Linux, and Windows (via Git Bash). Do not use `robocopy` — Git Bash converts `/`-prefixed flags (e.g. `/E`) into drive paths (`E:/`), breaking the command.

Resolve two absolute paths first:
- `$SKILL_DIR` — the directory containing this SKILL.md file
- `$REPO_ROOT` — the root of the target repository

Then run:

```bash
SRC="$SKILL_DIR/rules"
DST="$REPO_ROOT/rules"

mkdir -p "$DST"
cp -rf "$SRC/common"     "$DST/"
cp -rf "$SRC/csharp"     "$DST/"
cp -rf "$SRC/typescript" "$DST/"
```

**Skip mode:** Do not run the copy. Report existing files as skipped.

After the copy completes:
1. Do NOT touch `$REPO_ROOT/rules/skills/` — that directory is managed separately
2. Verify with `ls "$DST/common" "$DST/csharp" "$DST/typescript"` and confirm file counts match the manifest in Step 1

### Step 4 — Report

Provide a summary using the file counts returned by the verification listing:

| Directory | Files Deployed | Status |
|---|---|---|
| rules/common/ | (count) | Created / Overwritten / Skipped |
| rules/csharp/ | (count) | Created / Overwritten / Skipped |
| rules/typescript/ | (count) | Created / Overwritten / Skipped |

## Anti-patterns

- ❌ Reading each rule file and re-writing its content by hand instead of `cp -rf` — slow, and risks silently diverging from the bundled source.
- ❌ Overwriting `rules/skills/` — that directory belongs to a different, separately-managed mechanism.
- ❌ Deploying rules and then immediately generating CLAUDE.md content yourself instead of handing off to `bootstrap-clean-arch` — keep the two concerns separate.
- ❌ Treating this bundle as a substitute for the org's own ADR-backed standards (`ai-enablement`'s `_shared/csharp-coding-standards.md`, `engineering-decisions` ADRs) where the target repo already has those installed — this bundle's rule files are written to defer to and cite specific ADRs rather than compete with them; if a rule file and an ADR disagree, the ADR wins and the rule file has a bug that should be fixed at the source, not worked around per-repo.

## Validation

- [ ] `<repo-root>/rules/common/`, `csharp/`, `typescript/` exist and file counts match this skill's own `rules/` manifest
- [ ] `rules/skills/` (if it existed before) is untouched
- [ ] No rule file content was modified during deployment — a `diff -r` between the skill's bundled `rules/` and the deployed copy is empty
- [ ] Report table shown to the user with accurate Created/Overwritten/Skipped status per directory

## Related skills

- **bootstrap-clean-arch** — run after this skill; reads the deployed `/rules` to scaffold modules and generate `CLAUDE.md` files.
- **write-unit-tests** — also reads `/rules/csharp/testing.md` or `/rules/typescript/testing.md` (if deployed) as its highest-priority testing convention source.
- **ai-enablement's `bootstrap-dotnet-from-template` / `extend-dotnet-solution`** — a different, template-repo-centric scaffolding path for new services cloned from `skillable-template-dotnet-react`. Use this skill instead when the target repo already exists with its own structure and isn't being bootstrapped from that template.

## Notes for skill authors

The bundled rule files under `rules/` are meant to be read *by other skills* (`bootstrap-clean-arch`, `write-unit-tests`) as well as by a human working in the repo directly. When editing a bundled rule file, prefer citing a specific `engineering-decisions` ADR by ID over inventing a new convention — if no ADR covers the topic, say so explicitly in the rule file rather than presenting an invented convention as if it were settled.

**Version history:**
- 1.1.0 — Adopted the `ai-enablement` `SKILL.md` house style (frontmatter metadata, Inputs/Outputs/Anti-patterns/Validation/Related-skills sections) for consistency with skills this repo is installed alongside.
- 1.0.0 — Initial release.

**Maintainer:** Jose Murillo Castro
