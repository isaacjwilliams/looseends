# Claude Code integration

The integration ships as a single Claude Code plugin, installed and removed as one unit:

```sh
looseends harness install claude-code
looseends harness status
looseends harness uninstall claude-code
```

The plugin bundles two things: the `/looseends` skill, which teaches Claude the workflow and vocabulary, and a local stdio MCP server, which exposes focused domain operations. Both are backed by the same Go domain layer as the CLI. Conversational operations are not a second implementation of vault behavior.

## The `/looseends` skill

The skill requires Claude to:

- Resolve the current repository to a registered project before querying
- Distinguish evidence, inference, and recommendation in everything it reports
- Use MCP operations rather than editing vault files directly
- Append working notes rather than rewriting curated finding content
- Treat `adopt`, `defer`, `dismiss`, and `resolve` as decisions that belong to the user, and never infer one from ambiguous phrasing
- Report a finding's status and provenance honestly, including low confidence

The skill can be invoked explicitly as `/looseends`. Claude may also select it when a request clearly concerns findings.

## MCP tools

The server exposes focused operations rather than filesystem or database access. There is no `write_file`, no generic SQL, and no unrestricted `update_finding`.

Read-only:

- `list_findings` — query by project, status, kind, impact, and recency
- `get_finding` — return one canonical finding
- `get_report` — return a generated report

Append-only:

- `append_finding_note` — add a dated working note to a finding's `## Notes` section
- `link_findings` — record a non-directional relationship

Decisions:

- `adopt_finding`
- `defer_finding` — requires a date or condition
- `dismiss_finding` — requires a reason
- `resolve_finding` — requires a reason
- `reopen_finding` — requires a reason

Mutating tools advertise themselves as writes, so Claude Code's approval policy applies its own confirmation. That confirmation is a host-level safeguard; it does not replace the vault's own rules about what may change.

## What the integration cannot do

An interactive session may read your repository and append notes to a finding. It may not rewrite a finding's title, summary, evidence, impact, confidence, or recommendation. Curated content changes when you edit the file yourself, or through an explicit decision command.

A direct, fully specified instruction — "dismiss LE-0044 as intentional behavior" — is a decision you made, and Claude may carry it out. Broad instructions such as "clean this finding up" have no v1 implementation; Claude should say so rather than improvise. The staged-change-set mechanism that would allow model-authored revision is described in [ROADMAP.md](../ROADMAP.md).

Adopting a finding never authorizes an implementation change. Finding approval and code approval stay separate.
