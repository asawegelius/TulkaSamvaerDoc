# Architecture

Service overviews for Tulka's backend modules, based on [service-overview-template.md](../templates/service-overview-template.md). Start with the [platform overview](platform-overview.md) for how sections and shared platform modules fit together.

| Document | Module |
|:---------|:-------|
| [platform-overview.md](platform-overview.md) | Whole system: sections, platform modules, frontend layers |
| [booking-service-overview.md](booking-service-overview.md) | `sections/booking` |
| [list-queries-design.md](list-queries-design.md) | Search, filtering, paging, counts and live updates for lists (proposal, [ADR-007](../decisions/ADR-007-database-driven-list-queries.md)) |

Add one overview per module when work on it starts. Decisions behind the structure are in [decisions/](../decisions/README.md).
