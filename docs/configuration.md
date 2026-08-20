# Configuration

`config.toml` lives at the root of the vault. It is intentionally small, portable, and never written by a scan. Project registration and finding state do not live here; they live with the documents in the vault.

```toml
version = 1
timezone = "America/New_York"

[analysis]
# The harness Loose Ends invokes for unattended analysis.
harness = "claude-code"
# Empty means: use the harness's current default model.
model = ""

[scan]
max_session_age = "30d"
max_excerpt_bytes = 120000

[reports]
schedule = "daily@17:30"

[retention]
cache = "30d"
runs = "forever"
```

## Keys

| Key | Default | Meaning |
| --- | --- | --- |
| `timezone` | system | Used for report windows and schedule times |
| `analysis.harness` | `claude-code` | Which harness runs unattended analysis |
| `analysis.model` | `""` | Empty inherits the harness default, so model upgrades need no edit |
| `scan.max_session_age` | `30d` | Sessions older than this are never considered |
| `scan.max_excerpt_bytes` | `120000` | Tool output larger than this is summarized before analysis |
| `reports.schedule` | unset | Cadence used by `schedule install` |
| `retention.cache` | `30d` | How long disposable cache entries are kept |
| `retention.runs` | `forever` | Run records are the scan audit trail |

Read and write values with the CLI rather than editing by hand when convenient:

```sh
looseends config set analysis.model MODEL
looseends config get analysis.model
```

## Exclusions

A repository-level `.looseendsignore` excludes paths from repository inspection, using gitignore syntax:

```gitignore
fixtures/customer-data/**
vendor/**
tmp/**
```

Vault-level exclusions can omit whole repositories, individual sessions, or text patterns from scanning. Exclusion is applied before any model invocation, so excluded material is never sent anywhere.

## Environment

| Variable | Effect |
| --- | --- |
| `LOOSEENDS_VAULT` | Vault path; takes precedence over discovery |
| `CLAUDE_CONFIG_DIR` | Honored by the Claude Code adapter when locating session transcripts |
