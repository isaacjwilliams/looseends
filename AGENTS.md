# Loose Ends repository instructions

## Sources of truth

- Treat [README.md](README.md) as the current user-facing product specification.
- Treat [ROADMAP.md](ROADMAP.md) as noncommittal exploration. Do not implement or advertise a roadmap item as current behavior unless the user explicitly promotes it into scope.
- When behavior changes, update the user-facing documentation in the same change. Do not let implementation silently redefine terminology or guarantees.

## Product invariants

These invariants apply to design, implementation, tests, and review:

1. **Markdown is canonical.** The complete accepted state must be reconstructable without a database, cache, report, or model call.
2. **The user owns the files.** Manual edits are supported and take precedence over stale generated changes. Preserve unknown frontmatter and user-authored sections.
3. **There are no silent decisions.** A scan may create a `proposed` finding and append an immutable source observation. It may not adopt, dismiss, defer, resolve, reopen, merge, split, or otherwise make a user decision.
4. **There are no silent rewrites.** Model-authored changes to existing curated finding content require a staged, reviewable change set and approval.
5. **Every claim has provenance.** Keep session evidence, repository evidence, inference, and recommendation distinguishable in storage and presentation.
6. **Every mutation is auditable.** Scan ingestion points to a run record. User decisions and curated-content changes retain applied, rejected, or direct one-operation change sets.
7. **Scans are idempotent.** Reprocessing the same source material cannot duplicate findings, sources, or evidence.
8. **Unattended capture is read-only.** Scheduled scanning and enrichment must not modify source repositories, execute project commands, connect to project services, install dependencies, or provision infrastructure.
9. **Interactive investigation inherits Codex.** A user-initiated investigation runs in the user's active Codex environment and permission model. Loose Ends records it but does not silently broaden access or provision resources.
10. **Derived state is disposable.** Reports, indexes, and caches must be rebuildable from canonical documents.
11. **Model output is untrusted input.** Deterministic Go code validates schemas, identifiers, state transitions, paths, revisions, and file operations.
12. **Finding approval and code approval stay separate.** Adopting or revising a finding never authorizes an implementation change.

## Implementation boundaries

- Implement the product in Go. Keep distribution compatible with a single compiled binary and do not introduce a JavaScript or TypeScript runtime.
- Do not add SQL or another opaque canonical store. A disposable index is acceptable only when deleting it loses no accepted state.
- Keep CLI and MCP behavior behind the same domain layer. Do not create a second set of mutation semantics for conversational use.
- Expose focused domain operations; do not give the model arbitrary vault file writes or generic database access.
- Keep scan schedules and report schedules independent.
- Use the configured Codex model for unattended analysis. An empty model setting inherits the current Codex default; do not hard-code a model as a permanent product default.
- Run scanner-created Codex sessions ephemerally so Loose Ends cannot ingest its own analysis sessions.
- Do not add Loose Ends-managed databases, containers, worktrees, or other investigation infrastructure unless the roadmap item is explicitly promoted.

## Terminology

- A **finding** is a durable record of a shortcoming or opportunity.
- A finding is **proposed** by scanning and **adopted** only by the user.
- An **investigation** records exploratory questions, experiments, evidence, and conclusions without replacing the finding.
- A **change set** is a versioned proposal to mutate one or more findings.
- **Adopt** changes a finding's lifecycle state. **Approve** applies a reviewed change set. Do not use the words interchangeably.
- A **report** is a generated view and never canonical state.

## Change discipline

- Reject stale writes by content revision; never overwrite a newer manual edit.
- For multi-file mutations, validate all preconditions before writing and provide recoverable journaled application.
- Prefer explicit uncertainty and open questions to unsupported enrichment.
- Keep scheduled failure behavior conservative: do not advance source checkpoints until resulting canonical writes succeed.
- Preserve dismissal and resolution decisions during reconciliation. New evidence may recommend reconsideration but cannot silently reopen a finding.
- Do not execute a code-changing workflow merely because a finding was adopted or a change set was approved.

## Verification

When the relevant code exists:

- Run `gofmt` on changed Go files.
- Run `go test ./...` after implementation changes.
- Add behavioral tests for every affected invariant, especially idempotency, stale revisions, interrupted transactions, manual Markdown edits, and scan-versus-investigation permission boundaries.
- Treat examples and commands in README.md as executable documentation. Update tests and documentation together when their contract changes.
