# Database-driven search, filtering and paging: design

Status: **proposal** for review, belongs to [ADR-007](../decisions/ADR-007-database-driven-list-queries.md). Nothing here is implemented yet. The Events list is the first user; the pattern is meant to be reused by other sections' lists.

## 1. Starting point

What the code does today (TulkaSamvaerAPI PR 1, TulkaSamvaerWeb `feature/events-poc`):

| Area | Today | Problem against the requirements |
|:-----|:------|:---------------------------------|
| Database | PostgreSQL in production (`org.postgresql` driver); H2 in PostgreSQL mode for `dev` and tests | H2 has none of the features below (tsvector, GIN, pg_trgm, PL/pgSQL triggers, LISTEN/NOTIFY) |
| List query | JPQL `EventRepository.findVisible`, Spring Data `Page`, `OFFSET`/`LIMIT`, `order by startsAt, id` | Paging and filtering through Hibernate; offset paging |
| Count | A second JPQL count query with the same visibility predicate on every page request | Total and filtered count are the same query and are recomputed for every "Show more" |
| Summary fields | RSVP and bring-list counts computed in Java from lazy collections, loaded with `hibernate.default_batch_fetch_size: 50` | Hibernate batch fetching replaces SQL aggregation |
| Filters | Only a date window (`from`, `to`) | No search, no combined filters, one fixed sort |
| Live updates | None; the client reloads on navigation | Sessions do not see others' inserts, updates and deletes |
| Missing indexes | No index on `events_bring_item.event_id`, `events_programme_item.event_id`, `events_sleeping_spot.event_id` | PostgreSQL does not index foreign keys automatically |

## 2. Target design at a glance

```text
Angular list page
  filter form ──debounce 250 ms──► GET /api/v1/events?q=..&kind=..&sort=..&limit=20[&after=cursor][&count=true]
  "Show more"  ─────────────────► GET ...&after=<nextCursor>
  EventSource ◄── SSE ─────────── GET /api/v1/events/changes   (ids + "refresh" hints, never content)

Spring Boot (events module)
  EventListQuery (JdbcClient, whitelisted SQL fragments, bound parameters)
  EventCountQuery (filtered count, capped)
  EventTotals     (reads counter table)
  ChangeListener  (one LISTEN connection per instance) ──► SseHub (per-viewer routing)

PostgreSQL
  events_event (+ generated search_vector)        events_event_audience (who can see what)
  events_count (striped counter, trigger-maintained)
  statement-level triggers ──► pg_notify('events_changed', ids + audience keys)
```

Hibernate/JPA stays for writes of the event aggregate (create, edit with `@Version`, RSVP, bring-list), which the requirements allow. Every list, filter, sort, page and count is a hand-written SQL statement.

## 3. Schema changes

All changes are new Liquibase changesets in `db/changelog/events/` ([ADR-006](../decisions/ADR-006-liquibase-migrations.md)). PostgreSQL-specific DDL goes in `sql` changesets with `dbms: postgresql` and `splitStatements: false` for function bodies.

### 3.1 Search column

```sql
CREATE EXTENSION IF NOT EXISTS pg_trgm;

ALTER TABLE events_event
  ADD COLUMN search_vector tsvector
  GENERATED ALWAYS AS (
    setweight(to_tsvector('simple', coalesce(title, '')), 'A') ||
    setweight(to_tsvector('simple', coalesce(location, '')), 'B') ||
    setweight(to_tsvector('simple', coalesce(description, '')), 'C')
  ) STORED;
```

`simple` lowercases and splits words without language stemming, which suits a mix of Danish, English and names. Accents are kept (`unaccent` is not immutable, so it cannot feed a generated column without a wrapper function; a wrapper can be added later if needed).

### 3.2 Audience table (visibility as data)

Today the visibility rule is an `OR` of four branches with two `EXISTS` subqueries (host, all members, group match, named on the guest list). That shape is hard to index and is repeated in the count query. The proposal stores the rule's result as rows:

```sql
CREATE TABLE events_event_audience (
  audience_key uuid NOT NULL,   -- a member id, a group id, or the ALL_MEMBERS sentinel
  event_id     uuid NOT NULL REFERENCES events_event(id) ON DELETE CASCADE,
  PRIMARY KEY (audience_key, event_id)
);
CREATE INDEX ix_events_event_audience_event ON events_event_audience (event_id);
```

