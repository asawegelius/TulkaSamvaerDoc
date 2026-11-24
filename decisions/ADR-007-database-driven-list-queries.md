---
id: ADR-007
title: "Run search, filtering, sorting, paging and counts in the database"
status: draft
owner: "Åsa Wegelius (project owner)"
related: [ADR-001, ADR-003, ADR-005, ADR-006]
---

# ADR-007: Run search, filtering, sorting, paging and counts in the database

## Context

On 2026-11-24 the project owner asked to revise how lists are searched, filtered and paged, starting from what the database can do, and to describe the design before the implementation changes. Her requirements (translated from Danish):

1. Dynamic filtering, sorting and paging run in the database.
2. Users can combine filters across the relevant fields.
3. Search results update from the first typed character, when the user pauses briefly.
4. The API fetches only the number of rows the client asks for.
5. Hibernate is not used for filtering, paging or batch fetching as a replacement for efficient SQL.
6. Paging has a deterministic sort order.
7. Active user sessions can react to relevant inserts, updates and deletes.
8. No caching of search results in frontend or backend; the database handles its own caching.
9. The total number of rows and the filtered number of rows are two separate problems.

She also asked to investigate a transactionally maintained counter table for the total and an efficient strategy for filtered counts.

Production runs PostgreSQL; development and tests use H2 in PostgreSQL mode. The current Events list (API PR 1) pages with JPQL and `OFFSET`, counts with a second JPQL query on every page, and builds summary counts from lazy collections with `hibernate.default_batch_fetch_size`. The full analysis is in [list-queries-design.md](../architecture/list-queries-design.md).

## Decision

**Proposed.** Lists are read with hand-written PostgreSQL queries; Hibernate/JPA remains only for writing aggregates.

1. **SQL access:** `JdbcClient` with named parameters. A builder appends one fixed fragment per active filter and maps sort keys from an enum. User input is only ever bound as a parameter.
2. **Visibility as data:** a trigger-maintained `events_event_audience (audience_key, event_id)` table replaces the four-branch visibility predicate with one indexed `EXISTS`.
3. **Search:** a generated `tsvector` column (`simple` configuration) with a GIN index gives word-prefix matching from the first character; `pg_trgm` GIN indexes add substring matching from 3 characters. Frontend debounce 250 ms, `switchMap` cancels superseded requests.
4. **Deterministic order:** every sort ends with the unique `id`; each sort has a matching partial index `(sort columns, id) WHERE cancelled_at IS NULL`.
5. **Paging:** keyset (cursor) paging with `LIMIT n + 1`. The cursor is opaque and bound to the filters and sort. Offset paging is kept only for future numbered admin tables.
6. **Total counts:** a striped counter table `events_count (dimension, key, slot, n)` maintained by statement-level triggers in the writing transaction, with ordered upserts to avoid deadlocks. It covers totals that are dimensions of the data: all active events, public events, events per host, events per group.
7. **Filtered counts:** a separate exact `count(*)` over the same predicate, capped at 1000 rows, requested only on the first page or when filters change, and run in the same `REPEATABLE READ READ ONLY` snapshot as that page.
8. **Live updates:** statement-level triggers call `pg_notify` with event ids and affected audience keys (no content). Each application instance listens on one dedicated connection and pushes hints over Server-Sent Events to the sessions whose audience keys match. Clients coalesce hints and refetch from the database.
9. **No result caching:** no client-side or server-side result cache; list responses send `Cache-Control: no-store`.
10. **Tests and development on PostgreSQL:** Testcontainers for integration tests and Docker Compose for local development replace H2.

## Alternatives considered

