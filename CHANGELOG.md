# Changelog

Document changes that affect adopters here. See the [template compatibility contract](UPGRADING.md#template-compatibility).

## Unreleased

## v1.0.1 - 2026-11-27

### Fixed

- Verify a complete legacy adoption across its approved contiguous commits while preserving project-owned records at every step. Existing single-plan verification remains available for individual transitions.
- Reject transition plans whose approval references differ from the frozen run decision, even when the run and plan digests match.
- Separate signed archive verification from Git-snapshot adoption and explain how to reconcile unknown legacy anchors.

### Changed

- `AGENTS.md` explicitly states that examples and procedures cannot authorize environment branches, merges, tagging, releases or deployments. The project's existing authorization remains required.
- Shared Git guidance and the starter README describe project choices. Template release requirements belong to the template maintainer, and the adopter procedure governs changes to managed content.
- The adopter procedure describes frozen plans, approval binding, endpoint checks and retained evidence for complete legacy adoption. A local verifier result leaves authorized acceptance to the project's authenticated review.

### Added

- Executable legacy and adapted provenance examples.

## v1.0.0 - 2026-11-26

### Added

- An explicit distribution inventory and named Git release archive.
- An adoption workflow with project-owned provenance, per-path anchors and explicit optional capability state.
- Support for canonical skill sources with client-specific mappings, installed-content verification and controlled removal.
- Verification and recovery instructions for maintainers and adopters.

The first baseline distributes core documentation and includes no optional capability payloads.
