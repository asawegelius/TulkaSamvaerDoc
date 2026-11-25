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
2. **Visibility as data:** a trigger-maintained `events_event_audience (audience_key, event_id)` table replaces the four-branch visibility predicate with one indexed `EXISTS`. Access to a `GROUPS` event comes from the groups or an explicit access grant, never from an answer alone. When the host narrows visibility, a preview lists who loses access, the host picks who keeps it, answers stay as history, lost "Going" answers are cancelled and the members are notified (design 3.2.1).
3. **Search:** every search word must match on its own, as a word prefix from the first character (generated `tsvector`, `simple` configuration, GIN index) or as a substring from 3 characters (`pg_trgm` GIN indexes). Frontend debounce 250 ms inside `switchMap`, so each keystroke cancels the request in flight and stale responses are never shown.
4. **Deterministic order:** every sort ends with the unique `id`; each sort has a matching partial index `(sort columns, id) WHERE cancelled_at IS NULL`.
5. **Paging:** keyset (cursor) paging. A materialized CTE selects at most `limit + 1` ids first; RSVP and bring-list aggregates run only for the returned rows. The cursor is opaque and bound to the filters and sort. Offset paging is kept only for future numbered admin tables.
6. **Total counts:** a striped counter table `events_count (dimension, key, slot, n)` maintained by statement-level triggers in the writing transaction, with ordered upserts to avoid deadlocks. It covers totals that are dimensions of the data: all active events, public events, events per host, events per group.
7. **Filtered counts:** a separate exact `count(*)` over the same predicate, capped per list (1000 for Events, configurable; no global cap), requested only on the first page or when filters change, and run in the same `REPEATABLE READ READ ONLY` snapshot as that page.
8. **Live updates:** statement-level triggers call `pg_notify` with event ids and affected audience keys (no content), falling back to `resync` when the finished payload reaches 7900 bytes. Each application instance listens on one dedicated connection and pushes hints over Server-Sent Events to the sessions whose audience keys match. Clients coalesce hints and refetch from the database.
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
  - Events may last at most 90 days, so the overlap window stays index-friendly; a longer stay is entered as two events.

## Related artifacts

- [list-queries-design.md](../architecture/list-queries-design.md): schema, indexes, SQL, counts, live updates, performance
- [ADR-001](ADR-001-one-modular-spring-boot-app.md): module boundaries (no cross-module joins; names fetched through directories)
- [ADR-003](ADR-003-openapi-contract.md): the list contract changes
- [ADR-005](ADR-005-login-approach.md): the SSE stream relies on the session cookie
- [ADR-006](ADR-006-liquibase-migrations.md): new changesets, PostgreSQL-only `sql` changesets

## Owner answers (2026-11-25)

1. **Totals:** the list shows "x of y", where y is the filtered count. Counter-backed totals cover "My events", per group and public events; the module-wide total is for administrators only.
2. **Longest event:** at most 90 days per event, validated in the API and enforced by a check constraint. No automatic retention for now; a member can delete their own events that ended more than x days ago (design section 6.3). GDPR for friends invited by friends is a later issue.
3. **Filter set:** text, kind, host, group, my answer, series and date window are enough.
4. **Filtered count cap:** 1000 for Events. The cap is set per list and configurable, never global.

## Follow-up work

After approval, in order. Owner of each: Åsa Wegelius (review), implementation by the assigned developer.

1. Requirements: add the 90-day rule, "delete my events older than x days" and the access rule for narrowing visibility (preview, keep access, cancel and notify) as Events requirements, and open an issue for GDPR handling of invited friends' data.
2. Move tests to Testcontainers PostgreSQL and `dev` to Docker Compose. Acceptance: existing specs pass on PostgreSQL.
3. Liquibase changesets: missing foreign-key indexes, 90-day check constraint, `ON DELETE CASCADE` on child foreign keys, search column and indexes, audience table and triggers, counter table and triggers, notify triggers. Acceptance: parity and counter specs pass; drift query returns no rows.
4. `EventListQuery`, `EventCountQuery`, `EventTotals` and the batched "delete older than" endpoint with `JdbcClient`; remove `findVisible` and `default_batch_fetch_size`. Acceptance: `EXPLAIN` of the default list shows an index scan without a sort node on a seeded dataset of at least 100,000 events; `EXPLAIN (ANALYZE, BUFFERS)` of a common and a rare one-character search each stays under 50 ms and 2,000 buffers on that dataset (otherwise the GIN-first fallback in design 5.2 is built); search contract, keyset and cursor specs pass.
5. Contract change in TulkaSamvaerWeb `api/openapi.yaml` and the mock; frontend search, filters, URL state, debounce.
6. `ChangeListener`, `SseHub`, `/api/v1/events/changes`; frontend `EventSource` handling. Acceptance: notification specs pass; a change appears in a second browser session within 2 seconds.
7. Load test with concurrent readers, typers and writers; record results here.

## Replacement history

None.
