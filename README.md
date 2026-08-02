# Loose Ends

**Ambient issue capture for Codex.**

Loose Ends turns the unfinished observations buried in Codex sessions into a durable, reviewable backlog. It notices shortcomings, risks, cleanup opportunities, and product ideas that came up while you were working but were never resolved. It researches them just enough to be useful, writes them to plain Markdown, and brings the important ones back when you have time to act on them.

You do not maintain a second TODO list while coding. You keep working. Loose Ends remembers what was left behind.

```text
Codex sessions ──▶ unresolved observations ──▶ Markdown findings ──▶ reports
                                                       ▲                │
                                                       │                ▼
                                                       └──── Codex review and investigation
```

Loose Ends is local-first, single-user software. It is distributed as one Go binary, uses your existing Codex login, runs without a database or resident daemon, and never changes a source repository during a scheduled scan.

## Quick start

### Requirements

- macOS or Linux
- Codex CLI installed and available as `codex`
- A ChatGPT or API-key login configured through Codex
- Git repositories for the projects you want to monitor

Loose Ends delegates model work to `codex exec`, which [reuses your saved Codex authentication](https://learn.chatgpt.com/docs/non-interactive-mode). A ChatGPT subscription login works; a separate OpenAI API key is not required.

```sh
codex login
codex login status
```

Loose Ends never opens or copies Codex's credential store. The `codex` child process owns authentication.

### Install

With Homebrew:

```sh
brew install looseends
```

Release archives contain a single `looseends` binary and require no language runtime. To build the current checkout instead:

```sh
go install ./cmd/looseends
```

### Initialize a vault

```sh
looseends init --vault ~/LooseEnds
```

The vault is the tool's memory. It contains ordinary Markdown files and can live anywhere you choose. `~/LooseEnds` is only the suggested location.

Register a project from its repository:

```sh
cd ~/Code/my-app
looseends project add .
```

Registration records the repository path and identity in the vault. It does not add files to the repository.

### Choose the scan model

By default, Loose Ends uses the current default model configured for Codex. You can pin a different Codex model for every scan:

```sh
looseends config set analysis.model <MODEL>
```

Or override it for one run:

```sh
looseends scan --model <MODEL>
```

Scheduled scans use the configured model. Leaving `analysis.model` empty follows the Codex default, so Loose Ends picks up model changes without a configuration edit. Model availability and usage limits come from the account and workspace used by `codex login`.

Run the first scan and generate a report:

```sh
looseends scan
looseends report --since 7d
```

The second command prints the report path. Pass `--stdout` to print the report itself.

### Connect Loose Ends to Codex

```sh
looseends codex install
looseends codex status
```

This installs the `$looseends` user skill and registers `looseends mcp serve` as a local stdio MCP server. Codex supports both [user-level skills](https://learn.chatgpt.com/docs/build-skills) and [local MCP servers](https://learn.chatgpt.com/docs/extend/mcp).

Start Codex in any registered project and ask naturally:

```text
Use $looseends to show me findings from the last week.
```

### Schedule scans and reports

Scanning and reporting have independent schedules:

```sh
looseends schedule install \
  --scan every=3h \
  --report daily@17:30
```

Loose Ends installs a user-level systemd timer on Linux or launchd agent on macOS. There is no always-running Loose Ends daemon. Missed scans resume from their last source checkpoint the next time the command runs.

```sh
looseends schedule status
looseends schedule run scan
looseends schedule run report
looseends schedule uninstall
```

## The workflow

Loose Ends is designed for two different modes of attention.

During the day, it works in the background:

1. Read new Codex sessions associated with registered projects.
2. Identify observations that were neither fixed nor intentionally rejected in those sessions.
3. Reconcile them with existing findings so repeated observations do not become repeated TODOs.
4. Add provenance, evidence, confidence, and an opinionated proposed direction.
5. Generate a report without changing the state of any existing finding.

When you have time to review, Codex becomes the interface:

```text
You:    Use $looseends to show findings from the last week.

Codex:  There are six new findings and two older findings with new evidence.
        The highest-impact items are LE-0042 and LE-0047.

You:    Dismiss LE-0044 as intentional behavior. Adopt LE-0042.
        Investigate LE-0047 and the two retry-related findings.

Codex:  Dismissed LE-0044 and adopted LE-0042. I investigated the other
        three at the current repository revision and staged CHG-0018:

        - Refine LE-0047 from "missing timeout" to "unbounded retry latency."
        - Merge LE-0039 into LE-0047 as supporting evidence.
        - Split LE-0048 into a reliability finding and an observability finding.

        No source files were changed. Would you like to see the full diff?

You:    Show me.

Codex:  [Shows the semantic and Markdown diff.]

You:    Approve CHG-0018.

Codex:  Applied. The affected findings now reference INV-0023, and the
        next report will reflect the new state.
```

The conversation is convenient, but it is not the memory. The Markdown vault is.

## Findings are proposals, not automatically accepted TODOs

A model noticing something does not make it true or worth doing. New findings begin as `proposed`. They become part of your intentional backlog only when you adopt them.

| Status | Meaning |
| --- | --- |
| `proposed` | Loose Ends found something that may deserve attention. |
| `adopted` | You agree it belongs in the backlog. |
| `deferred` | Valid, but intentionally postponed until a condition or date. |
| `resolved` | Addressed or otherwise no longer present. |
| `dismissed` | Incorrect, duplicate, intentional, or not worth pursuing. A reason is required. |
| `superseded` | Replaced by another finding after a merge or split. |

Investigation is separate from status. A proposed, adopted, or deferred finding can have an active investigation without being forced into an `investigating` status.

Two verbs have deliberately different meanings:

- **Adopt** accepts a finding into the backlog.
- **Approve** applies a staged change set to one or more findings.

Loose Ends does not silently reopen dismissed or resolved findings. Materially new evidence can produce a report entry recommending reconsideration, but only the user can reopen the original finding. A genuine regression becomes a new finding linked to the resolved one.

## The Markdown vault

There is no SQL database. Canonical memory is stored in Markdown with small, typed YAML frontmatter blocks.

```text
~/LooseEnds/
├── README.md
├── config.toml
├── projects/
│   └── my-app/
│       ├── project.md
│       ├── findings/
│       │   ├── LE-0042.md
│       │   └── LE-0047.md
│       ├── investigations/
│       │   └── LE-0047/
│       │       └── INV-0023-unbounded-retries.md
│       ├── changes/
│       │   ├── pending/
│       │   ├── applied/
│       │   └── rejected/
│       └── reports/
│           ├── daily/
│           └── weekly/
├── reports/
│   └── weekly/
└── .looseends/
    ├── checkpoints.json
    ├── lock
    └── cache/
```

The distinction is intentional:

- Findings, investigations, change sets, projects, and reports are readable documents.
- `config.toml` is explicit user configuration.
- `.looseends/checkpoints.json` contains replaceable source cursors and machine identity.
- `.looseends/cache/` contains disposable indexes and model-run artifacts. Removing it never loses canonical state.

You can put the vault in a private Git repository for history and synchronization. Loose Ends does not require Git for the vault and does not push or pull it for you.

### A finding

```md
---
id: LE-0047
title: Retries can extend request latency without a fixed bound
project: my-app
status: adopted
kind: reliability
impact: high
confidence: high
created_at: 2026-08-01T14:22:08-04:00
updated_at: 2026-08-02T16:41:19-04:00
sources:
  - session: 019c...
    observed_at: 2026-08-01T14:17:02-04:00
    repository_revision: 92af461
investigations:
  - INV-0023
related:
  - LE-0039
---

# Retries can extend request latency without a fixed bound

## Summary

The HTTP client limits individual attempts but does not impose a deadline on
the retry sequence as a whole. A degraded dependency can therefore occupy a
request worker much longer than the caller expects.

## Why it matters

This converts a dependency failure into elevated application latency and can
amplify worker exhaustion during an incident.

## Evidence

- `internal/client/retry.go` starts each attempt with a fresh timeout.
- INV-0023 reproduced a 47-second request with a nominal 10-second timeout.

## Recommended direction

Carry one deadline through the complete retry sequence. Stop retrying when the
remaining budget cannot accommodate the next backoff and attempt.

## Alternatives

- Reduce the retry count. Simpler, but still does not define a latency bound.
- Remove retries. Predictable, but gives up useful transient-failure recovery.

## Open questions

- Should callers be able to supply a smaller total retry budget?
```

The revision used for concurrency checks is a SHA-256 hash of the complete file; it is calculated rather than embedded in the file. Unknown frontmatter fields and user-authored sections are preserved by semantic updates.

Scanner-owned source references and evidence observations are append-only. A scan may record that another session independently supports a finding, but it cannot rewrite the title, summary, impact, recommendation, status, relationships, or user-authored text. Those changes require a change set.

### An investigation

Investigations hold the verbose, evolving work that would make a finding noisy. They are resumable across Codex sessions and remain useful even when their proposed changes are rejected.

```md
---
id: INV-0023
finding: LE-0047
status: completed
finding_base_revision: sha256:da9c...
repository_revision: 92af461
branch: retry-cleanup
codex_session: 019d...
started_at: 2026-08-02T15:03:11-04:00
completed_at: 2026-08-02T16:36:40-04:00
---

# Investigation: total retry latency

## Questions
...

## Experiments
...

## Evidence
...

## Competing explanations
...

## Conclusion
...

## Proposed changes
...
```

Codex checkpoints an investigation after material experiments and before ending an investigation turn. It does not save every intermediate thought. If a session stops unexpectedly, the last coherent checkpoint remains available to resume.

### A change set

Every user decision and every modification to existing curated finding content has a change-set audit record, including simple commands issued through Codex or the CLI. New `proposed` findings and append-only source observations instead point to the scan run that ingested them.

A change set records:

- The exact semantic operations requested
- The base revision of every affected finding
- The investigation and repository revision that support the change
- A human-readable semantic diff
- Its author, creation time, approval time, and final disposition

Applying a multi-finding change set is transactional. Loose Ends validates all base revisions before writing any finding, journals the operation, replaces files atomically, and can finish or roll back an interrupted application on the next run.

Applied and rejected change sets are retained. Pending change sets can be abandoned, but are archived rather than erased.

## Preventing drift

An investigation should not quietly diverge from the finding it is meant to improve. Loose Ends uses a two-phase workflow:

```text
finding@rev7
    └──▶ investigation@repository-SHA
             └──▶ CHG-0018(base=rev7)
                       └──▶ review diff
                                  └──▶ apply ──▶ finding@rev8
```

Codex may inspect code and develop a recommendation freely. It may also checkpoint evidence into an investigation. It may not replace canonical finding content with model-authored text until it has staged a change set and the user has approved that change set.

At apply time:

1. Every finding must still match the base revision in the change set.
2. Every referenced investigation must still exist and be valid.
3. Repository drift is displayed. Evidence that depends on changed code must be revalidated.
4. The complete change is shown as both a semantic diff and a Markdown diff.
5. All operations apply together or none do.

If a finding was edited manually after the investigation, application fails as stale. Codex can rebase the proposal onto the new document, but the rebased diff requires review again. Manual edits always win by becoming the new base revision.

A direct and fully specified instruction counts as approval for exactly that operation:

```text
Dismiss LE-0044 as intentional behavior.
```

Loose Ends still records a one-operation change set. Broader instructions such as “clean this finding up,” model-authored conclusions, merges, and splits are always staged for review first. Codex's MCP approval policy may add a host-level confirmation for writes; that confirmation complements the vault change set rather than replacing it.

## How scanning works

### 1. Select source sessions

Loose Ends reads local Codex CLI session history under the configured `CODEX_HOME`. It considers only sessions associated with registered repositories and only content newer than that source's checkpoint.

Session selection is conservative:

- Loose Ends analysis sessions are run with `codex exec --ephemeral` and cannot recursively become findings.
- Sessions generated by reports or Loose Ends MCP operations are excluded.
- Tool output larger than configured limits is summarized before analysis.
- Repositories, paths, sessions, and content patterns can be excluded.
- A session is not marked processed until its findings have been safely written.

Use `looseends scan --dry-run` to see which session ranges would be considered without invoking Codex or changing the vault.

### 2. Extract unresolved observations

The local parser builds bounded excerpts around signals such as an identified bug, workaround, missing invariant, product friction, architectural weakness, or explicit “later” decision. Codex then evaluates those excerpts with a strict structured-output schema.

An observation is excluded when the same session shows that it was:

- Fixed and verified
- Deliberately accepted as intended behavior
- Disproved during investigation
- Already captured through Loose Ends
- Merely a hypothetical possibility with no project-specific basis

The extractor separates evidence from recommendation. Opinionated product ideas are welcome, but are labeled as proposals and never presented as observed fact.

### 3. Reconcile with memory

Candidates are compared with active, resolved, dismissed, and superseded findings. Reconciliation can:

- Attach a source or new evidence to an existing finding
- Recommend reopening without changing status
- Link related but distinct findings
- Create a genuinely new finding
- Suppress a duplicate of a dismissed idea

The scan is idempotent. Reprocessing the same session does not duplicate findings or evidence.

### 4. Enrich useful findings

For sufficiently supported candidates, a scheduled scan may inspect repository files in [Codex's read-only sandbox](https://learn.chatgpt.com/docs/sandboxing) to clarify impact and produce a concrete recommended direction. This is static enrichment: it does not run tests or project commands, write diagnostic code, connect to a database, start containers, or provision services.

Enrichment is bounded. Loose Ends is trying to preserve a useful lead, not perform an autonomous multi-hour implementation project. Uncertainty and open questions are preferable to invented certainty.

### 5. Investigate interactively

When the user asks Codex to investigate a finding, the investigation runs in that active Codex session—not in a special Loose Ends sandbox. It inherits the session's current repository, worktree, sandbox, approval policy, and access to host services.

That means an interactive investigation can run a Rails test suite against the developer's existing Postgres service, or write and run a focused test, when the active Codex permissions allow it. Codex asks for approval when an action crosses the active sandbox boundary. Loose Ends only creates checkpoints, records commands and results, and stages resulting changes to the findings vault.

Loose Ends does not provision an investigation database, container stack, worktree, or other isolated resources. Managed disposable investigation environments are an exploratory feature described in [ROADMAP.md](ROADMAP.md#managed-investigation-environments), not part of the current product contract.

### 6. Write and report

New proposals and append-only source observations are written to the vault. Existing narrative and user decisions are never changed by a scan. Reports are regenerated from current Markdown state and can be deleted and rebuilt at any time.

## Reports

Reports are inboxes, not databases. They contain links to canonical findings and summarize what deserves attention.

```sh
looseends report daily
looseends report weekly
looseends report --project my-app --since 7d
looseends report --status proposed,adopted --stdout
```

A report includes:

- New proposed findings
- Existing findings with material new evidence
- High-impact adopted findings that remain untouched
- Draft investigations that may be worth resuming
- Pending change sets awaiting approval
- Dismissed or resolved findings for which new evidence suggests reconsideration
- Scan failures or projects that could not be inspected

Report generation never changes finding status. Reading a report never marks an item accepted, dismissed, or resolved.

## CLI reference

All read commands support `--json`. Mutation commands support `--dry-run`, and commands that create a change set print its stable ID.

```text
looseends init                         Create or connect to a vault
looseends config                       Inspect or edit explicit configuration

looseends project add [PATH]           Register a repository
looseends project list                 List registered projects
looseends project inspect NAME         Show identity and scan health
looseends project remove NAME          Stop scanning; preserve its documents

looseends scan [PROJECT]               Scan new Codex sessions
looseends runs                         Show scan and report runs
looseends report [daily|weekly]        Generate a Markdown report

looseends list                         Query findings
looseends show ID                      Print a canonical finding
looseends adopt ID                     Adopt a proposed finding
looseends defer ID --until DATE        Defer a finding
looseends dismiss ID --reason TEXT     Dismiss a finding
looseends resolve ID --reason TEXT     Mark a finding resolved
looseends reopen ID --reason TEXT      Reopen a terminal finding

looseends investigation start ID       Start or resume an investigation
looseends investigation show INV-ID    Show its latest checkpoint
looseends investigation complete ID    Complete an investigation

looseends changes list                 List pending change sets
looseends changes show CHG-ID           Show operations and provenance
looseends changes diff CHG-ID           Show semantic and Markdown diffs
looseends changes apply CHG-ID          Validate and apply
looseends changes reject CHG-ID         Reject with a reason

looseends codex install                Install the skill and MCP integration
looseends codex status                 Diagnose the integration
looseends mcp serve                    Run the local stdio MCP server
looseends schedule ...                 Manage native user schedules
looseends doctor                       Validate and repair derived state
```

Filters compose:

```sh
looseends list \
  --project my-app \
  --since 7d \
  --status proposed,adopted \
  --kind reliability,product-opportunity
```

## Codex integration

The integration has two parts backed by the same Go application logic as the CLI.

### The `$looseends` skill

The skill teaches Codex the workflow and language of the product. In particular, it requires Codex to:

- Resolve the current repository to a registered project before querying
- Distinguish evidence, inference, and recommendation
- Use investigations for exploratory work
- Use domain operations instead of editing vault files directly
- Stage model-authored changes and show a diff
- Interpret “adopt” and “approve” unambiguously
- Recheck stale evidence before rebasing a proposal

The skill can be invoked explicitly with `$looseends`; Codex may also select it when a request clearly concerns findings or Loose Ends.

### The local MCP server

The MCP server exposes focused tools rather than arbitrary filesystem access.

Read-only tools:

- `list_findings`
- `get_finding`
- `get_investigation`
- `get_change_set`
- `diff_change_set`

Investigation tools:

- `begin_investigation`
- `checkpoint_investigation`
- `complete_investigation`

Mutation tools:

- `stage_change_set`
- `apply_change_set`
- `reject_change_set`
- `apply_explicit_transition`

Mutating tools advertise themselves as writes. `apply_change_set` and terminal status transitions require approval under the default Codex MCP policy. There is no `write_file`, generic SQL, or unrestricted `update_finding` tool.

## Configuration

`~/LooseEnds/config.toml` is intentionally small and portable:

```toml
version = 1
timezone = "America/New_York"

[analysis]
backend = "codex"
command = "codex"
# Applies to unattended scanning. Interactive investigations inherit the
# active Codex session's permissions.
sandbox = "read-only"
# Empty means: use the current Codex default.
model = ""

[scan]
max_session_age = "30d"
max_excerpt_bytes = 120000
enrich = true

[reports]
daily_time = "17:30"
weekly_day = "friday"

[retention]
cache = "30d"
applied_changes = "forever"
rejected_changes = "forever"
```

Project registration and finding state do not live in this file. They remain with the project documents in the vault.

Repository-level `.looseendsignore` files can exclude paths from enrichment:

```gitignore
fixtures/customer-data/**
vendor/**
tmp/**
```

Global exclusions can omit repositories, Codex sessions, or text patterns from scanning. Exclusion is applied before model invocation.

## Privacy and safety

Loose Ends reads conversation history, so its privacy boundary should be explicit.

- Only registered projects are scanned.
- `looseends scan --dry-run` performs no model calls.
- Model calls go through the configured Codex CLI and inherit that login's workspace policies and data handling.
- Relevant excerpts and repository context may be sent to the model. Do not register projects whose Codex history must not be processed this way.
- Authentication files and tokens are never read by Loose Ends.
- Scheduled scans use Codex's read-only sandbox.
- Scans never modify source repositories, execute project build scripts, or install dependencies.
- User-initiated investigations inherit the active Codex session's permissions and may run project commands or make diagnostic edits with the user's approval.
- Loose Ends does not provision databases, containers, credentials, or isolated compute for investigations.
- Vault writes use restrictive user-only permissions by default.
- Findings never trigger code changes, issue creation, messages, or external side effects on their own.

Run `looseends scan --save-prompts DIR` when you need an exact, local audit of model inputs and structured outputs. These audit bundles can contain sensitive project material and are not retained by default.

## Failure behavior

Loose Ends prefers an incomplete report over corrupt or misleading memory.

- If Codex is not authenticated, the scan fails before advancing checkpoints.
- If structured model output is invalid, the affected excerpt is quarantined for retry.
- If a repository moved, it is reported as unavailable rather than silently remapped.
- If a finding has invalid frontmatter, scans leave it untouched and `doctor` explains the problem.
- If an apply operation is stale, no affected finding is changed.
- If the machine stops during a vault transaction, the journal is recovered on the next command.
- If the optional cache is corrupt, it is deleted and rebuilt from Markdown.

Each scheduled run writes a concise Markdown run record containing its inputs, counts, warnings, Codex session ID, token usage when available, and resulting finding IDs.

## Trust contract

Loose Ends makes a small set of promises that should be visible without understanding its implementation:

- Your Markdown files are canonical, editable, and sufficient to reconstruct accepted state.
- Scans can propose findings and append sourced observations; they cannot make user decisions or rewrite curated content.
- Every claim distinguishes evidence, inference, and recommendation, and every decision is auditable.
- Unattended capture is read-only with respect to source repositories.
- Reports and indexes are derived, disposable views.

The complete engineering form of these invariants lives in [AGENTS.md](AGENTS.md), where it applies directly to implementation work and tests.

## What Loose Ends is not

- It is not a replacement for an issue tracker. Adopted findings can inform one, but Loose Ends captures ideas before they deserve team-visible tickets.
- It is not a transcript summarizer. Most sessions should produce no findings.
- It is not an autonomous repair bot. It never fixes findings merely because it discovered them.
- It is not a hidden memory service. There is no hosted Loose Ends account and no opaque server-side state.
- It is not a general personal knowledge base. Its schema and workflows are deliberately specific to software shortcomings and improvements.

## Development

Loose Ends is implemented in Go and builds as a single static-friendly executable. The CLI and MCP server share the same domain layer; conversational operations are not a second implementation of vault behavior.

```sh
go test ./...
go build ./cmd/looseends
```

The code is organized around product boundaries:

```text
cmd/looseends/        CLI entry point
internal/vault/       Markdown parsing, validation, revisions, and transactions
internal/findings/    Lifecycle and semantic operations
internal/sources/     Versioned conversation-history adapters
internal/analysis/    codex exec invocation and structured outputs
internal/reconcile/   Deduplication and evidence attachment
internal/reports/     Disposable Markdown views
internal/mcp/         Structured Codex tools
internal/schedule/    launchd and systemd user integration
```

The scanner invokes `codex exec --ephemeral` with a JSON output schema and a read-only sandbox. The current Codex default model is used unless the user configures a persistent or per-run override. Interactive investigations remain in the user's active Codex session instead of being launched by this analyzer. This keeps authentication, model availability, workspace policy, and usage accounting in Codex instead of duplicating them in Loose Ends.

The vault layer is model-agnostic and deterministic. Tests treat the examples and guarantees in this README as the behavioral contract.

## FAQ

### Why not SQLite?

Loose Ends is a personal, human-scale backlog, not an event warehouse. SQL would make direct inspection and editing harder while adding a second representation that could drift from the reports. Markdown is the source of truth; a disposable search index can be introduced without changing that contract.

### Can I edit findings in my text editor?

Yes. That is a primary interface, not an escape hatch. The next command sees a new content revision, preserves the edit, and rejects pending changes based on the older version.

### Does this use my ChatGPT subscription?

Yes, when Codex is signed in with ChatGPT. Loose Ends calls `codex exec`, which reuses the Codex CLI's saved authentication. If Codex is signed in with an API key instead, usage is billed through that API account. See [Codex authentication](https://learn.chatgpt.com/docs/auth).

### Why not call the Responses API directly?

Direct API calls require separate API credentials and duplicate authentication, model selection, policy, and execution behavior already provided by Codex. Loose Ends uses Codex as its analysis runtime and remains a local memory/workflow tool.

### Will a dismissed idea come back every day?

No. Dismissed findings participate in reconciliation. Repeated evidence is attached silently; genuinely new evidence may appear as a recommendation to reconsider, but the status does not change automatically.

### Can multiple machines share one vault?

Yes, through a private synchronization mechanism such as Git, provided only one machine writes at a time. Source checkpoints are machine-specific. Concurrent collaborative editing is outside Loose Ends' single-user consistency model; conflicts must be resolved before applying pending changes.

### Can Loose Ends open GitHub issues or modify code?

Not as a consequence of scanning or approving a finding. Those actions belong to a separate, explicit implementation workflow. A future integration may help hand an adopted finding to such a workflow, but the handoff must be user initiated and cannot change the vault's approval rules.
