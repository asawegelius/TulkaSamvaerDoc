---
id: ADR-004
title: "Use one section key across frontend, backend, API and documentation"
status: draft
owner: "Åsa Wegelius (project owner)"
related: [ADR-001, ADR-002]
---

# ADR-004: Use one section key across frontend, backend, API and documentation

## Context

New sections (robot workshop, music studio) must be easy to add, and the user should only see sections they have access to.

## Decision

- Each section has one lowercase key, for example `booking`. The key names the Angular feature folder, the backend module, the API prefix and the table prefix. Its uppercase form is the documentation domain (`REQ-BOOKING-001`).
- The backend module `platform/sections` holds the registry of sections and who may use each. `GET /api/v1/me/sections` returns the sections the current user may see.
- The frontend's `SECTIONS` list holds title, icon, order and the lazy route loader. The app shell builds its menu and routes from it, filtered by the backend's answer.
- Frontend route guards are a convenience only; the backend enforces section access on every call.

## Alternatives considered

- **Hand-written menu and routes per section**: rejected; every new section would touch shared files in several places.

## Consequences

- Benefits: adding a section is new folders plus one registry entry on each side.
- Costs, risks and tradeoffs: keys must stay stable once data and links exist.

## Related artifacts

- [Platform overview](../architecture/platform-overview.md)

## Follow-up work

None.

## Replacement history

None.
