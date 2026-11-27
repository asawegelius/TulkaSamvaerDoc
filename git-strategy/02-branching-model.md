# Branching model

| Branch | Responsibility |
| --- | --- |
| `develop` | Integration for reviewed implementation and fixes |
| `main` | Approved stable release snapshots |
| `test` | Optional shared staging line for projects with a separate staging need |
| `feature/*` | Isolated work from the approved `develop` revision |
| `fix/*` | Corrections based on the affected candidate or staging line |
| `hotfix/*` | Urgent corrections based on the affected stable release |

The upstream documents-template repository uses a release PR and pinned verification. If a project uses a permanent `test` line, promote the candidate through reviewed PRs from `develop` to `test` and then to `main`. The [staging diagram](diagrams/branch-overview.png) shows that variant.

Review scope and evidence on the exact resulting stable commit before release. The accountable maintainer approves promotion, tagging and publication. See [versioning and tagging](05-versioning-and-tagging.md).
