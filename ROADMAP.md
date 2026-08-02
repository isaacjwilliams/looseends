# Loose Ends roadmap

This file records plausible extensions that are worth preserving but are not part of the current product contract. An item here is a design prompt, not a promise or an implementation queue.

The current behavior is documented in [README.md](README.md). Moving an item from this file into the README requires an explicit product decision.

## Managed investigation environments

**Status:** Exploratory; not part of v1.

Today, unattended scans perform static, read-only inspection. User-initiated investigations run in the user's active Codex session and inherit whatever repository, worktree, commands, services, sandbox, and approval policy that session already has. Loose Ends records the investigation but provisions no resources for it.

A managed investigation environment could give a longer-running or scheduled investigation an isolated place to form and test cause theories. Depending on the project, that might include:

- A disposable Git worktree at the investigated revision
- A project-defined container or development environment
- Dedicated Postgres, Redis, or other service instances
- Explicit fixtures or seed data with a known cleanup policy
- Permission to write diagnostic code and focused tests
- Captured test output and artifacts attached to the investigation
- A choice to discard diagnostic changes or hand them to an implementation workflow

This cannot be treated as merely a more permissive Codex sandbox. A sandbox constrains access to existing machine resources; it does not create databases, containers, credentials, or safe test data.

Before adopting this feature, the design needs answers to several questions:

1. Does Loose Ends own environment provisioning, or invoke a project-supplied environment adapter?
2. Are environments always ephemeral, resumable for an investigation, or user-selected?
3. How are migrations, seeds, credentials, ports, disk use, and cleanup handled safely?
4. How does the tool prove that a test used an isolated service rather than a developer or production resource?
5. Which commands can an unattended investigation execute, and how are they approved?
6. Can diagnostic commits be promoted into an implementation branch without coupling finding approval to code approval?
7. How should this work for projects that do not use containers or cannot cheaply duplicate their dependencies?

The likely safe shape is opt-in, project-defined, and adapter-based. Loose Ends would orchestrate a declared environment contract rather than attempting to infer how to reproduce every application stack. That direction remains a hypothesis until real investigation workflows demonstrate that static enrichment plus an interactive Codex session is insufficient.
