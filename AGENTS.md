# Loose Ends repository instructions

## Sources of truth

This repository practices README-driven development. Each document below owns one kind of truth. Keep them separate.

- [README.md](README.md) is the **product pitch and user interface**: what Loose Ends is for, what the user types, what they get back. Keep it short. Do not add schemas, file formats, internal mechanics, exhaustive flag lists, or package layouts to it.
- [docs/](docs/) is the **specification**. Behavior, formats, contracts, and guarantees are defined here and are the authority for implementation and tests.
- [ROADMAP.md](ROADMAP.md) is **noncommittal exploration**. Do not implement or advertise a roadmap item as current behavior unless the user explicitly promotes it into scope.
- [CONTRIBUTING.md](CONTRIBUTING.md) covers building, testing, and code layout.

When behavior changes, update the specification in `docs/` in the same change, and update the README only if the user-visible interface changed. Do not let implementation silently redefine terminology or guarantees.

When a change seems to require explaining a mechanism in the README, reconsider the mechanism's visibility rather than adding a README section.

## Product invariants

The authoritative statement is [docs/trust.md](docs/trust.md). It applies to design, implementation, tests, and review. In summary:

1. **Markdown is canonical.** Accepted state is reconstructable without a database, cache, report, or model call.
2. **The user owns the files.** Manual edits take precedence over any generated write. Preserve unknown frontmatter and user-authored sections.
3. **There are no silent decisions.** A scan may create a `proposed` finding and append an immutable source observation. It may not make a user decision.
4. **There are no silent rewrites.** No code path may rewrite curated finding content. Do not add one; the governed version of this capability is a roadmap item.
5. **Every claim has provenance.** Keep session evidence, repository evidence, inference, and recommendation distinguishable in storage and in presentation.
6. **Every mutation is auditable.** Scan ingestion points to a run record; user decisions record reason and time.
7. **Scans are idempotent.** Reprocessing the same source material cannot duplicate findings, sources, or evidence.
8. **Unattended capture is read-only.** Scheduled scanning must not modify source repositories, execute project commands, connect to project services, install dependencies, or provision infrastructure.
9. **Derived state is disposable.** Reports, indexes, and caches must be rebuildable from canonical documents.
10. **Model output is untrusted input.** Deterministic Go code validates schemas, identifiers, state transitions, paths, revisions, and file operations.
11. **Finding approval and code approval stay separate.** Adopting a finding never authorizes an implementation change.
12. **Loose Ends does not ingest itself.** Enforce this in the vault layer via run records, never solely through a harness mechanism.

## Implementation boundaries

- Implement in Go. Keep distribution compatible with a single compiled binary and do not introduce a JavaScript or TypeScript runtime.
- Do not add SQL or another opaque canonical store. A disposable index is acceptable only when deleting it loses no accepted state.
- Keep CLI and MCP behavior behind the same domain layer. Do not create a second set of mutation semantics for conversational use.
- Expose focused domain operations. Never give a model arbitrary vault file writes or generic database access.
- Keep harness-specific code behind the adapter contract in [docs/harnesses.md](docs/harnesses.md). Harness details must not leak into the scanner, the vault, or finding semantics. Do not treat Claude Code as the assumed harness.
- Anti-recursion belongs to the vault layer. A harness may add isolation as defense in depth, but no correctness guarantee may depend on it.
- Use the configured analysis model. An empty setting inherits the harness default; do not hard-code a model as a permanent product default.
- Do not add Loose Ends-managed databases, containers, worktrees, or other investigation infrastructure.

## Scope discipline

v1 scope is: register repositories, scan sessions, propose findings as Markdown, reconcile idempotently, report, and let the user decide via the CLI or the `/looseends` skill. Treat anything beyond this as out of scope.

Change sets, merge and split, investigation documents, unattended enrichment, and multi-harness configuration are in [ROADMAP.md](ROADMAP.md). Do not implement them incidentally, and do not add a "small" version of one as a stepping stone without an explicit decision.

## Terminology

- A **finding** is a durable record of a shortcoming or opportunity.
- A finding is **proposed** by scanning and **adopted** only by the user.
- A **source** is an append-only record of a session that observed a finding.
- A **note** is an append-only dated entry under a finding's `## Notes` section.
- A **run record** documents one scan or report execution, including the sessions it created.
- A **report** is a generated view and never canonical state.
- A **harness** is a coding agent, used for ingestion, analysis, or both.

Reserve **approve** for the roadmap's change-set mechanism. Do not use it as a synonym for **adopt**.

## Change discipline

- Reject stale writes by content revision; never overwrite a newer manual edit.
- For multi-file mutations, validate all preconditions before writing and provide recoverable journaled application.
- Prefer explicit uncertainty and open questions to unsupported claims.
- Do not advance source checkpoints until the resulting canonical writes succeed.
- Preserve dismissal and resolution decisions during reconciliation.
- Do not execute a code-changing workflow merely because a finding was adopted.

## Verification

When the relevant code exists:

- Run `gofmt` on changed Go files.
- Run `go test ./...` after implementation changes.
- Add behavioral tests for every affected invariant, especially idempotency, stale revisions, interrupted transactions, manual Markdown edits, anti-recursion, and the read-only boundary of unattended scanning.
- Treat examples and commands in README.md and `docs/` as executable documentation. Update tests and documentation together when their contract changes.
