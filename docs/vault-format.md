# Vault format

The vault is canonical memory. Everything Loose Ends has accepted must be reconstructable from these files alone, without a database, cache, report, or model call.

## Layout

```text
~/LooseEnds/
├── README.md                     Written by init; explains the vault to a human
├── config.toml                   Explicit user configuration
├── projects/
│   └── my-app/
│       ├── project.md            Identity, repository path, scan health
│       └── findings/
│           ├── LE-0042.md
│           └── LE-0047.md
├── reports/
│   └── 2026-08-19.md
└── .looseends/
    ├── checkpoints.json          Per-source cursors and machine identity
    ├── runs/                     One record per scan or report run
    ├── lock                      Single-writer guard
    └── cache/                    Disposable indexes and model-run artifacts
```

The distinction is intentional:

- Findings, projects, and reports are readable documents.
- `config.toml` is explicit user configuration, never written by a scan.
- `.looseends/checkpoints.json` holds replaceable source cursors and machine identity.
- `.looseends/runs/` holds run records. These are append-only and are load-bearing for the anti-recursion invariant described in [scanning.md](scanning.md).
- `.looseends/cache/` is disposable. Removing it never loses canonical state.

A vault may live in a private Git repository. Loose Ends neither requires Git nor performs any Git operation on the vault.

## Identifiers

- Findings are `LE-NNNN`, allocated per vault, never reused.
- Runs are `RUN-NNNN`.
- Identifiers are stable. A finding's ID does not change when its content, project, or status changes.

## Finding schema

Frontmatter is small and typed. Unknown keys are preserved verbatim.

| Field | Type | Owner | Notes |
| --- | --- | --- | --- |
| `id` | `LE-NNNN` | system | Immutable |
| `title` | string | user | One line, no trailing period |
| `project` | string | system | Must match a registered project |
| `status` | enum | user | `proposed`, `adopted`, `deferred`, `resolved`, `dismissed` |
| `kind` | enum | scan, user | e.g. `reliability`, `correctness`, `security`, `performance`, `maintainability`, `product-opportunity` |
| `impact` | `low`\|`medium`\|`high` | scan, user | |
| `confidence` | `low`\|`medium`\|`high` | scan, user | How well evidence supports the claim |
| `created_at` | RFC 3339 | system | |
| `updated_at` | RFC 3339 | system | |
| `deferred_until` | date | user | Required when `status: deferred` |
| `dismissed_reason` | string | user | Required when `status: dismissed` |
| `resolved_reason` | string | user | Required when `status: resolved` |
| `sources` | list | scan | Append-only; see below |
| `related` | list of IDs | scan, user | Non-directional association |

### Sources are append-only

Each entry records where an observation came from:

```yaml
sources:
  - session: 019c8f42-...
    harness: claude-code
    observed_at: 2026-08-01T14:17:02-04:00
    repository_revision: 92af461
    run: RUN-0031
```

A scan may append a source entry to an existing finding to record that another session independently supports it. It may not modify or remove an existing entry, and appending a source never changes status, title, summary, impact, confidence, or any user-authored prose.

### Body sections

The body is Markdown. Loose Ends recognizes these headings and preserves any others you add:

- `## Summary` — what was observed
- `## Why it matters` — consequence
- `## Evidence` — observed facts, each traceable to a session or a file
- `## Recommended direction` — proposal, explicitly labeled as such
- `## Alternatives` — optional
- `## Open questions` — optional, preferred over invented certainty
- `## Notes` — append-only working notes from you or an interactive agent session

`## Notes` is the one body section an agent may append to without a decision from you. It is additive: an agent appends a dated entry and never edits or removes an existing one.

## Revisions and manual edits

A finding's revision is a SHA-256 hash of the complete file. It is computed on read rather than stored in the file, so editing the file in any editor produces a new revision with no bookkeeping on your part.

Manual edits always win. Loose Ends re-reads a finding immediately before writing to it and refuses to write over content that changed since it was read. If that happens during a scan, the finding is left untouched and the run record notes it.

## Validation

`looseends doctor` validates every document and reports problems without fixing canonical content. A finding with invalid frontmatter is skipped by scans rather than repaired, so a malformed file never becomes silent data loss.
