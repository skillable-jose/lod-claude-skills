# Command-Line Interface Design Standards

## Command & Subcommand Naming

- Commands are lowercase, single words: `init`, `deploy`, `migrate`
- Subcommands follow `topic command` (noun verb) pattern: `db migrate`, `user create`, `report export`
- Use kebab-case for multi-word names: `dry-run`, `output-format`
- Never use underscores; never use camelCase
- Be consistent within a product — don't mix styles

## Options vs Arguments

- **Prefer options (flags) over positional arguments** for anything optional — flags are self-documenting and order-independent
- Positional arguments reserved for the primary required operand (e.g., a file path, a name)
- Every option has a long form `--option-name`; common options also have a short form `-o`
- Standard short aliases: `-v` (verbose), `-q` (quiet), `-h` (help), `-o` (output)
- Options with values: `--output json` or `--output=json` (both forms valid)

## Standard Flags

Always wire on every CLI tool:

| Flag | Short | Purpose |
|---|---|---|
| `--help` | `-h` | Show help for the command |
| `--verbose` | `-v` | Increase output detail |
| `--quiet` | `-q` | Suppress non-error output |

Wire on the root command only:

| Flag | Purpose |
|---|---|
| `--version` | Show version |

Wire when applicable:

| Flag | When |
|---|---|
| `--json` | Tool produces structured data output |
| `--dry-run` | Tool has destructive or irreversible operations |

## Exit Codes

- `0` — success
- `1` — general failure (operation failed)
- `2` — misuse / bad arguments (wrong flags, missing required options)
- Never exit `0` on failure. Never exit non-zero on success.

## Output Conventions

- **stdout**: primary output, data results, machine-readable content
- **stderr**: errors, warnings, progress indicators, log messages
- When stdout is a TTY: colorize, use progress indicators, human-friendly formatting
- When stdout is piped: plain text, no ANSI escape codes, no progress spinners
- `--json` outputs valid JSON to stdout; no other text on stdout when active
- `--quiet` suppresses all stdout except data output; errors still go to stderr

## Help Text Standards

Each command's help must include:
1. One-line description (imperative mood: "Create a new user", not "Creates a new user")
2. Usage line: `myapp topic command [OPTIONS] [ARGUMENT]`
3. Description of all options with types and defaults
4. At least one usage example
5. Exit code documentation for non-obvious codes

## Error Messages

- Write all errors to stderr
- Format: `error: <what failed> (<context>).` — e.g., `error: file not found (path: /etc/config.json).`
- Always suggest a fix when possible: `Run 'myapp init' to create a config file.`
- On bad arguments: print the specific bad value, not just "invalid argument"

## Signal Handling

- Handle Ctrl+C (SIGINT) gracefully — cancel in-progress work, clean up temp files, exit with code `1`
- Long-running operations must be cancellable
- Print a message to stderr when cancelled: `Cancelled.`

## Scripting Compatibility

- No interactive prompts in non-TTY mode — fail with a clear error and instruct which flag to use
- All output parseable with standard tools (`grep`, `jq`, `awk`)
- Idempotent commands where possible
