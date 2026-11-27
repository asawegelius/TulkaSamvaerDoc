# Verification and release promotion

Work items flow through isolated branches and reviewed pull requests into `develop`. Verify the pinned candidate before promoting it to `main`. For the upstream documents-template repository, a candidate PR and clean-checkout validation serve the staging role. Projects that need a shared staging line can use the optional `test` branch.

The template source repository runs read-only verification on candidates. It checks schema conformance, the exact archive inventory and content, links and dependencies in the extracted archive, and upgrade and recovery fixtures. Record the source commit, tool versions, commands, scope and observed results. Rerun affected checks on the resulting stable commit before release.

For the template, the accountable human approves the candidate, creates its annotated tag and publishes the GitHub Release. Downstream projects choose their own checks and approve adoption in a separate pull request. See the [release policy](05-versioning-and-tagging.md) and [upgrade guide](../UPGRADING.md).