| Visibility | Audience rows |
|:-----------|:--------------|
| `ONLY_ME` | host |
| `ALL_MEMBERS` | host, sentinel `00000000-0000-0000-0000-000000000000` |
| `GROUPS` | host, each group id, each member with a row in `events_answer` |

A viewer's keys are `{sentinel, own member id, own group ids}`, fetched once per request from `GroupDirectory`. Visibility becomes `EXISTS (SELECT 1 FROM events_event_audience v WHERE v.event_id = e.id AND v.audience_key = ANY(:keys))`, served by the primary key. Member ids and group ids are both random UUIDs, so they do not collide.

The rows are maintained by triggers on `events_event`, `events_event_group` and `events_answer`, in the same transaction as the change. A parity test asserts that the table and `Visibility.allows` agree for every combination. The table holds ids only and joins no other module's tables, so [ADR-001](../decisions/ADR-001-one-modular-spring-boot-app.md) still holds.

### 3.3 Counter table

```sql
CREATE TABLE events_count (
  dimension varchar(10) NOT NULL,   -- ACTIVE, PUBLIC, HOST, GROUP
  key       uuid        NOT NULL,   -- sentinel for ACTIVE and PUBLIC, else member or group id
  slot      smallint    NOT NULL,   -- 0..7, see "Hot rows"
  n         bigint      NOT NULL,
  PRIMARY KEY (dimension, key, slot)
) WITH (fillfactor = 70);
```

| Dimension | Counts | Who may read it |
|:----------|:-------|:----------------|
| `ACTIVE` | all non-cancelled events | administrators (it includes private events) |
| `PUBLIC` | non-cancelled events shared with all members | every member |
| `HOST` | non-cancelled events a member hosts | that member ("My events") |
| `GROUP` | non-cancelled events shared with a group | members of that group |

Read: `SELECT coalesce(sum(n), 0) FROM events_count WHERE dimension = :d AND key = :k` — at most 8 rows from the primary key.

Maintenance: three statement-level `AFTER INSERT`, `AFTER UPDATE`, `AFTER DELETE` triggers on `events_event` (and two on `events_event_group`) with transition tables (`REFERENCING NEW TABLE AS new_rows OLD TABLE AS old_rows`). Each trigger aggregates the statement's rows into deltas and applies them in one ordered upsert:

```sql
INSERT INTO events_count (dimension, key, slot, n)
SELECT dimension, key, slot, sum(delta)
FROM ( /* +1 rows from new_rows, -1 rows from old_rows,
          for ACTIVE, PUBLIC, HOST and (via events_event_group) GROUP,
          where cancelled_at IS NULL */ ) d
GROUP BY dimension, key, slot
HAVING sum(delta) <> 0
ORDER BY dimension, key, slot
ON CONFLICT (dimension, key, slot) DO UPDATE SET n = events_count.n + EXCLUDED.n;
```

Statement level matters: creating a weekly series inserts about 104 occurrences in one statement, which becomes one upsert per affected key instead of 104 row-level updates. Cancelling (`cancelled_at` from null to a value), restoring, changing visibility and hard deletes all go through the same delta logic.

### 3.4 Change notification trigger

Statement-level `AFTER INSERT`, `AFTER UPDATE` and `AFTER DELETE` triggers (one per operation, each with the transition tables it needs) on `events_event`, `events_event_audience`, `events_answer`, `events_bring_item`, `events_claim`, `events_programme_item`, `events_sleeping_spot` call one function that sends:

```sql
PERFORM pg_notify('events_changed', json_build_object(
  'ids',  (SELECT array_agg(DISTINCT event_id) FROM changed),
  'keys', (SELECT array_agg(DISTINCT audience_key) FROM old_and_new_audience)
)::text);
```

- PostgreSQL delivers a notification only when the transaction commits, and drops identical payloads within one transaction. A rolled-back change never reaches a browser.
- The payload holds ids and audience keys only, never titles or other content.
- A payload is limited to 8000 bytes. When a statement touches more than about 100 events, the function sends `{"resync": true}` instead.

### 3.5 Indexes

Partial indexes exclude cancelled events, which the list never shows.

