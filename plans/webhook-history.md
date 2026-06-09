# Webhook History & Performance Admin View

## Context

The imbi-gateway receives webhook notifications from third-party
services (GitHub, PagerDuty, etc.) and dispatches them through a
synchronous handler pipeline. Each successful match is already
recorded as one row per matched project in the ClickHouse `events`
table by `imbi_gateway.notifications._record_events`. Today there
is no UI to browse this history.

Admins need a way to answer questions like "did our PagerDuty
webhook reach Imbi at 2:14am?", "which projects did this push fan
out to?", and "did the release-handler succeed on the last 5
deliveries?". The intent is a new admin view that lists recent
webhook notifications and shows how each one was dispatched (which
handlers ran and whether they succeeded), styled and structured to
feel like the existing Operations Log so the operational vocabulary
stays consistent. Specific events must also be shareable via
deep-link URLs.

## Decisions

- **"Disposition" → per-handler outcomes captured in metadata.**
  Each event row carries a `handlers` array of
  `{handler, status, error?, duration_ms?}` objects so the UI can
  render "N handlers ok / M total" without per-TPS logic.

- **HARD INVARIANT: an event row is always recorded when an Imbi
  project is matched.** The current `_record_events` call site —
  immediately after project matching, *before* filter eval and
  handler dispatch — does not move. If handlers crash, time out,
  or the request is cancelled, the row is still on disk. This
  preserves the existing best-effort guarantee.

