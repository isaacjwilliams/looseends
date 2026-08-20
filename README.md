# Loose Ends

**Ambient issue capture for your coding agent.**

Loose Ends turns the unfinished observations buried in your coding sessions into a durable, reviewable backlog. It notices shortcomings, risks, cleanup opportunities, and product ideas that came up while you were working but were never resolved. It writes them to plain Markdown and brings the important ones back when you have time to act on them.

You do not maintain a second TODO list while coding. You keep working. Loose Ends remembers what was left behind.

```text
agent sessions ──▶ unresolved observations ──▶ Markdown findings ──▶ reports
                                                      ▲                 │
                                                      │                 ▼
                                                      └───── review and decide
```

Loose Ends is local-first, single-user software. It is distributed as one Go binary, uses your existing coding-agent login, runs without a database or resident daemon, and never changes a source repository during a scheduled scan.

[Claude Code](https://claude.com/claude-code) is the first supported harness. Loose Ends is built around a harness contract rather than a single vendor, so reading sessions from other agents is an adapter, not a rewrite — see [docs/harnesses.md](docs/harnesses.md).

## Quick start

### Requirements

- macOS or Linux
- Claude Code installed and available as `claude`
- A Claude subscription or API key configured through Claude Code
- Git repositories for the projects you want to monitor

Loose Ends delegates model work to headless Claude Code, which reuses your existing login. Loose Ends never reads or copies your credentials; the `claude` child process owns authentication.

### Install

```sh
brew install looseends
```

Release archives contain a single `looseends` binary and require no language runtime. To build the current checkout instead:

```sh
go install ./cmd/looseends
```

### Create a vault and register a project

```sh
looseends init --vault ~/LooseEnds

cd ~/Code/my-app
looseends project add .
```

The vault is the tool's memory: ordinary Markdown files that can live anywhere you choose. `~/LooseEnds` is only the suggested location. Registration records the repository path and identity in the vault; it does not add files to your repository.

### Scan and report

```sh
looseends scan
looseends report --since 7d
```

`scan` reads new Claude Code sessions for registered projects and writes findings. `report` prints the path to a generated Markdown report; pass `--stdout` to print the report itself.

Use `looseends scan --dry-run` to see which sessions would be considered without making a model call or touching the vault.

### Connect Loose Ends to Claude Code

```sh
looseends harness install claude-code
looseends harness status
```

This installs a Claude Code plugin that bundles the `/looseends` skill and a local MCP server, so the integration arrives and leaves as one unit. Then start Claude Code in any registered project and ask for what you want:

```text
/looseends show me findings from the last week
```

### Schedule scans and reports

```sh
looseends schedule install --scan every=3h --report daily@17:30
```

Loose Ends installs a user-level systemd timer on Linux or a launchd agent on macOS. There is no resident daemon. Missed scans resume from their last checkpoint the next time the command runs.

## The workflow

During the day, Loose Ends works in the background: it reads new sessions for registered projects, identifies observations that were neither fixed nor deliberately rejected, reconciles them against findings you already have so repeated observations do not become repeated TODOs, and generates a report without changing the state of anything.

When you have time to review, your agent becomes the interface:

```text
You:     /looseends what did I leave behind last week?

Claude:  Six new findings across two projects. The two worth your
         attention are LE-0042 and LE-0047.

You:     Dismiss LE-0044, that's intentional. Adopt LE-0042.
         What's the story on LE-0047?

Claude:  Dismissed LE-0044 and adopted LE-0042.

         LE-0047 says retries can extend request latency without a
         fixed bound. I read internal/client/retry.go at the current
         revision: each attempt starts a fresh timeout, so a degraded
         dependency can hold a request worker much longer than the
         caller expects. I appended what I found to the finding's
         notes.

You:     Adopt it, but defer it until after the release.

Claude:  Adopted LE-0047 and deferred it until 2026-09-05.
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

Only you change a finding's status. A scan can create a new `proposed` finding and append sourced evidence to an existing one; it cannot adopt, dismiss, defer, resolve, or reopen anything, and it cannot rewrite a finding's title, summary, impact, or recommendation. Curated content changes when you edit the file or issue an explicit command — never because a model decided it should.

Loose Ends also does not silently reopen dismissed or resolved findings. Repeated evidence for a dismissed idea is absorbed quietly. A genuine regression becomes a new finding linked to the resolved one.

## The Markdown vault

There is no SQL database. Canonical memory is stored in Markdown with small, typed YAML frontmatter blocks.

```text
~/LooseEnds/
├── config.toml
├── projects/my-app/
│   ├── project.md
│   └── findings/
│       ├── LE-0042.md
│       └── LE-0047.md
├── reports/
└── .looseends/          disposable: checkpoints, run records, cache
```

Findings, projects, and reports are readable documents. `config.toml` is explicit user configuration. Everything under `.looseends/` is replaceable — removing it never loses accepted state.

A finding looks like this:

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
related: [LE-0039]
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

## Recommended direction

Carry one deadline through the complete retry sequence. Stop retrying when the
remaining budget cannot accommodate the next backoff and attempt.

## Open questions

- Should callers be able to supply a smaller total retry budget?
```

Editing that file in your text editor is a primary interface, not an escape hatch. Loose Ends preserves unknown frontmatter fields and any sections you write yourself. You can keep the vault in a private Git repository for history and synchronization; Loose Ends does not require Git and will not push or pull for you.

The full schema is documented in [docs/vault-format.md](docs/vault-format.md).

## Reports

Reports are inboxes, not databases. They link to canonical findings and summarize what deserves attention: new proposals, existing findings with material new evidence, high-impact adopted findings that have gone untouched, and projects that could not be scanned.

```sh
looseends report --since 7d
looseends report --project my-app --status proposed,adopted --stdout
```

Generating or reading a report never changes a finding's status. Reports are derived views and can be deleted and rebuilt at any time.

## Trust contract

Loose Ends makes a small set of promises that should be visible without understanding its implementation:

- Your Markdown files are canonical, editable, and sufficient to reconstruct accepted state.
- Scans propose findings and append sourced evidence. They cannot make decisions or rewrite curated content.
- Every claim distinguishes evidence, inference, and recommendation.
- Unattended capture is read-only with respect to source repositories. Scans never modify your code, run project commands, or install dependencies.
- Only registered projects are scanned, and session excerpts are sent to the model that your harness is logged into. Do not register projects whose history must not be processed this way.
- Findings never trigger code changes, issue creation, messages, or external side effects on their own.
- Reports, indexes, and caches are derived and disposable.

The privacy boundary and failure behavior are detailed in [docs/trust.md](docs/trust.md).

## What Loose Ends is not

- It is not a replacement for an issue tracker. Adopted findings can inform one, but Loose Ends captures ideas before they deserve team-visible tickets.
- It is not a transcript summarizer. Most sessions should produce no findings.
- It is not an autonomous repair bot. It never fixes findings merely because it discovered them.
- It is not a hidden memory service. There is no hosted account and no opaque server-side state.
- It is not a general personal knowledge base. Its schema is deliberately specific to software shortcomings and improvements.

## Documentation

| Document | Contents |
| --- | --- |
| [docs/cli.md](docs/cli.md) | Full command reference |
| [docs/configuration.md](docs/configuration.md) | `config.toml` and exclusions |
| [docs/vault-format.md](docs/vault-format.md) | Vault layout, schemas, revisions |
| [docs/scanning.md](docs/scanning.md) | Session selection, extraction, reconciliation |
| [docs/harnesses.md](docs/harnesses.md) | The harness contract and the Claude Code adapter |
| [docs/integration.md](docs/integration.md) | The Claude Code plugin, skill, and MCP tools |
| [docs/trust.md](docs/trust.md) | Invariants, privacy boundary, failure behavior |
| [ROADMAP.md](ROADMAP.md) | Explored but uncommitted directions |
| [CONTRIBUTING.md](CONTRIBUTING.md) | Building, testing, and code layout |

## FAQ

### Can I edit findings in my text editor?

Yes. That is a primary interface. The next command sees your edit, preserves it, and treats it as the current state.

### Does this use my Claude subscription?

Yes, when Claude Code is signed in with a subscription. Loose Ends invokes Claude Code headlessly, which reuses its saved authentication. If Claude Code is signed in with an API key instead, usage is billed to that account.

### Will a dismissed idea come back every day?

No. Dismissed findings participate in reconciliation, so repeated evidence is absorbed silently rather than resurfacing as a new proposal.

### Can multiple machines share one vault?

Yes, through a private synchronization mechanism such as Git, provided only one machine writes at a time. Session checkpoints are machine-specific. Concurrent collaborative editing is outside the single-user consistency model.

### Can Loose Ends open GitHub issues or modify code?

Not as a consequence of scanning or adopting a finding. Those actions belong to a separate, explicit implementation workflow.