| Index | Serves |
|:------|:-------|
| `events_event (starts_at, id) WHERE cancelled_at IS NULL` (replaces `ix_events_event_starts_at`) | default sort "soonest first" and its keyset; also descending via a backward scan |
| `events_event (lower(title) COLLATE "da-DK-x-icu", id) WHERE cancelled_at IS NULL` | sort by title, Danish order of æ, ø, å |
| `events_event (created_at, id) WHERE cancelled_at IS NULL` | sort "newest first" |
| `events_event (host_id, starts_at, id) WHERE cancelled_at IS NULL` | filter "hosted by" plus default sort |
| `events_event USING gin (search_vector)` | word-prefix search from the first character |
| `events_event USING gin (title gin_trgm_ops)`, same for `location` | substring search inside words, 3 characters or more |
| `events_event_audience` primary key `(audience_key, event_id)` and `(event_id)` | visibility, notification routing |
| `events_answer (event_id, status)` | RSVP counts per event in the summary |
| `events_answer (member_id, status, event_id)` (replaces `ix_events_answer_member`) | filter "my answer is Going/Maybe/Can't/Pending" |
| `events_bring_item (event_id)`, `events_programme_item (event_id)`, `events_sleeping_spot (event_id)` | missing foreign-key indexes; summary counts, detail page, cascades |
| `events_count` primary key | totals |

`kind` has five values and gets no index of its own; it is applied as a filter on top of the chosen index. Each index costs write time and space, so the list is limited to the access paths above. Every index is checked with `EXPLAIN (ANALYZE, BUFFERS)` against a seeded dataset before it is kept (see follow-up work).

The list's date window is an overlap test (`ends_at >= :from AND starts_at < :to`). With only an index on `starts_at`, the database would walk every past event to find the ones that are still running. The proposal adds a longest-event rule (an open question for the owner, for example 90 days) and the predicate `starts_at >= :from - interval '90 days'`, so the index scan starts close to `:from`. The alternative is a GiST index on `tstzrange(starts_at, ends_at)`, which handles any length but cannot also deliver the sort order.

## 4. API contract

`GET /api/v1/events` (replaces the `page` parameter; contract changes per [ADR-003](../decisions/ADR-003-openapi-contract.md)):

| Parameter | Meaning |
|:----------|:--------|
| `q` | free text; every word must match as a word prefix (from 1 character) or, from 3 characters, as a substring of title or location |
| `kind` | repeatable: `GATHERING`, `BIRTHDAY`, `HOLIDAY`, `TRIP`, `OTHER` |
| `host` | member id, or `me` |
| `group` | repeatable group id (events shared with any of them) |
| `rsvp` | repeatable: the viewer's own answer |
| `series` | series id |
| `from`, `to` | date window, as today |
| `sort` | `startsAt` (default), `-startsAt`, `title`, `-createdAt` |
| `limit` | 1 to 100, default 20 |
| `after` | opaque cursor from the previous response |
| `count` | `true` adds `filteredCount` to the response |

All filters combine with AND; repeated values of one filter combine with OR.

Response:

```json
{
  "items": [ /* EventSummary, as today */ ],
  "nextCursor": "eyJzIjoic3RhcnRzQXQiLCJ2IjpbIjIwMjYtMTItMDFUMTc6MDA6MDBaIl0sImlkIjoiLi4uIiwiZiI6IjkxYzMifQ",
  "filteredCount": { "value": 57, "exact": true }
}
```

- `nextCursor` is null on the last page. It encodes the sort, the last row's sort values and id, and a hash of the filters. A cursor sent with different filters or sort answers 400, so the client restarts from the first page.
- `GET /api/v1/events/totals` answers the counter-backed totals the viewer may see (section 6.1).
- `GET /api/v1/events/changes` is the SSE stream (section 8).
- List responses carry `Cache-Control: no-store`.

## 5. SQL strategy

### 5.1 How queries are built

- Spring `JdbcClient` with named parameters, mapping rows straight into the response records. No entities are loaded for lists.
- A small builder appends a fixed SQL fragment per active filter. Absent filters add nothing, so the planner never sees `(:x IS NULL OR ...)` branches that spoil plans. Sort keys come from an enum that maps to fixed `ORDER BY` and keyset fragments. User input is only ever a bound parameter.
- The search string is normalised in Java to words of letters and digits (`[\p{L}\p{N}]+`), lowercased, and bound as one tsquery string such as `fød:* & mor:*`. Operator characters never reach `to_tsquery`.
- Each list request sets `SET LOCAL statement_timeout = '2s'` in its read-only transaction, so a pathological query cannot hold a connection.

