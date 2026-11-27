# Git strategy overview

Use branches and reviewed pull requests to keep development, candidate verification and stable promotion traceable. Link the work item, affected document IDs, tested commit and evidence to each significant change.

- `develop` integrates reviewed work.
- A release candidate receives verification before promotion. The upstream documents-template repository uses a candidate PR and a clean checkout. Other projects can use an optional `test` branch for shared staging.
- `main` contains approved stable snapshots. The release publisher tags and publishes an approved snapshot after verification.
- Work item branches isolate features and fixes. Open a PR to the appropriate base and propagate fixes to each affected supported line.

See the [branching model](02-branching-model.md), [versioning and tagging](05-versioning-and-tagging.md) and [verification flow](06-ci-cd-flow.md).
