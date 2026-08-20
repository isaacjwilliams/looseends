# Loose Ends roadmap

This file records plausible extensions that are worth preserving but are not part of the current product contract. An item here is a design prompt, not a promise or an implementation queue.

Current behavior is documented in [README.md](README.md) and [docs/](docs/). Moving an item from this file into the product requires an explicit product decision.

## Staged change sets

**Status:** Deferred from v1.

v1 gets "no silent rewrites" by removing the capability rather than governing it: nothing automated rewrites curated finding content. You edit the Markdown, or you issue an explicit decision command. That is a complete and honest product, but it means an agent that has just read your code cannot improve a finding's wording, sharpen its recommendation, or fold in what it learned beyond appending a note.

The mechanism that would allow it, without giving up the guarantee:

- A **change set** is a versioned proposal to mutate one or more findings.
- It records the exact semantic operations, the base revision of every affected finding, the supporting repository revision, a human-readable semantic diff, and its author, creation, approval, and disposition times.
- Applying is transactional: validate all base revisions, journal the operation, replace files atomically, and finish or roll back an interrupted application on the next run.
- If a finding was edited manually after the proposal was staged, application fails as stale. A rebased proposal requires review again. Manual edits always win.
- **Adopt** accepts a finding into the backlog. **Approve** applies a change set. The words are not interchangeable.
- Applied and rejected change sets are retained. Pending ones can be abandoned but are archived rather than erased.

Open questions:

1. Does a fully specified single instruction ("dismiss LE-0044 as intentional") stage a one-operation change set, or remain a direct command with an audit entry?
2. Is a semantic diff worth the complexity over a plain Markdown diff, given that findings are short?
3. What is the review surface in a terminal conversation, where a long diff is expensive to read?

## Merge, split, and supersede

**Status:** Deferred from v1; depends on change sets.

Reconciliation in v1 can link related findings but cannot restructure them. Merging duplicates, splitting a finding that turned out to be two problems, and a `superseded` status all mutate curated content across multiple documents at once, which is exactly what change sets exist to govern.

## Investigation documents

**Status:** Deferred from v1.

v1 records exploratory work as dated entries under a finding's `## Notes`. That is sufficient for a short investigation and keeps the finding as the single document.

A first-class investigation would hold verbose, evolving work that would make a finding noisy: questions, experiments, evidence, competing explanations, and a conclusion, stored as `INV-NNNN` documents with their own lifecycle. It would be checkpointed after material experiments, resumable across sessions, and useful even when its proposed changes are rejected.

This becomes worth building when notes routinely grow long enough to bury the finding, and it is most valuable alongside change sets, since an investigation's natural output is a proposal to revise the finding.

## Unattended enrichment

**Status:** Deferred from v1.

v1 scanning extracts and reconciles; it does not open your repository. Enrichment would let a scheduled scan inspect repository files in a read-only sandbox to clarify impact and produce a concrete recommended direction, bounded to static inspection: no tests, no project commands, no databases, no containers.

It is deferred because it is the slowest, most expensive, and highest-variance step, and because a finding with an honest open question is more useful than one with an invented recommendation. The bounding rules matter more than the feature: enrichment is meant to preserve a useful lead, not to perform an autonomous multi-hour investigation.

## Multi-harness configuration

**Status:** Exploratory; the contract exists, the configuration does not.

[docs/harnesses.md](docs/harnesses.md) already separates the ingestion role from the analysis role. v1 collapses them: one harness, vault-wide. The full shape:

- **Multiple ingestion harnesses at once.** A vault should be able to read Claude Code and Codex sessions in the same scan, with each finding's sources recording which harness observed it.
- **Per-project ingestion.** Different repositories may be worked in different agents. Ingestion configuration belongs alongside project registration, not only in the vault root.
- **One analysis harness and model.** Analysis should stay singular and vault-wide, so findings are produced by a consistent evaluator.
- **Ingestion and analysis are independent.** Reading Codex sessions and analyzing them with Claude Code must be expressible. If it is not, the seam is not real.