### 5.2 Page query (default sort, with search and keyset)

```sql
SELECT e.id, e.kind, e.title, e.starts_at, e.ends_at, e.all_day, e.location,
       e.host_id, e.visibility_scope, e.series_id, e.version,
       my.status                         AS my_rsvp,
       r.going, r.maybe, r.cant, r.pending,
       b.items, b.taken
FROM events_event e
LEFT JOIN events_answer my
       ON my.event_id = e.id AND my.member_id = :viewer
LEFT JOIN LATERAL (
       SELECT count(*) FILTER (WHERE a.status = 'GOING')   AS going,
              count(*) FILTER (WHERE a.status = 'MAYBE')   AS maybe,
              count(*) FILTER (WHERE a.status = 'CANT')    AS cant,
              count(*) FILTER (WHERE a.status = 'PENDING') AS pending
       FROM events_answer a WHERE a.event_id = e.id) r ON true
LEFT JOIN LATERAL (
       SELECT count(*) AS items,
              count(*) FILTER (WHERE EXISTS (SELECT 1 FROM events_claim c WHERE c.item_id = i.id)) AS taken
       FROM events_bring_item i WHERE i.event_id = e.id) b ON true
WHERE e.cancelled_at IS NULL
  AND EXISTS (SELECT 1 FROM events_event_audience v
              WHERE v.event_id = e.id AND v.audience_key = ANY(:keys))
  AND e.ends_at >= :from AND e.starts_at < :to
  AND e.starts_at >= :from - interval '90 days'
  -- fragments below only when the filter is present
  AND e.kind = ANY(:kinds)
  AND (e.search_vector @@ to_tsquery('simple', :tsquery)
       OR e.title ILIKE :likeTerm OR e.location ILIKE :likeTerm)   -- ILIKE only for terms of 3+ characters
  -- keyset fragment only when "after" is present
  AND (e.starts_at, e.id) > (:afterStartsAt, :afterId)
ORDER BY e.starts_at, e.id
LIMIT :limit + 1;
```

- `LIMIT :limit + 1` tells whether a next page exists without counting. The extra row is not returned.
- The `LATERAL` aggregates run only for the at most 101 rows that survive the limit, using `events_answer (event_id, status)` and `events_bring_item (event_id)`.
- Host names and group names live in other modules. They are fetched with one `IN (...)` call per page through `MemberDirectory` and `GroupDirectory`, which is explicit SQL, not Hibernate batch fetching.

### 5.3 Deterministic order

Every sort ends with `e.id`, which is unique, so two rows never tie and a page boundary is always well defined. Only non-null sort columns are offered (`starts_at`, `title`, `created_at`); a nullable column would need `NULLS LAST` in both the index and the keyset predicate. Descending sorts reverse every column, for example `ORDER BY e.created_at DESC, e.id DESC` with `(e.created_at, e.id) < (:a, :b)`. The text sort uses the same collation expression in `ORDER BY`, keyset and index.

### 5.4 Keyset or offset

| | Keyset (cursor) | Offset |
|:-|:----------------|:-------|
| Cost of page N | about the same as page 1 (index seek to the cursor) | grows with N: the database reads and throws away every skipped row |
| Concurrent insert or delete before the current position | no duplicates, no skipped rows | rows shift: duplicates or gaps across pages |
| Jump to page 7, "page 3 of 12" | not possible | possible |
| Fits | "Show more", infinite scroll | numbered pages in an admin table |

The Events list uses "Show more", so the proposal is keyset. Offset stays available in the shared builder for a future admin table with numbered pages, capped at a maximum offset.

### 5.5 Search from the first character

- Word prefix (`tok:*` against the GIN tsvector index) works from one character: "f" finds "Fødselsdag" and "Farmor". This is the main path.
- Substring inside a word ("dag" in "fødselsdag") needs trigrams, which require 3 characters. From 3 characters the query adds the `ILIKE` branch, served by the trigram indexes.
- A one-character search matches many rows. The page query is still cheap because it stops after `limit + 1` rows in sort order; the planner either walks the sort index and checks the condition, or uses the GIN bitmap and sorts the matches. The filtered count is the expensive part, which is why it is capped (section 6.2).
- Results are ordered by the chosen sort, not by relevance, so they stay deterministic and keyset-pageable. A relevance sort (`ts_rank`, then `id`) can be added later as one more sort key, but it cannot use an index for its order.

