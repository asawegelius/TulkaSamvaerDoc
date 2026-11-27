# Git strategy

Use reviewed task branches with `develop` for integration and `main` for stable releases. The upstream documents-template repository verifies a pinned candidate before promotion. The shared `test` line is available for projects that need separate staging.

## Reading order

1. [Overview](01-overview.md)
2. [Branching model](02-branching-model.md)
3. [Feature development](03-feature-development.md)
4. [Bugfix handling](04-bugfix-handling.md)
5. [Versioning and tagging](05-versioning-and-tagging.md)
6. [Verification and release promotion](06-ci-cd-flow.md)

Diagrams are in `diagrams/`. The text explains the workflow without requiring an image reader.

## Applying the guidance

- Confirm which branches exist and what protections apply before following commands.
- Use a task branch and reviewed pull request to the approved integration or release branch.
- Use the project's work-item system; Jira is an example. Link requirement, use case and test IDs in the pull request independently of branch names.
- Record evidence for the actual tested revision and rerun affected checks after integration changes.
- Record release and promotion approval separately from verification evidence.

## Review checklist

- Does the model fit the release cadence and contributors?
- Can a change be traced from requirement to evidence and pull request?
- Are fixes propagated to every affected supported line?
- Are release permissions explicit for both people and agents?

See [CONVENTIONS.md](../CONVENTIONS.md) and [AGENTS.md](../AGENTS.md). Configure project branch protections and required checks through the project's approved settings.
