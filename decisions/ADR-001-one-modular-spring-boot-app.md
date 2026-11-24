---
id: ADR-001
title: "Build the backend as one modular Spring Boot application"
status: draft
owner: "Åsa Wegelius (project owner)"
related: [ADR-003, ADR-004]
---

# ADR-001: Build the backend as one modular Spring Boot application

## Context

Tulka is a members webapp for family and friends: shared calendar, booking, events, recipes, chat and garden diary, with a robot workshop and a music studio planned. All business logic must live in the backend. New sections must be easy to add. The user base is small: one family and friends circle.

## Decision

The backend is one deployable Spring Boot application split into modules with Spring Modulith.

- **Section modules** (`sections/booking`, `sections/events`, `sections/recipes`, `sections/garden`, ...) own one area each.
- **Platform modules** (`platform/identity`, `groups`, `visibility`, `calendar`, `media`, `messaging`, `sections`) are shared by every section.
- Each module has the layers `api`, `application`, `domain`, `infrastructure`. Only types in the module's root package are visible to other modules.
- Modules talk through public services, domain events and extension points such as `CalendarContributor`. They never read each other's tables.
- A Spring Modulith `ApplicationModules.verify()` test fails the build when a boundary is broken.

The project owner chose "one modular app" over separate services in the planning conversation on 2026-11-24. Formal approval is pending review of this record.

## Alternatives considered

- **Microservices, one per section**: rejected. It adds deployment, networking and data consistency work that a family-sized system does not need.
- **Plain layered monolith (controllers/services/repositories at top level)**: rejected. Sections would become tangled, which makes adding a new one harder.

## Consequences

- Benefits: one jar to deploy and back up; boundaries are still enforced; a module can be extracted later if it ever needs to scale on its own.
- Costs, risks and tradeoffs: discipline needed to keep root packages small; Spring Modulith becomes a dependency.

## Related artifacts

- [ADR-003](ADR-003-openapi-contract.md) API contract
- [ADR-004](ADR-004-section-registry.md) section registry
- [Platform overview](../architecture/platform-overview.md)

## Follow-up work

- Align with the Spring Boot template when it is ready (package names, build setup). Owner: Åsa Wegelius.

## Replacement history

None.