## 6. Counts

The total and the filtered count are separate problems with separate mechanisms.

### 6.1 Total: transactional counter table

The counter (section 3.3) is exact, read in constant time, and correct at commit because the trigger runs in the writing transaction.

**What a counter cannot count.** A counter changes only when rows are written. Two parts of the list's "total" change without any write:

1. **Time.** The default list is "from today". Yesterday's events leave that window at midnight with no insert, update or delete.
2. **Per-viewer visibility.** An event can be visible to one viewer through several paths (all members, a group, being invited, hosting it), and a viewer's paths overlap. Adding up per-key counters counts such events twice. An exact per-viewer counter would need one row per member, updated for every member on each `ALL_MEMBERS` event and recomputed when someone joins or leaves a group in the groups module. That is a write fan-out to every member and a dependency on another module's data.

So the proposal uses the counter for totals that are true dimensions of the data (all active events, public events, events I host, events shared with a group) and shows "x of y" in the list with the filtered count. Which totals the UI should show is an open question for the owner.

**Hot rows.** Every insert into the same key updates the same counter row, and the row lock is held until commit. Concurrent writers on that key would queue behind each other, and every update leaves a dead tuple for vacuum.

- **Striping:** each key has 8 slots; a change goes to slot `abs(hashtext(event_id::text)) % 8`. Two concurrent transactions on the same key collide with a probability of about 1 in 8. The read sums at most 8 rows.
- **Deadlocks:** one transaction can touch several keys (an event shared with three groups). The upsert sorts by `(dimension, key, slot)`, so all writers take locks in the same order.
- **Bloat:** `n` is not indexed, so updates are HOT updates that stay on the same page; `fillfactor = 70` leaves room for them. Autovacuum thresholds for this small table are set low.
- **Drift check:** a scheduled or on-demand job compares each counter with `count(*)` and reports differences. Drift can only come from bypassing triggers (for example `TRUNCATE` or `session_replication_role = replica`).
- **Alternative if writes ever grow a lot:** an insert-only delta table (one `+1`/`-1` row per change, no updates, no lock contention) plus a periodic job that rolls deltas into the counter. Not needed at Samvær's size; recorded as a fallback.

### 6.2 Filtered count: separate, capped, exact

```sql
SELECT count(*) AS n FROM (
  SELECT 1 FROM events_event e
  WHERE <same WHERE clause as the page query, without the keyset fragment>
  LIMIT :cap + 1            -- cap = 1000
) s;
```

- Asked for only with `count=true`: on the first page and when filters change, not on "Show more".
- Runs in the same `REPEATABLE READ READ ONLY` transaction as the first page, so page and count come from one snapshot and cannot contradict each other.
- Uses the same indexes as the page query. Without `ORDER BY` and the `LATERAL` joins it can often use a bitmap or index-only scan.
- Above the cap it answers `{ "value": 1000, "exact": false }` and the UI shows "more than 1000". This bounds the worst case (a one-character search over a long history) while staying exact for every realistic family-sized result.

Rejected for filtered counts:

- `count(*) OVER ()` in the page query: computes every matching row on every page, which removes the benefit of keyset paging.
- Planner estimates (`EXPLAIN` row estimates, `pg_class.reltuples`): fast but often far off for combined filters and free text.
- Counter rows per filter combination: the combinations are unbounded once free text is involved.

## 7. Frontend behaviour

- The search field emits on every keystroke; the stream applies `debounceTime(250)`, trims, `distinctUntilChanged`, then `switchMap`, which cancels the previous HTTP request when a new one starts. There is no minimum length: one character searches.
- Filters live in the URL query parameters, so reload and back keep them.
- No `shareReplay`, store-level result cache or HTTP cache. The list held on screen is display state, replaced on every new query and refreshed on change hints.
- Changing a filter or sort clears the list and the cursor and asks for `count=true`.

Cancelling the HTTP request does not stop a query already running in PostgreSQL; the 2-second statement timeout and the cheap page query bound that cost.

## 8. Live updates for active sessions

### 8.1 Path of a change

