# Harnesses

A *harness* is a coding agent that Loose Ends works with. Loose Ends uses a harness in two distinct roles:

- **Ingestion** — reading local session transcripts to find unresolved observations.
- **Analysis** — invoking a model headlessly to evaluate excerpts and produce structured output.

These roles are deliberately separable. Nothing in the design requires that the agent whose sessions you read is the agent that analyzes them.

Claude Code is the first supported harness and the only one implemented in v1. The contract below exists so that adding another is an adapter, not a redesign.

## The ingestion contract

An ingestion adapter answers a small set of questions about local session history:

| Capability | Description |
| --- | --- |
| Discover | Enumerate session transcripts on this machine |
| Associate | Map a session to a repository working directory |
| Order | Provide a stable, resumable cursor per session source |
| Read | Yield turns as structured records: role, text, tool name, tool result |
| Identify | Provide a stable session ID that a run record can name |

The adapter yields normalized turns. It does not decide what is interesting; excerpt selection and extraction are harness-independent and live in the scanner.

Adapters are versioned. A harness changing its on-disk format is an adapter version bump, not a change to finding semantics.

## The analysis contract

An analysis adapter runs a prompt headlessly and returns structured output:

| Capability | Description |
| --- | --- |
| Invoke | Run a prompt non-interactively and return JSON matching a supplied schema |
| Constrain | Restrict the run to read-only tools |
| Authenticate | Reuse the harness's own login without Loose Ends handling credentials |
| Identify | Report the session ID the invocation created, for the run record |

The last capability is required, not optional. It is what makes the anti-recursion invariant in [scanning.md](scanning.md) enforceable.

## The Claude Code adapter

**Ingestion.** Claude Code writes one JSONL transcript per session under `~/.claude/projects/<slugified-working-directory>/<session-id>.jsonl`, where the slug is the absolute path with separators replaced by hyphens. The adapter maps a registered repository path to its slug, enumerates transcripts, and reads turns from the JSONL records.

`CLAUDE_CONFIG_DIR` relocates that tree. The adapter honors it when set.

**Analysis.** Loose Ends invokes `claude` in headless mode with a JSON output schema and a read-only tool allowlist, and records the resulting session ID in the run record.

**Authentication.** The `claude` child process owns authentication entirely. Loose Ends never reads, parses, or copies a credential file, and never handles a token. If Claude Code is not authenticated, a scan fails before advancing any checkpoint.

## Installing a harness integration

```sh
looseends harness install claude-code
looseends harness status
looseends harness uninstall claude-code
```

For Claude Code this installs a plugin bundling the `/looseends` skill and the local MCP server; see [integration.md](integration.md).

## Configuration

v1 configures one analysis harness for the whole vault:

```toml
[analysis]
harness = "claude-code"
model = ""            # empty inherits the harness default
```

Ingestion in v1 uses the same harness. Per-project ingestion, multiple simultaneous ingestion harnesses, and an analysis harness distinct from the ingestion harness are described in [ROADMAP.md](../ROADMAP.md).
