# CLI reference

All read commands support `--json`. All mutating commands support `--dry-run`. Commands that need a vault find it from `LOOSEENDS_VAULT`, then `config.toml` discovery, then `~/LooseEnds`.

## Vault and configuration

```text
looseends init [--vault PATH]          Create or connect to a vault
looseends config get KEY               Read a configuration value
looseends config set KEY VALUE         Write a configuration value
looseends doctor [--verbose]           Validate documents and derived state
```

`doctor` reports problems and repairs only derived state. It never rewrites canonical finding content.

## Projects

```text
looseends project add [PATH]           Register a repository
looseends project list                 List registered projects
looseends project inspect NAME         Show identity and scan health
looseends project remove NAME          Stop scanning; preserve its documents
```

Removing a project stops scanning it. Its findings remain in the vault.

## Scanning and reporting

```text
looseends scan [PROJECT]               Scan new sessions
  --dry-run                            Report candidate sessions; no model call, no writes
  --since DURATION                     Override the checkpoint for this run
  --model MODEL                        Override the configured analysis model
  --harness NAME                       Override the configured analysis harness
  --save-prompts DIR                   Write model inputs and outputs for local audit

looseends report                       Generate a Markdown report
  --since DURATION                     Window, e.g. 7d
  --project NAME                       Restrict to one project
  --status LIST                        Restrict to statuses
  --stdout                             Print the report instead of its path
```

## Findings

```text
looseends list                         Query findings
looseends show ID                      Print a canonical finding
looseends adopt ID                     Adopt a proposed finding
looseends defer ID --until DATE        Defer a finding
looseends dismiss ID --reason TEXT     Dismiss a finding
looseends resolve ID --reason TEXT     Mark a finding resolved
looseends reopen ID --reason TEXT      Reopen a terminal finding
looseends note ID --text TEXT          Append a dated working note
looseends link ID OTHER-ID             Record a relationship
```

Filters compose:

```sh
looseends list \
  --project my-app \
  --since 7d \
  --status proposed,adopted \
  --kind reliability,product-opportunity
```

To change a finding's title, summary, evidence, or recommendation, edit the Markdown file. That is the supported path in v1; see [integration.md](integration.md).

## Harness integration

```text
looseends harness list                 Show available and installed harnesses
looseends harness install NAME         Install the integration for a harness
looseends harness status [NAME]        Diagnose the integration
looseends harness uninstall NAME       Remove the integration
looseends mcp serve                    Run the local stdio MCP server
```

`claude-code` is the only harness available in v1.

## Scheduling

```text
looseends schedule install             Install user-level timers
  --scan SPEC                          e.g. every=3h
  --report SPEC                        e.g. daily@17:30
looseends schedule status              Show installed schedules and last runs
looseends schedule run scan|report     Run one now, as the scheduler would
looseends schedule uninstall           Remove installed timers
```

Loose Ends installs a systemd user timer on Linux or a launchd agent on macOS. There is no resident daemon.