Sketch:

```toml
[analysis]
harness = "claude-code"
model = ""

[[ingest]]
harness = "claude-code"

[[ingest]]
harness = "codex"
projects = ["work-api"]
```

Open questions: how a session ID stays unambiguous across harnesses in `sources`; whether checkpoints are per harness per project or per transcript source; and how `harness install` behaves when several harnesses want a conversational integration to the same vault.

## Isolated harness home for analysis

**Status:** Exploratory; harness-specific.

The anti-recursion guarantee lives in the vault: run records name the sessions Loose Ends created, and ingestion skips them. That is harness-independent and is where correctness belongs.

Separately, a scheduled scan currently inherits whatever configuration the user's agent has: hooks, plugins, MCP servers, status lines, custom permissions. None of that should influence unattended analysis. Running analysis against a dedicated `CLAUDE_CONFIG_DIR` would make scans hermetic and reproducible, and would keep the analysis agent from having access to the Loose Ends MCP server at all.

The obstacle is authentication. On Linux, credentials live inside the config directory, so relocating it likely leaves the analysis run unauthenticated; macOS keychain-backed auth may behave differently. Options are to symlink the credential file into the isolated home, which weakens the "never reads or copies credentials" promise to "never reads, parses, or copies," or to have `harness install` create the home and ask the user to authenticate it once, which keeps the promise intact at the cost of a setup step.

This needs an experiment before it is documented as behavior. It must remain defense in depth: no correctness guarantee may move out of the vault layer and into a harness-specific mechanism.

## Managed investigation environments

**Status:** Exploratory; well beyond v1.

Unattended scans perform static, read-only inspection. Interactive investigation runs in the user's active agent session and inherits whatever repository, worktree, commands, services, sandbox, and approval policy that session already has. Loose Ends records the work but provisions nothing.

A managed investigation environment could give a longer-running or scheduled investigation an isolated place to form and test cause theories:

- A disposable Git worktree at the investigated revision
- A project-defined container or development environment
- Dedicated Postgres, Redis, or other service instances
- Explicit fixtures or seed data with a known cleanup policy
- Permission to write diagnostic code and focused tests
- Captured test output and artifacts attached to the investigation
- A choice to discard diagnostic changes or hand them to an implementation workflow

This cannot be treated as merely a more permissive sandbox. A sandbox constrains access to existing machine resources; it does not create databases, containers, credentials, or safe test data.

Before adopting this feature, the design needs answers to several questions:

1. Does Loose Ends own environment provisioning, or invoke a project-supplied environment adapter?
2. Are environments always ephemeral, resumable for an investigation, or user-selected?
3. How are migrations, seeds, credentials, ports, disk use, and cleanup handled safely?
4. How does the tool prove that a test used an isolated service rather than a developer or production resource?
5. Which commands can an unattended investigation execute, and how are they approved?
6. Can diagnostic commits be promoted into an implementation branch without coupling finding approval to code approval?
7. How should this work for projects that do not use containers or cannot cheaply duplicate their dependencies?

The likely safe shape is opt-in, project-defined, and adapter-based: Loose Ends orchestrates a declared environment contract rather than inferring how to reproduce every application stack. That remains a hypothesis until real investigation workflows demonstrate that static inspection plus an interactive session is insufficient.

## Smaller deferred items

- `looseends runs` — a command to browse run records. The records exist in v1; the browser does not.
- Reconsideration entries — reporting that a dismissed or resolved finding has materially new evidence, without changing its status.
- Independent scan and report schedules with distinct cadences, and separate daily and weekly report types.
- A disposable search index over findings, permissible only if deleting it loses no accepted state.
- Handing an adopted finding to an implementation workflow. The handoff must be user initiated and must not change the vault's approval rules.