- **Keep JPQL/Spring Data `Page` with offset:** rejected by requirement 5, cost grows with page number, and pages shift under concurrent changes.
- **JPA Criteria API or Specifications for dynamic filters:** rejected; still Hibernate-generated SQL, harder to control index use and plans.
- **jOOQ for type-safe SQL:** a reasonable choice, but a new dependency and code generation step; `JdbcClient` is enough for a handful of list queries. Can be revisited.
- **Offset paging everywhere:** rejected for "Show more" lists (see the design's keyset/offset table); kept for numbered pages.
- **Per-viewer counter rows for the member's total:** exact, but every public event would update one row per member, and group membership changes in another module would force recounts. Rejected.
- **`count(*) OVER ()` in the page query, or planner estimates, for filtered counts:** rejected; the first defeats keyset paging, the second is inaccurate for combined filters.
- **Single unstriped counter row per key:** simplest, but serialises concurrent writers on that key. Striping costs little and is chosen.
- **Insert-only delta table with periodic rollup for counts:** no lock contention at all; kept as the fallback if write volume ever demands it.
- **Trigram-only search:** cannot use an index for 1–2 character terms. **`ILIKE '%term%'` without index:** sequential scan. Both rejected as the main path.
- **WebSocket instead of SSE:** two-way is not needed; SSE works over plain HTTP with the session cookie and reconnects by itself.
- **Polling or change data capture (logical replication, Debezium):** rejected as wasteful or as an extra running component.
- **Maintaining audience rows and counters in Java instead of triggers:** possible and easier to test on H2, but any write path that forgets it (a migration, a manual fix, a future module) silently breaks visibility, counts and notifications. Triggers keep them in the same transaction for every writer.

## Consequences

- Benefits:
  - Page cost stays flat with depth; no duplicates or gaps while others write.
  - Search works from one character and stays indexed.
  - Totals are exact and read in constant time; filtered counts are exact up to the cap and never delay "Show more".
  - All sessions see others' changes within about a second, without polling and without holding database connections.
  - Visibility is enforced by one indexed predicate in every list, count and notification.
- Costs, risks and tradeoffs:
  - PostgreSQL-specific SQL and PL/pgSQL triggers; H2 can no longer run the schema, so tests need Docker.
  - Every write pays for trigger work (audience rows, counter upsert, notification).
  - Business-relevant derived data (audience, counters) lives in triggers, which must be reviewed and tested as code.
  - NOTIFY is not durable; clients must resync after reconnecting.
  - The counter cannot express a time window ("from today") or an exact per-viewer total; those are filtered counts.
  - The list API contract changes (`page` becomes `after`/`limit`, new filters, `filteredCount`); web and mock must follow.
  - The overlap window needs a longest-event rule (or a GiST range index) to stay index-friendly.

## Related artifacts

- [list-queries-design.md](../architecture/list-queries-design.md): schema, indexes, SQL, counts, live updates, performance
- [ADR-001](ADR-001-one-modular-spring-boot-app.md): module boundaries (no cross-module joins; names fetched through directories)
- [ADR-003](ADR-003-openapi-contract.md): the list contract changes
- [ADR-005](ADR-005-login-approach.md): the SSE stream relies on the session cookie
- [ADR-006](ADR-006-liquibase-migrations.md): new changesets, PostgreSQL-only `sql` changesets

## Open questions for the owner

1. **Which totals should the UI show?** Proposed: "x of y matching" in the list from the filtered count, plus counter-backed totals where they are meaningful ("My events", per group, public). The module-wide total includes private events and would be visible to administrators only.
2. **Longest event:** may we add a validation rule, for example at most 90 days per event, so the date window stays index-friendly? Otherwise a GiST range index is used.
3. **Filter set:** text, kind, host, group, my answer, series, date window. Anything missing (for example "has open bring-list items")?
4. **Filtered count cap:** 1000 proposed.

## Follow-up work

After approval, in order. Owner of each: Åsa Wegelius (review), implementation by the assigned developer.

1. Move tests to Testcontainers PostgreSQL and `dev` to Docker Compose. Acceptance: existing specs pass on PostgreSQL.
2. Liquibase changesets: missing foreign-key indexes, search column and indexes, audience table and triggers, counter table and triggers, notify triggers. Acceptance: parity and counter specs pass; drift query returns no rows.
3. `EventListQuery`, `EventCountQuery`, `EventTotals` with `JdbcClient`; remove `findVisible` and `default_batch_fetch_size`. Acceptance: `EXPLAIN` of the default list shows an index scan without a sort node on a seeded dataset of at least 100,000 events; keyset spec under concurrent writes passes.
4. Contract change in TulkaSamvaerWeb `api/openapi.yaml` and the mock; frontend search, filters, URL state, debounce.
5. `ChangeListener`, `SseHub`, `/api/v1/events/changes`; frontend `EventSource` handling. Acceptance: notification specs pass; a change appears in a second browser session within 2 seconds.
6. Load test with concurrent readers, typers and writers; record results here.

## Replacement history

None.