1. A write commits. The trigger (section 3.4) has queued `pg_notify('events_changed', {ids, keys})`; PostgreSQL delivers it at commit.
2. Each application instance holds one dedicated JDBC connection outside the Hikari pool with `LISTEN events_changed`, read by a virtual thread through `PGConnection.getNotifications(timeout)`. Several instances each listen, so no message broker is needed.
3. `SseHub` keeps, per open SSE connection, the viewer's audience keys (authorisation state, not result data). It forwards the event ids only to connections whose keys intersect the payload's `keys`, which include both old and new audience, so a viewer who just lost access is also told.
4. The browser receives `event: events.changed` with `data: {"ids": [...]}`.

### 8.2 How the client reacts

- Hints are coalesced for 1 second, plus a random 0–500 ms delay so many sessions do not refetch at the same instant.
- List page: re-run the current query from the top with `limit` equal to the number of loaded rows (at most 100), and `count=true`. New matches appear in place, deleted or cancelled ones disappear, edits show. The client never patches rows from the push itself.
- Detail page: if its id is in the hint, refetch it. If the host's edit form is open and `version` changed, show "This event was changed by someone else".
- After reconnecting, or on `{"resync": true}`, refetch everything on screen. Notifications sent while a listener was disconnected are lost (NOTIFY is not durable), so the server also sends `resync` to all its sessions after its LISTEN connection reconnects.
- When a member joins or leaves a group, the groups module publishes a domain event; `SseHub` refreshes that member's keys and sends `resync`.

### 8.3 Why SSE

- One direction (server to browser) is all that is needed; requests still go through the REST API.
- Plain HTTP: works with the session cookie from [ADR-005](../decisions/ADR-005-login-approach.md), proxies and HTTP/2; `EventSource` reconnects on its own.
- A heartbeat comment every 25 seconds keeps proxies from closing idle streams.
- WebSocket is the choice only if two-way messaging (for example chat) arrives; the same `SseHub` routing can then move behind it.

Rejected: polling with `updated_at > :since` (needs tombstones for deletes, polls when nothing changes), and change data capture through logical replication or Debezium (an extra running component for a small system).

## 9. Performance and concurrent users

Expectations below are design reasoning, not measurements. They are to be confirmed with `EXPLAIN (ANALYZE, BUFFERS)` and a load test on a seeded PostgreSQL (follow-up work).

| Situation | Expected effect |
|:----------|:----------------|
| Page read | index seek plus at most `limit + 1` rows and their small aggregates; cost does not grow with page number |
| Many readers | MVCC: readers never block writers or each other; each request holds a pool connection for one short read-only transaction |
| Typing in search | at most about 4 requests per second per typing user after debounce, each cancelled when superseded; a filtered count only when the filter changes |
| One-character search | page stays cheap; filtered count bounded by the cap of 1000 rows |
| Write path | each write pays for the audience, counter and notify triggers: a few extra index updates and one upsert per affected counter key per statement |
| Concurrent writers on the same key | serialised only within one of 8 slots; no deadlocks thanks to ordered upserts |
| Series creation (about 104 rows) | statement-level triggers: one aggregate upsert and one notification, not 104 |
| Open SSE connections | no database connection per session; one LISTEN connection per instance; with virtual threads (`spring.threads.virtual.enabled`) an open stream costs memory, not a platform thread |
| A change seen by many sessions | each refetches once after coalescing and jitter; every refetch is a page query of at most 100 rows |
| No result cache | repeated queries rely on PostgreSQL's shared buffers; the hot part (recent events, indexes) stays in memory at this data size |

## 10. Testing consequences

H2 cannot run tsvector, GIN or trigram indexes, PL/pgSQL triggers or LISTEN/NOTIFY. Integration tests and the `dev` profile therefore move to PostgreSQL: Testcontainers in tests (one container per test run, Liquibase applied once) and Docker Compose for local development. Specs to add:

- parity between `events_event_audience` and `Visibility.allows`;
- counter equals `count(*)` after inserts, cancels, restores, visibility changes, series creation and concurrent writers;
- keyset paging under concurrent inserts and deletes returns no duplicates and no gaps;
- every sort is total (no ties) and matches its index (`EXPLAIN` shows an index scan, no sort node, for the default list);
- notifications arrive only after commit, never after rollback, and only for viewers who can see or could see the event.
