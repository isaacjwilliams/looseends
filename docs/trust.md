# Trust, privacy, and failure behavior

## Invariants

These hold for every version of Loose Ends. They are the product, not implementation preferences.

1. **Markdown is canonical.** Accepted state is reconstructable without a database, cache, report, or model call.
2. **The user owns the files.** Manual edits are supported and take precedence. Unknown frontmatter and user-authored sections are preserved.
3. **There are no silent decisions.** A scan may create a `proposed` finding and append an immutable source observation. It may not adopt, dismiss, defer, resolve, or reopen anything.
4. **There are no silent rewrites.** No automated path rewrites curated finding content. In v1 the capability does not exist: scans and agent sessions may only create findings, append sources, and append notes.
5. **Every claim has provenance.** Session evidence, repository evidence, inference, and recommendation stay distinguishable in storage and in presentation.
6. **Every mutation is auditable.** Scan-created content points to a run record. User decisions record their reason and time.
7. **Scans are idempotent.** Reprocessing the same source cannot duplicate findings, sources, or evidence.
8. **Unattended capture is read-only.** Scheduled scanning never modifies source repositories, runs project commands, connects to project services, installs dependencies, or provisions infrastructure.
9. **Derived state is disposable.** Reports, indexes, and caches are rebuildable from canonical documents.
10. **Model output is untrusted input.** Deterministic Go code validates schemas, identifiers, state transitions, paths, and file operations.
11. **Finding approval and code approval stay separate.** Adopting a finding never authorizes an implementation change.
12. **Loose Ends does not ingest itself.** Sessions created by Loose Ends are excluded by run record, in the vault layer, independent of harness.

## Privacy boundary

Loose Ends reads conversation history, so the boundary should be explicit.

- Only registered projects are scanned.
- `looseends scan --dry-run` performs no model call.
- Model calls go through your configured harness and inherit that login's workspace policies and data handling. Loose Ends adds no telemetry and contacts no service of its own.
- Relevant session excerpts and repository context are sent to the model. **Do not register projects whose history must not be processed this way.**
- Authentication files and tokens are never read, parsed, or copied by Loose Ends. The harness child process owns authentication.
- Exclusions are applied before model invocation, so excluded material is never transmitted.
- Vault writes use restrictive user-only permissions by default.
- Findings never trigger code changes, issue creation, messages, or external side effects on their own.

For a local audit of exactly what was sent and returned:

```sh
looseends scan --save-prompts DIR
```

These bundles can contain sensitive project material. They are not written unless you ask, and are not retained.

## Failure behavior

Loose Ends prefers an incomplete report over corrupt or misleading memory.

| Condition | Behavior |
| --- | --- |
| Harness not authenticated | Scan fails before advancing any checkpoint |
| Invalid structured model output | The affected excerpt is quarantined for retry; the run continues |
| Repository moved or missing | Reported as unavailable; never silently remapped |
| Finding has invalid frontmatter | Left untouched by scans; `doctor` explains the problem |
| Finding changed since it was read | The write is refused as stale; your edit stands |
| Interrupted mid-write | The journal is recovered on the next command |
| Cache corrupt | Deleted and rebuilt from Markdown |

Every run writes a record to `.looseends/runs/` describing its inputs, counts, warnings, created sessions, and resulting finding IDs. A failed run leaves a record too.
