# lod-claude-skills

Claude Code skills for clean-architecture .NET/TypeScript solutions (built against the `lab-on-demand` conventions).

## Skills

- **install-clean-arch-rules** — deploys the bundled coding/testing rules to `<repo-root>/rules/`. Run this first.
- **bootstrap-clean-arch** — scaffolds clean-architecture modules and generates per-module `CLAUDE.md` files from the deployed rules.
- **write-unit-tests** — plans and writes xUnit + Moq(Strict) unit tests for a class, following the one-method-deep / virtual-mock testing convention.