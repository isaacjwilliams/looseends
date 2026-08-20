# Contributing

Loose Ends is implemented in Go and builds as a single static-friendly executable.

```sh
go build ./cmd/looseends
go test ./...
gofmt -l .
```

Before changing behavior, read [AGENTS.md](AGENTS.md). It defines which document owns which kind of truth and lists the product invariants that apply to every change.

## Code layout

The code is organized around product boundaries, not technical layers:

```text
cmd/looseends/        CLI entry point
internal/vault/       Markdown parsing, validation, revisions, transactions
internal/findings/    Lifecycle and semantic operations
internal/harness/     Harness contract; ingestion and analysis adapters
internal/scan/        Excerpt selection, extraction, run records
internal/reconcile/   Deduplication and evidence attachment
internal/reports/     Disposable Markdown views
internal/mcp/         Structured tools for conversational use
internal/schedule/    launchd and systemd user integration
```

Two rules keep the boundaries honest:

- **The CLI and the MCP server share one domain layer.** Conversational operations are not a second implementation of vault behavior. If an operation exists in both, it is the same function.
- **Harness details stay inside `internal/harness/`.** The scanner, the vault, and finding semantics must not know which agent produced a session. See [docs/harnesses.md](docs/harnesses.md).

The vault layer is model-agnostic and deterministic. Model output enters through validation and never reaches the filesystem unvalidated.

## Testing

Documentation is the behavioral contract. Examples and guarantees in [README.md](README.md) and [docs/](docs/) should be reflected in tests.

Give particular attention to:

- Idempotent rescanning of the same session
- Stale-revision rejection after a manual Markdown edit
- Interrupted transactions and journal recovery
- Preservation of unknown frontmatter and user-authored sections
- The anti-recursion invariant: a session created by a scan is never ingested
- The read-only boundary of unattended scanning

Tests should not require network access or a real model. Harness adapters are interfaces; analysis is exercised against recorded fixtures.
