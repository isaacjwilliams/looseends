# Scanning

A scan reads new agent sessions for registered projects, extracts observations that were never resolved, reconciles them against existing memory, and writes findings. It is the only unattended path that writes to the vault.

Scanning is read-only with respect to your source repositories. It never modifies code, runs project commands, connects to project services, installs dependencies, or provisions infrastructure.

## 1. Select source sessions

Loose Ends reads local session transcripts through a harness adapter (see [harnesses.md](harnesses.md)). It considers only sessions associated with registered repositories, and only content newer than that source's checkpoint.

Selection is conservative:

- Sessions produced by Loose Ends itself are excluded. See the anti-recursion invariant below.
- Sessions older than `scan.max_session_age` are ignored.
- Repositories, paths, sessions, and content patterns can be excluded by configuration.
- A session is not marked processed until the findings derived from it have been written successfully.

`looseends scan --dry-run` reports which session ranges would be considered without making a model call or touching the vault.

## The anti-recursion invariant

Loose Ends invokes a coding agent, and that agent's own sessions are exactly the kind of material Loose Ends ingests. Without a defense, findings would compound into findings about findings.

The guarantee is enforced in the vault, not in the harness:

> Every scan run writes a run record naming the session IDs it created. Ingestion never reads a session named by any run record, and never reads a transcript located inside the vault.

This is deterministic, requires no content inspection, is testable without invoking a model, and holds for every harness — including harnesses that provide no isolation mechanism of their own.

Harness-level isolation is welcome as defense in depth where a harness supports it, but no correctness guarantee may rest on it. See [ROADMAP.md](../ROADMAP.md) for the isolated Claude Code home under exploration.

## 2. Extract unresolved observations

A local parser builds bounded excerpts around signals such as an identified bug, a workaround, a missing invariant, product friction, an architectural weakness, or an explicit decision to handle something later. The model then evaluates those excerpts against a strict structured-output schema.

An observation is excluded when the same session shows it was:

- Fixed and verified
- Deliberately accepted as intended behavior
- Disproved during the session
- Already captured through Loose Ends
- Merely hypothetical, with no project-specific basis

Extraction separates evidence from recommendation. Opinionated product ideas are welcome but are labeled as proposals and never presented as observed fact.

Model output is untrusted input. Deterministic Go code validates schemas, identifiers, enumerations, paths, and every proposed file operation before anything is written.

## 3. Reconcile with memory

Candidates are compared against active, resolved, and dismissed findings. Reconciliation may:

- Append a source or new evidence to an existing finding
- Link related but distinct findings
- Create a genuinely new `proposed` finding
- Suppress a duplicate of a dismissed idea

Reconciliation never changes a status. A dismissed finding stays dismissed; repeated evidence is absorbed silently. A genuine regression is created as a new finding linked to the resolved one.

Scans are idempotent. Reprocessing the same session cannot duplicate a finding, a source, or an evidence item.

## 4. Write and report

New proposals and appended sources are written. Existing narrative and user decisions are never touched by a scan. Reports are regenerated from current Markdown state and can be deleted and rebuilt at any time.

## Run records

Each run writes a concise Markdown record to `.looseends/runs/` containing its inputs, the sessions it read, the sessions it created, counts, warnings, token usage when available, and the resulting finding IDs.

Run records serve two purposes: they are the audit trail for scan-created content, and they are the mechanism that makes the anti-recursion invariant enforceable.

## Checkpoints and failure

Checkpoints advance only after the canonical writes derived from a session succeed. If a scan fails partway, the next run reprocesses from the last safe cursor; idempotent reconciliation makes that harmless. Failure behavior is detailed in [trust.md](trust.md).
