# API

Endpoint contracts between the Angular frontend and the Spring Boot backend, based on [api-contract-template.md](../templates/api-contract-template.md). The generated OpenAPI spec is the binding contract ([ADR-003](../decisions/ADR-003-openapi-contract.md)); these documents explain intent, rules and error codes that a spec does not.

## Conventions (proposed)

| Topic | Rule |
|:------|:-----|
| Paths | `/api/v1/<section>/<resource>`, plural nouns, kebab-case |
| IDs | UUIDs |
| Dates | ISO 8601; dates as `2026-12-24`, instants in UTC as `2026-12-24T17:00:00Z`; the client shows local time |
| Paging | `?page=0&size=20&sort=date,desc`; response `{ items, page, size, totalItems }` |
| Visibility | `visibility: { scope, groupIds }`, scope is `ONLY_ME`, `GROUPS` or `ALL_MEMBERS` |
| Permissions | Every item returns `allowedActions`, for example `["edit", "delete"]` |
| Errors | RFC 9457 problem details `{ type, title, status, detail, code }` with a stable `code` such as `booking.date-taken` |
| Status codes | 400 invalid input, 401 not logged in, 403 not allowed, 404 not found or not visible, 409 conflict |
| Concurrency | `version` on editable items; stale update returns 409 |

## Contracts

| Document | Endpoints |
|:---------|:----------|
| [booking-api-contract.md](booking-api-contract.md) | Booking resources, availability, bookings |
