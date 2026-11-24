# Platform overview

Status: draft. Owner: Åsa Wegelius (project owner).

## Purpose

Tulka is a members webapp for family and friends around a farmhouse whose barn holds a guest apartment and a pottery studio. Each area of the app is a **section**. Sections share one **platform** for members, groups, visibility, calendar, media and messaging.

## System shape

```text
Angular frontend (pure view)            see ADR-002
  core/       auth, section registry, error handling, generated API client
  layout/     app shell: toolbar, side navigation from SECTIONS
  shared/     reusable layouts, shared components, theme
  features/   calendar, booking, events, recipes, chat, garden, members, admin, <new>
        |
        |  REST + JSON under /api/v1, typed from OpenAPI      see ADR-003
        v
Spring Boot backend (one modular app, Spring Modulith)        see ADR-001
  sections/   booking, events, recipes, garden, <new>
  platform/   identity, groups, visibility, calendar, media, messaging, sections
  shared/     error model, paging, base entity, clock
        |
        v
PostgreSQL (tables per module, Flyway per module)  +  media storage (disk or S3)
```

## Sections

| Key | Section | Status | Uses platform |
|:----|:--------|:-------|:--------------|
| `calendar` | Shared calendar | planned | calendar (aggregates all sections) |
| `booking` | Guest apartment and pottery studio booking | planned | calendar, identity, messaging |
| `events` | Events, invites, recurring events | planned | calendar, visibility, groups, media |
| `recipes` | Recipes with categories, tags, cooking view | planned | visibility, media |
| `chat` | Conversations | planned | messaging |
| `garden` | Garden diary per person, plant index, photo export | planned | visibility, media, optionally calendar |
| `robot` | Robot workshop | idea | to be decided |
| `music` | Music studio | idea | to be decided; the studio could be a bookable resource |

## Platform modules

| Module | Responsibility |
|:-------|:---------------|
| `identity` | Members, roles (proposed: Admin, Inner circle, Member), invitations, login (ADR-005 open) |
| `groups` | Member-created groups and memberships |
| `visibility` | Visibility value (Only me, selected groups, all members) and the query filter every list endpoint uses |
| `calendar` | `CalendarContributor` extension point; aggregated calendar endpoint |
| `media` | Upload, full-size storage, thumbnails, zip export |
| `messaging` | Chat conversations and notifications |
| `sections` | Section registry and per-user section access (ADR-004) |

## Rules every section follows

- Business rules live in the section's backend module, never in the frontend.
- Shareable items embed the platform visibility value; hidden items answer 404.
- Every item returned by the API lists `allowedActions`.
- Dated items reach the shared calendar by implementing `CalendarContributor`.
- Uploads go through the media module; sections store media IDs.

## Adding a section

1. Backend module `sections/<key>` with `api`, `application`, `domain`, `infrastructure`; Flyway migrations in `db/migration/<key>/`.
2. Register the section in `platform/sections`.
3. Frontend feature `features/<key>` built from existing layouts; add it to `SECTIONS`.
4. Regenerate the API client; add mock data for frontend work.
5. Documentation: requirements with domain `<KEY>`, a service overview in this folder, API contracts in [api/](../api/README.md), and a coverage view.
