---
id: ADR-006
title: "Use Liquibase for database migrations"
status: draft
owner: "Åsa Wegelius (project owner)"
related: [ADR-001]
---

# ADR-006: Use Liquibase for database migrations

## Context

The backend needs versioned database schema changes, and each module owns its own tables ([ADR-001](ADR-001-one-modular-spring-boot-app.md)). The project owner's Spring Boot template uses Liquibase, and the owner asked on 2026-11-24 for the documentation to use Liquibase instead of Flyway.

## Decision

- Liquibase manages all schema changes.
- One master changelog, `db/changelog/db.changelog-master.yaml`, includes one folder per module: `db/changelog/<key>/` (for example `db/changelog/booking/`).
- Each module's changesets only touch that module's tables.
- Changesets are never edited after they reach a shared environment; corrections go in new changesets.

The changelog format (YAML, XML or SQL) and file naming follow the Spring Boot template once it is ready.

## Alternatives considered

- **Flyway**: proposed in the first architecture draft; replaced because the Spring Boot template uses Liquibase.

## Consequences

- Benefits: consistent with the Spring Boot template; changelogs per module keep a new section's tables self-contained; rollback support is available where needed.
- Costs, risks and tradeoffs: the master changelog is a shared file that every new module touches with one include line.

## Related artifacts

- [Platform overview](../architecture/platform-overview.md)

## Follow-up work

- Align changelog format and naming with the Spring Boot template. Owner: Åsa Wegelius.

## Replacement history

None.
