# Template versioning and tagging

This chapter describes how the upstream documents-template versions and publishes its releases. This project records the adopted template release in [template-provenance.yml](../template-provenance.yml). The project's own release versioning is not defined here.

Use [Semantic Versioning](https://semver.org/) and an annotated `vMAJOR.MINOR.PATCH` tag on the exact approved `main` commit. Use the tag and full resolved SHA in provenance. The GitHub Release publishes the named adopter archive and reviewed release notes.

## Compatibility contract

The contract includes documented artifact types, locations, IDs, required metadata, relationships, status and evidence semantics, and supported validation and adoption interfaces. It also includes mandatory policies, agent behavior, development and review operations, quality gates, human approvals, and verification and evidence obligations.

| Version | Change |
| --- | --- |
| Patch | Correction that preserves published behavior and obligations. |
| Minor | Compatible addition or deprecation. Projects that keep their existing configuration can still meet their current obligations. |
| Major | Change to a promised interface or obligation that requires migration or changes a governance guarantee. Obtain explicit human approval and verify migration and recovery. |

Projects explicitly opt into optional capabilities. Once adopted, supported native triggers run the required workflow on applicable work. Existing human approval gates continue to apply.

## Release record

Summarize impact, prerequisites, optional changes, migration order, verification and recovery in the [changelog](../CHANGELOG.md) and GitHub Release. Generated notes can supplement the reviewed release explanation with the PR list. Keep previously published source snapshots immutable and issue fixes under new versions. Follow the [adoption guide](../UPGRADING.md) to verify an archive or use it in an existing project.
