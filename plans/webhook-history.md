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
  - Discussed with a coworker: this is the agreed direction over
    `ReplacingMergeTree` because it avoids changing the table
    engine, the sort key, and any historical-row semantics.

- **Schema changes on `events`.**
  - Engine stays `MergeTree`. Sort key stays
    `(project_id, recorded_at)`.
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
    definition. Engine, partitioning, and sort key stay as-is.
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
- Migration step: an idempotent `ALTER TABLE events ADD COLUMN
  IF NOT EXISTS version UInt8 DEFAULT 0` plus a
  `CREATE VIEW IF NOT EXISTS events_latest AS ...`. Existing
  rows acquire `version = 0` automatically.
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

1. **imbi-common** — schema migration: add `version UInt8
   DEFAULT 0` to the `events` table and create the
   `events_latest` view that selects one row per `id` by highest
   `version`. Update the `Event` model. Verify migration replays
   cleanly on a fresh local ClickHouse and on one with existing
   rows.
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

- Migration test: applying the new schema to a table with
  pre-existing rows preserves all rows, assigns `version = 0`,
  and creates the `events_latest` view.
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
