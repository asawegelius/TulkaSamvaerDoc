---
id: ADR-002
title: "Use a pure Angular frontend built from reusable layouts"
status: draft
owner: "Åsa Wegelius (project owner)"
related: [ADR-003, ADR-004]
---

# ADR-002: Use a pure Angular frontend built from reusable layouts

## Context

The project owner wants the frontend to be a pure view with no business logic, built on [AngularAWTemplate](https://github.com/asawegelius/AngularAWTemplate) and using the principle of reusable layout.

## Decision

- Start from AngularAWTemplate (Angular, standalone components, Angular Material, NgRx) and keep its layers: `core/`, `shared/`, `layout/`, `features/`.
- Each section is one lazy-loaded feature folder with the template's anatomy: `pages/`, `components/`, `data-access/`, `models/`, `state/`, `facade/`, `testing/`.
- Pages are composed from layout components in `shared/layouts/`: `PageLayout`, `ListDetailLayout`, `CalendarLayout`, `TimelineLayout`, `GalleryLayout`, `FormLayout`, `ReaderLayout`. Layouts own structure and responsiveness and expose named slots; they know nothing about any section.
- The frontend makes no business decisions. It shows actions the API lists in `allowedActions` and leaves availability, visibility and permission rules to the backend.
- The theme in `shared/theme/` implements the Stoneware glaze style guide as design tokens.

## Alternatives considered

- **Section-specific page structures**: rejected; every new section would rebuild list, calendar and form screens.
- **Business rules duplicated in the frontend for speed**: rejected; rules would drift from the backend.

## Consequences

- Benefits: a new section is mostly composition of existing layouts; consistent look across sections.
- Costs, risks and tradeoffs: layouts need careful slot design up front; a section with a truly new screen type needs a new layout.

## Related artifacts

- [ADR-003](ADR-003-openapi-contract.md), [ADR-004](ADR-004-section-registry.md)

## Follow-up work

- Template fixes noted during review of AngularAWTemplate: `ResourceService.update()` skips `toServerModel()`; `getList(index, page)` parameter names do not match `limit`/`offset`; `handleError` drops HTTP status and error body; README says Angular 20 while `package.json` uses 22.

## Replacement history

None.