- **Two-phase recording, coalesced through a ClickHouse view.**
  - Phase 1 (current behavior): immediately after match, insert
    a row with `metadata = {webhook_id, headers}`,
    `handlers = []`, and `version = 0`.
  - Phase 2 (new): after `_run_handlers` returns, insert a second
    row with the same `id` and a populated `handlers` array, at
    `version = 1`. Both rows live in the underlying `events`
    table forever; storage growth is bounded (≤ 2x event count).
  - The phase-2 insert is wrapped in `try/except` like phase 1;
    if it fails the phase-1 row is the source of truth.
  - Engine choice (revised after review): the `events` table uses
    `ReplacingMergeTree(version)`, so background merges collapse the
    phase-0/phase-1 pair to the highest-`version` row per event. The
    `events_latest` view is kept on top for read-after-write
    correctness in the window before a merge runs. (An earlier draft
    kept plain `MergeTree` and relied on the view alone, to avoid
    touching the engine/sort key; that was reversed per reviewer
    feedback — see imbi-common #159.)

- **Schema changes on `events`.**
  - Engine becomes `ReplacingMergeTree(version)`; sort key becomes
    `(project_id, id)` so dedup keys on the event id. On an existing
    cluster this needs the manual rebuild in the deployment playbook
    below — neither the engine nor the leading sort key can be
    altered in place.
  - Add column `version UInt8 DEFAULT 0`.
  - Create a sibling view `events_latest` (regular view, not
    materialized) that selects the highest-`version` row per
    `id`:
    ```sql
    CREATE VIEW events_latest AS
    SELECT * EXCEPT (rn) FROM (
      SELECT *,
             row_number() OVER (
               PARTITION BY id
               ORDER BY version DESC, recorded_at DESC
             ) AS rn
      FROM events
    ) WHERE rn = 1;
    ```
  - Existing rows get `version = 0` for free via the column
    default; the view treats them as already-final (single row
    per id).

- **Query-side: all reads go through `events_latest`.**
  `_list_impl` and the new by-id endpoint read from
  `events_latest` instead of `events`. The pagination cursor
  scheme (`(recorded_at, id) < ...`) is unchanged because the
  view exposes the same columns and ordering. Writes still go to
  the underlying `events` table.

- **UI/Gateway coupling — generic rendering for v1.** We do not
  extend `/operations-log/plugin-templates`; those templates are
  scoped to API plugins (a different registry). The UI renders
  recorded fields generically (`{third_party_service} · {type}`
  plus an "N of M handlers ok" badge). Per-`(TPS, event_type)`
  friendly labels are an additive future enhancement.

- **Deep links to individual events.**
  Each row in the stream is addressable. URL pattern follows the
  existing admin query-param scheme:
  `/admin?section=webhook-history&event=<event-id>`. Loading that
  URL fetches the single event via a new endpoint and pins it at
  the top of the view with its detail panel expanded. Each
  rendered row exposes a copy-link affordance for sharing.

- **New endpoint: `GET /events/{event_id}`.**
  Returns a single `EventRecord` (coalesced via the same query
  rewrite). Gated on `admin:events:read`. Required to support
  deep-link landings where the target event is older than the
  current cursor page.

- **Placement: new section inside `Admin.tsx`** (sidebar entry +
  conditional render), gated by `<AdminProtectedRoute>`. No
  sub-router introduced.

- **Filter set: event type, third-party service, project, time
  range.** Time-range picker mirrors Operations Log
  (24h / 7d / 30d / all) by setting `since`. Filters round-trip
  via URL query string. Free-text search is out of scope for v1.

- **Server-side filtering via the existing `/events` endpoint,
  with one new query parameter.** Add `third_party_service` to
  the public signature of `list_events` (the implementation
  already filters on it via `_FILTER_FIELDS`).

- **Detail expand inline** showing `metadata.handlers`,
  `metadata.webhook_id`, redacted headers, and a collapsible raw
  `payload` viewer. No separate detail page.

## Critical files

### `imbi-common/` (schema + view)

- `src/imbi_common/clickhouse/schemata.toml` (≈ lines 4–18):
  - Add column `version UInt8 DEFAULT 0` to the `events`
    definition. Change the engine to
    `{replicated}ReplacingMergeTree(version)` and the sort key to
    `(project_id, id)`; partitioning stays `toYYYYMM(recorded_at)`.
    (For fresh installs the `CREATE … IF NOT EXISTS` applies these;
    existing clusters need the rebuild playbook below.)
  - Add a `events_latest` view definition alongside the table:
    a regular `CREATE VIEW` that returns one row per `id` using
    `row_number() OVER (PARTITION BY id ORDER BY version DESC,
    recorded_at DESC) = 1`. Verify the convention used in
    `schemata.toml` for declaring views vs. tables during
    implementation — if views aren't yet supported in the
    schema-driver, add minimal support (or fall back to a
    dedicated migration step).
  - `metadata.handlers` is a free-form JSON array; no separate
    column required.
- Migration step: the additive parts are idempotent — `ALTER TABLE
  events ADD COLUMN IF NOT EXISTS version UInt8 DEFAULT 0` plus a
  `CREATE VIEW IF NOT EXISTS events_latest AS ...` (existing rows
  acquire `version = 0` automatically). The engine/sort-key change
  is **not** idempotent and cannot be applied in place on an
  existing cluster — that requires the rebuild in the deployment
  playbook below.
- `src/imbi_common/models.py` — `Event` (≈ 896–921):
  `metadata` is already `dict[str, Any]` so no model change for
  `metadata.handlers`. Add `version: int = 0` so producers can
  set it explicitly on the phase-2 insert (the column default
  covers phase-1 callers that don't set it).

### `imbi-gateway/` (two-phase recording)

- `src/imbi_gateway/notifications.py`
  - `_record_events` (≈ 850–889): generalize to accept
    `version: int` and `handlers: list[HandlerOutcome] | None`.
    Phase-1 call passes `version=0, handlers=None`; phase-2
    call passes `version=1, handlers=outcomes`.
  - `process_notification` (≈ 132–336): leave the existing
    `_record_events` call where it is (≈ line 280) — this is the
    phase-1 write that guarantees recording on match. After
    `_run_handlers` returns (≈ after line 333) call a new helper
    that builds a second `_record_events` invocation with the
    collected outcomes and `version=1`.
  - `_run_handlers` (≈ 307–556): instrument the per-rule loop to
    collect a `HandlerOutcome` for each handler. Status from the
    existing try/except (≈ 538–556); `duration_ms` from
    `time.monotonic()` deltas around the awaited callable
    (line 531). Return the outcomes list (or accept a list to
    append into) without changing the surrounding control flow.
- `tests/test_notifications.py`: cover the two-phase shape, the
  partial-fail case, and the early-return paths (no match → no
  insert; match-but-no-rules → phase-1 row only with
  `handlers = []`).

### `imbi-api/` (filter + by-id endpoint + coalesced read)

- `src/imbi_api/endpoints/events.py`
  - `list_events` (≈ 142–177): add
    `third_party_service: str | None = None` to the public
    signature and pass through `query_filters`. `_FILTER_FIELDS`
    already includes it.
  - `_list_impl` (≈ 68–139): change `FROM events` to
    `FROM events_latest`. No other query change — the view
    exposes the same columns and ordering keys, so the
    `(recorded_at, id) < ...` cursor scheme keeps working
    unchanged.
  - New endpoint: `GET /events/{event_id}` → `EventRecord`.
    Reads from `events_latest` filtered by `id`. Gated on
    `admin:events:read`. 404 when no row exists for that id.
  - `EventRecord`: unchanged. `metadata` is `dict[str, Any]` so
    `handlers` passes through transparently.
- `tests/endpoints/test_events.py`:
  - `?third_party_service=` filtering.
  - Coalesced read: insert two rows with the same id and
    different `version`; assert the higher-version metadata is
    returned.
  - `GET /events/{id}` happy path + 404 + permission gating.

### `imbi-ui/` (admin section + view + deep link)

- `src/types/api-generated.ts`: regenerate via
  `npm run codegen:fetch` after the API changes land. The user
  owns the regeneration step.
- `src/components/admin/Admin.tsx`
  - Add `'webhook-history'` to the `AdminSection` union and
    `VALID_SECTIONS` (≈ 52–94).
  - Add a sidebar entry to `orgAdminSections` (≈ 122–194) with a
    `Webhook` icon and a one-line description.
  - Conditional render block (≈ 380–415):
    `{currentSection === 'webhook-history' && <WebhookHistory />}`.
- `src/components/admin/WebhookHistory.tsx` (new): top-level
  view. On mount, reads `event` from URL query params; when
  set, fetches `GET /events/{id}` and renders it as a pinned
  card above the stream with its detail panel pre-expanded.
  Composes a toolbar (filters), the stream, and infinite-scroll
  pagination.
- `src/components/admin/WebhookHistoryRow.tsx` (new): summary
  row + detail panel.
  - Summary: `recorded_at`, `{third_party_service} · {type}`
    badge, project slug (via `useProjects`), `attributed_to`,
    derived disposition badge —
    `metadata.handlers` empty → "Recorded · no handlers";
    all `status === 'succeeded'` → green "N handlers · ok";
    any failure → red "N of M handlers ok".
  - Detail: handler list with status pills, `webhook_id`,
    redacted headers, collapsible raw `payload`.
  - Includes a "copy link" button that copies
    `<origin>/admin?section=webhook-history&event=<id>` to the
    clipboard.
- `src/hooks/useInfiniteWebhookEvents.ts` (new): mirror of
  `useInfiniteOperationsLog.ts`. `useInfiniteQuery` against
  `/events`. Page size 100. Decodes `Link` header for the
  next-cursor. Filters (`type`, `third_party_service`,
  `project_id`, `since`) form part of the query key.
- `src/hooks/useWebhookEvent.ts` (new): one-shot `useQuery`
  against `GET /events/{id}` for the deep-link case.
- Filter dropdown sources:
  - **Projects:** existing `getProjects(orgSlug)` (already used
    by Operations Log toolbar).
  - **Third-party services:** the org-scoped TPS list endpoint
    (verify the hook name during implementation; add
    `useTpsList` if missing).
  - **Event types:** facet from the loaded pages, matching
    Operations Log's `OperationsLog.tsx:247–299` pattern.
- URL state: filter values and the optional `event` deep-link
  id all live in the query string so any shared URL replays
  the same view.

## Implementation outline (dependency-ordered)

1. **imbi-common** — schema: set the `events` engine to
   `ReplacingMergeTree(version)` and sort key to `(project_id, id)`,
   add `version UInt8 DEFAULT 0`, and create the `events_latest`
   view that selects one row per `id` by highest `version`. Update
   the `Event` model. On fresh ClickHouse the `CREATE … IF NOT
   EXISTS` applies it all; the additive `ALTER`/view replay cleanly
   on a cluster with existing rows, but the engine/sort-key change
   there is the manual rebuild (deployment playbook below).
2. **imbi-gateway** — two-phase recording: phase 1 stays where
   it is, phase 2 inserts after `_run_handlers`; `_record_events`
   accepts `version` and `handlers`. Tests cover the
   always-recorded invariant.
3. **imbi-api** — coalesced read in `_list_impl`; expose
   `third_party_service` query parameter; add
   `GET /events/{event_id}`. Tests.
4. **imbi-ui** — regenerate API types, add the `webhook-history`
   admin section, scaffold `WebhookHistory.tsx`,
   `WebhookHistoryRow.tsx`, `useInfiniteWebhookEvents.ts`,
   `useWebhookEvent.ts`. Wire filters and the `event` deep-link
   parameter into the URL. Add the "copy link" affordance.

Gateway can land independently of UI (UI's disposition rendering
just shows "Recorded · no handlers" for legacy rows that lack the
phase-2 write). API → UI dependency is the usual one (types
regenerated from the API's OpenAPI snapshot).

## Tests

### imbi-common

- Schema test: `schemata.toml` declares the `events` engine as
  `{replicated}ReplacingMergeTree(version)` with sort key
  `(project_id, id)`, the `version UInt8 DEFAULT 0` column, and the
  `events_latest` view. (Matches `test_events_engine` /
  `EventsLatestViewSchemaTestCase` in imbi-common.)
- Additive-migration test: replaying `events_add_version` + the
  view on a table with pre-existing rows preserves all rows and
  assigns `version = 0`.
- View correctness test: insert two rows with the same `id`
  and `version = 0, 1`; `SELECT * FROM events_latest WHERE id = ?`
  returns exactly the `version = 1` row.

### imbi-gateway

- `test_notifications.py`:
  - Match-with-no-rules → phase-1 row only,
    `metadata.handlers = []`.
  - Match-with-rules, all succeed → two rows with the same id;
    phase-2 row has `version = 1` and a populated `handlers`
    list, every entry `status: 'succeeded'`.
  - Match-with-rules, one handler raises → phase-2 row has a
    `failed` entry with an `error` string; the phase-1 row is
    still present.
  - Handler dispatch panic (simulated `asyncio.CancelledError`
    after phase-1 insert) → phase-1 row is still present;
    no phase-2 row.
  - No match → no insert at all (existing behavior).

### imbi-api

- `test_events.py`:
  - `?third_party_service=` filters as expected.
  - Listing reads through `events_latest`: with both a
    `version = 0` and `version = 1` row for the same id present
    in the underlying `events` table, the response contains a
    single row with the version-1 metadata.
  - `GET /events/{id}` happy path returns version-1 metadata.
  - `GET /events/{id}` 404 for unknown ids.
  - Admin permission gating on both endpoints.

### imbi-ui

- `WebhookHistoryRow` snapshot/RTL across three disposition
  states (no handlers / all ok / partial fail).
- `useInfiniteWebhookEvents` cursor handling
  (decoding `Link` → next-page query key).
- Toolbar filters round-trip via URL query string.
- Deep-link landing: visiting
  `?section=webhook-history&event=<id>` renders a pinned card
  with the detail panel expanded; copy-link button writes the
  expected URL to the clipboard.

## Verification

End-to-end spot-checks once each submodule has its changes:

1. **Migration & two-phase insert** —
   ```bash
   cd imbi-gateway && just serve &
   curl -fsS -X POST http://localhost:<gw-port>/notifications/<id> \
        -H 'X-GitHub-Event: push' -d @sample-push.json
   ```
   In ClickHouse, immediately after the request, the underlying
   table holds both phases:
   ```sql
   SELECT id, version, JSONExtractRaw(metadata, 'handlers')
   FROM events
   WHERE id = '<id-from-row>'
   ORDER BY version;
   ```
   Expect two rows (versions 0 and 1). The view collapses them:
   ```sql
   SELECT id, version, JSONExtractRaw(metadata, 'handlers')
   FROM events_latest
   WHERE id = '<id-from-row>';
   ```
   Expect exactly one row, with `version = 1` and the populated
   `handlers` array.
2. **Always-recorded invariant** — temporarily inject a handler
   that raises immediately and re-send the same webhook.
   Confirm a `version = 0` row exists in `events` for the
   project even when no `version = 1` row is written; the view
   surfaces that same `version = 0` row as the row's current
   state.
3. **API filter + coalesced read** —
   ```bash
   curl -fsS -H "Authorization: Bearer $ADMIN_TOKEN" \
        "http://localhost:8000/events?third_party_service=github-enterprise-cloud&limit=5" \
        | jq '.data | length, .data[0].metadata.handlers'
   ```
   Returned rows should reflect the version-1 metadata exclusively.
4. **Deep-link endpoint** —
   ```bash
   curl -fsS -H "Authorization: Bearer $ADMIN_TOKEN" \
        "http://localhost:8000/events/<event-id>" | jq
   ```
   Returns one `EventRecord`; 404 on a fake id.
5. **UI** — log in as admin, open `/admin?section=webhook-history`:
   - Recent deliveries appear, most-recent first.
   - Filters (TPS, event type, project, time range) narrow the
     list and update the URL.
   - Expanding a row shows the handler list with status pills
     and the raw payload.
   - The row's copy-link button produces a URL of the form
     `/admin?section=webhook-history&event=<id>`.
   - Opening that URL in a fresh tab lands directly on the
     event with its detail panel pre-expanded, even when the
     event is past the default cursor page.
   - Infinite scroll loads the next page near the bottom.
6. **Per-submodule gates** before opening PRs:
   - `imbi-common`: lint + test.
   - `imbi-gateway`: `just lint && just test`.
   - `imbi-api`: `just lint && just test`.
   - `imbi-ui`: lint + test.

## Deployment — events table migration playbook

This feature changes the `events` engine to
`ReplacingMergeTree(version)` and its sort key to `(project_id, id)`
so the phase-0/phase-1 writes coalesce. On a **fresh** install
`setup_schema()` creates the table correctly. On an **existing**
cluster it does not — and that path needs the manual rebuild below.

**Why it's manual, not a `setup_schema()` run.** The `events` table
predates this feature and already exists on deployed clusters with
`ENGINE = {replicated}MergeTree()` / `ORDER BY (project_id,
recorded_at)`. ClickHouse cannot change a table's **engine** or its
**leading sort key** in place (`ALTER … MODIFY ORDER BY` only
*appends* columns), and every `schemata.toml` statement is
`CREATE … IF NOT EXISTS`, so re-running `setup_schema()` is a **no-op
for the existing table**. There is also no automated schema-apply
step: `setup_schema()` is only invoked by the interactive `imbi-api
setup` wizard (`entrypoint.py:242`, via `just bootstrap` → `kubectl
exec -it … imbi-api setup`); the Helm chart has no migration
`Job`/hook. The additive pieces (the `version` column via
`events_add_version`, the `events_latest` view) *can* ride along on a
`setup_schema()` run; the engine/sort-key change must be a hand-run
rebuild, per cluster.

**Drop-and-recreate is not acceptable** — `events` is not disposable.
It backs the dashboard webhook-volume series (`dashboard.py:323`,
`FROM events WHERE recorded_at >= …`) and is the source for the
`pull_requests` materialized view. (Deployment records live in
`operations_log` + the AGE graph, which this migration doesn't touch.)
So we rebuild while preserving rows.

**Placeholders.** Resolve as `setup_schema()` does: clustered →
`{on_cluster}` = `ON CLUSTER '<name>'`, `{replicated}` = `Replicated`
(engine `ReplicatedReplacingMergeTree(version)`); single-node → both
empty (engine `ReplacingMergeTree(version)`, no `ON CLUSTER`). Run via
`clickhouse-client` against the cluster.

**Pre-flight.** Confirm clustered vs. single node (`echo
$CLICKHOUSE_CLUSTER_NAME`). `events` writes from imbi-gateway are
best-effort — scale the gateway to 0 for the window for a clean cut,
or accept a small write gap; the swap is atomic, so imbi-api reads via
`events_latest` stay up throughout.

**Steps** (clustered shown; drop `ON CLUSTER …` and use the
non-`Replicated` engine for single node):

1. Ensure the discriminator column exists (idempotent; lets the
   backfill `SELECT version`):
   ```sql
   ALTER TABLE imbi.events ON CLUSTER '<name>'
     ADD COLUMN IF NOT EXISTS version UInt8 DEFAULT 0;
   ```
2. Create the rebuilt table under a temporary name:
   ```sql
   CREATE TABLE imbi.events_v2 ON CLUSTER '<name>' (
     id                   String           DEFAULT '',
     project_id           LowCardinality(String),
     recorded_at          DateTime64(3, 'UTC'),
     type                 LowCardinality(String) DEFAULT '',
     third_party_service  LowCardinality(String) DEFAULT '',
     attributed_to        LowCardinality(String) DEFAULT '',
     metadata             JSON,
     payload              JSON,
     version              UInt8 DEFAULT 0
   ) ENGINE = ReplicatedReplacingMergeTree(version)
   PARTITION BY toYYYYMM(recorded_at)
   ORDER BY (project_id, id);
   ```
3. Backfill (run on **one** replica; data replicates via Keeper). This
   does **not** fire `pull_requests_mv` — that MV triggers on inserts
   into `imbi.events`, not `events_v2`:
   ```sql
   INSERT INTO imbi.events_v2
   SELECT id, project_id, recorded_at, type, third_party_service,
          attributed_to, metadata, payload, version
   FROM imbi.events;
   ```
   For a very large table, backfill partition-by-partition
   (`… WHERE toYYYYMM(recorded_at) = <YYYYMM>`) to bound memory/time.
4. Drop the materialized view **before** the swap. `DROP TABLE` on a
   `TO`-target MV removes only the trigger, **not** the
   `pull_requests` data, so materialized rows are preserved:
   ```sql
   DROP TABLE IF EXISTS imbi.pull_requests_mv ON CLUSTER '<name>';
   ```
   (An MV's trigger is bound to its source storage; after `EXCHANGE`
   it can stay attached to the *old* storage. Recreating it after the
   swap guarantees it fires on the new `imbi.events`.)
5. Atomic swap (requires the Atomic database engine — the modern
   default):
   ```sql
   EXCHANGE TABLES imbi.events AND imbi.events_v2 ON CLUSTER '<name>';
   ```
   `imbi.events` is now the rebuilt table; `imbi.events_v2` is the old
   table — keep it as the rollback backup.
6. Recreate the MV against the new `imbi.events` (verbatim from
   `schemata.toml [pull_request_mv]`, placeholders resolved):
   ```sql
   CREATE MATERIALIZED VIEW IF NOT EXISTS imbi.pull_requests_mv
   ON CLUSTER '<name>' TO imbi.pull_requests AS
   SELECT
       project_id,
       CAST(payload.pull_request.id, 'String')        AS pr_id,
       CAST(payload.pull_request.number, 'UInt32')    AS pr_number,
       CAST(payload.pull_request.title, 'String')     AS title,
       CAST(payload.pull_request.html_url, 'String')  AS url,
       CAST(payload.pull_request.state, 'String')     AS state,
       CAST(payload.pull_request.user.login, 'String') AS author,
       CAST(payload.pull_request.draft, 'Bool')       AS draft,
       CAST(payload.pull_request.merged, 'Bool')      AS merged,
       parseDateTime64BestEffort(
           CAST(payload.pull_request.created_at, 'String')) AS created_at,
       parseDateTime64BestEffort(
           CAST(payload.pull_request.updated_at, 'String')) AS updated_at,
       if(isNotNull(payload.pull_request.merged_at),
           parseDateTime64BestEffort(
               CAST(payload.pull_request.merged_at, 'String')),
           NULL)                                      AS merged_at,
       CAST(payload.pull_request.additions, 'UInt32')     AS additions,
       CAST(payload.pull_request.deletions, 'UInt32')     AS deletions,
       CAST(payload.pull_request.changed_files, 'UInt32') AS changed_files,
       recorded_at
   FROM imbi.events
   WHERE type = 'pull_request'
     AND third_party_service = 'github-enterprise-cloud'
     AND CAST(payload.action, 'String') IN ('opened', 'closed', 'reopened');
   ```
   `events_latest` is a plain (non-materialized) view that resolves its
   source by name at query time, so it reads the new table after the
   swap with no action. (Create it from `schemata.toml [events_latest]`
   if this cluster doesn't have it yet.)
7. Resume gateway writes if you paused them.

**Verification.**
```sql
SELECT count() FROM imbi.events;       -- rebuilt
SELECT count() FROM imbi.events_v2;    -- old backup; should match (historical rows are all version 0)
SHOW CREATE TABLE imbi.events;         -- confirm ReplacingMergeTree(version) + ORDER BY (project_id, id)
OPTIMIZE TABLE imbi.events FINAL;      -- force-collapse
SELECT count() AS rows, uniqExact(id) AS ids FROM imbi.events;  -- rows == ids after FINAL
SELECT count() FROM imbi.events_latest;
```
Optional two-phase smoke test: insert a `version=0` then `version=1`
row with the same `id`; `events_latest` must return the `version=1`
row, and after `OPTIMIZE … FINAL` the base table keeps only it.

**Rollback** (before dropping the backup): `EXCHANGE TABLES
imbi.events AND imbi.events_v2 ON CLUSTER '<name>';` then recreate the
MV against `imbi.events` again.

**Cleanup** (after a soak period): `DROP TABLE imbi.events_v2 ON
CLUSTER '<name>';`

**Keeper-path caveat.** `EXCHANGE TABLES` swaps names but not the
underlying Keeper paths. If `default_replica_path` embeds `{table}`,
the live `imbi.events` ends up with a path derived from `events_v2` —
functional but cosmetically off. With `{uuid}` paths (Atomic default)
this is a non-issue. Confirm the macro scheme before running in prod.

**Broader gap (own follow-up):** there is no mechanism to apply
*non-additive* ClickHouse changes — every engine/sort-key/partition
change is a manual runbook like this until a non-interactive
`imbi-api migrate-clickhouse` command or a Helm pre-upgrade `Job`
exists.
