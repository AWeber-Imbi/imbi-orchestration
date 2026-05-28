# Plan — Inject webhook payloads into ClickHouse from imbi-gateway

## Context

The Imbi UI's project activity feed
(`imbi-ui/src/components/ProjectActivityLog.tsx`) merges two ClickHouse
sources by timestamp: `operations_log` (human-curated entries created
through imbi-api) and `events` (third-party webhook events). The
`events` table schema and reader API already exist
(`imbi-common/src/imbi_common/clickhouse/schemata.toml:4–18`,
`imbi-api/src/imbi_api/endpoints/events.py:201–264`), but **no service
currently writes to it**. Every webhook the gateway processes is
invisible to project owners looking at the activity feed.

This plan makes `imbi_gateway.notifications.process_notification`
populate the `events` table once it has decided to act on a webhook —
**after** both early-return guards (no rule matched, no project
matched). One row is written per affected project, containing the raw
payload, the resolved user, and the inbound HTTP headers.

The activity feed renders the row body from `events.type` (line 274
of `ProjectActivityLog.tsx`: `entry.type.replace(/-/g, ' ')`), so a
blank `type` would show an empty body. To populate `type` correctly
across heterogeneous webhook sources (GitHub, SonarQube, ...), this
plan adds a per-webhook **`event_type_selector`** stored on the
`IMPLEMENTED_BY` edge alongside the existing `identifier_selector` and
`user_subject_selector`. The gateway resolves it at request time with
the layered rule decided below.

## Decisions

User-confirmed:

- **Scope is end-to-end** (imbi-common → imbi-api → imbi-ui →
  imbi-gateway). The new selector field must be settable through the
  existing webhook admin form.
- **`event_type_selector` resolution rules** (verbatim from the
  user):
  1. If the selector value starts with `/`, treat it as a JSON
     pointer evaluated against the payload body. The resolved value
     is stringified and used.
  2. Otherwise, treat it as an HTTP header name and look it up
     case-insensitively on the inbound request.
  3. If the header is absent (or the selector is something like
     `SonarQube Notification` that can't be a header), fall back to
     **using the selector itself as the literal value** so sources
     without a useful event-type field still get a stable label.
  4. If the selector is `None`/empty, store `''` (the schema's
     default). The activity feed will show an empty body — acceptable
     only for legacy webhook configs that haven't been updated yet.

Pre-decided defaults (sensible; not asked):

- **Target table:** ClickHouse `events`
  (`imbi-common/src/imbi_common/clickhouse/schemata.toml:4–18`).
- **One row per matching project.** Iterate the `records` list from
  the project-resolution Cypher query and emit a single batched
  insert.
- **Headers go in `metadata.headers`.** All inbound headers are
  captured unfiltered (FastAPI lower-cases keys; values are strings).
  `metadata.webhook_id` is also stored so a row traces back to its
  graph configuration.
- **Payload goes in `payload` as-is** (already a dict from
  `_extract_json_body`). Non-dict bodies fall back to `{}` to keep
  the typed insert honest.
- **`attributed_to`** ← `user_id` from `_resolve_user_id(...)` (which
  already returns the user's email). `None` becomes `''` via the
  schema's `LowCardinality(String) DEFAULT ''`.
- **`third_party_service`** ← `service['slug']` (the `Node` base in
  `imbi-common/src/imbi_common/models.py:99` declares `slug: str`,
  and the existing Cypher in `notifications.py` returns `tps{.*}`).
- **`id`** ← `nanoid.generate()`, mirroring `operations_log`
  (`imbi-api/src/imbi_api/endpoints/operations_log.py:216`).
- **`recorded_at`** ← `datetime.now(UTC)` (the `Event` model already
  defaults this).
- **Failure handling:** wrap the ClickHouse insert in `try/except`,
  log the exception, **do not** block handler execution or change
  the HTTP status — analytics are best-effort.
- **Schema ownership:** the gateway **does not** call
  `clickhouse.setup_schema()`. imbi-api remains the sole owner; the
  gateway only `initialize()`s and `aclose()`s.
- **Settings:** the gateway uses `imbi_common.settings.Clickhouse`
  via the existing `CLICKHOUSE_*` env vars; no new settings module.
- **Code placement:** new gateway module
  `src/imbi_gateway/lifespans.py` (plural, mirroring
  `imbi-api/src/imbi_api/lifespans.py`).
- **No automatic kebab-casing of resolved type values.** Users
  configure a selector that already produces a UI-friendly value.
  Storing literal `'SonarQube Notification'` renders as
  "SonarQube Notification"; storing `'deployment-status'` renders as
  "deployment status".

## Critical files

### imbi-common

- `src/imbi_common/clickhouse/schemata.toml:4–18` — read-only
  reference for column types and defaults.
- `src/imbi_common/models.py:584–594` — `Event` model. Add:
  - `id: str = pydantic.Field(default_factory=nanoid.generate)`
  - `type: str = ''`
  (Existing fields `project_id`, `recorded_at`, `third_party_service`,
  `attributed_to`, `metadata`, `payload` are already correct.)
- `src/imbi_common/clickhouse/__init__.py` — `insert(table, data:
  list[BaseModel])` is the entry point the gateway will call.

### imbi-api (selector plumbing)

- `src/imbi_api/domain/models.py` — webhook request/response Pydantic
  models. Add `event_type_selector: str | None = None` to:
  - `WebhookCreate` / `WebhookCreateRequest` (whichever the
    `create_webhook` handler accepts)
  - `WebhookResponse` (so the GET shape exposes the value)
  - `WebhookResponse.from_graph_record` factory (read the new key
    from the graph record)
- `src/imbi_api/endpoints/webhooks.py`:
  - **CREATE** (lines 240–284): add `event_type_selector` to the
    `IMPLEMENTED_BY` props in the `CREATE (w)-[:IMPLEMENTED_BY {{...}}]`
    template, and to `write_params`.
  - **`_FETCH_WEBHOOK_QUERY`** (lines 134–166): add
    `impl.event_type_selector AS event_type_selector` to the RETURN
    clause and to the `null AS event_type_selector` fallback when
    no IMPLEMENTED_BY edge exists.
  - **`list_webhooks` query** (lines 351–388): same — add
    `impl.event_type_selector AS event_type_selector` to RETURN and
    to the `db.execute` columns list.
  - **PATCH / UPDATE paths** (lines 320–340, 380–390, 410–411,
    460–461, 490–494): add `'event_type_selector'` to every list of
    selectable fields, the diff allowlist, and the
    `existing[0].get('event_type_selector')` defaults.
  - The `IMPLEMENTED_BY`-update SET clause (search for `SET
    impl.identifier_selector` or `set_clause` usage on impl) — add
    `event_type_selector` to the props it can update.
- `src/imbi_api/endpoints/third_party_services.py:462–487` — the
  webhook-listing query embedded in this endpoint also reads the
  `IMPLEMENTED_BY` edge. Update RETURN + columns list to surface the
  new field if/when it's used. (Verify whether this endpoint exposes
  it in its response model; if not, just keep the field selectable
  for symmetry.)

### imbi-ui

- `src/components/admin/webhooks/WebhookForm.tsx` — add a text input
  for `event_type_selector` next to the existing `identifier_selector`
  and `user_subject_selector` inputs. Help text should explain the
  three resolution modes (JSON pointer / header name / literal
  fallback).
- `src/components/admin/webhooks/WebhookDetail.tsx` — display the
  selector value in the read-only detail view alongside the existing
  selectors.
- `src/types/api-generated.ts` — regenerated by the user via
  `npm run codegen:fetch` once the imbi-api change is merged and
  served locally. (Don't hand-edit.)
- The `endpoints.ts` typed wrappers and the `WebhookResponse`
  TypeScript type may also need an explicit field if not regenerated
  automatically.

### imbi-gateway (the actual feature)

- `src/imbi_gateway/notifications.py:59–183` — `process_notification`.
  Insertion point is **after line 172** (post user resolution) and
  **before the `for record in records:` loop at line 173**. Both
  early-return guards have already passed at this point.
- `src/imbi_gateway/app.py:11–19` — compose the new
  `clickhouse_hook` into the `lifespan.Lifespan(...)` call.
- `src/imbi_gateway/lifespans.py` — **new module** with the
  ClickHouse hook (pattern copied verbatim from
  `imbi-api/src/imbi_api/lifespans.py:45–52`).

### Reference (read-only, for shape)

- `imbi-api/src/imbi_api/endpoints/events.py:32–40` — `EventRecord`
  response model (the read shape the UI expects).
- `imbi-ui/src/components/ProjectActivityLog.tsx:236–287` —
  `EventEntry` component renders `entry.type.replace(/-/g, ' ')` as
  the body and uses `entry.attributed_to` as the email for the
  Gravatar and admin-user-display-name lookup.

## Implementation outline

Ordered by dependency. Each component lints / type-checks /
tests cleanly before moving on.

### 1. imbi-common — extend the `Event` model

`src/imbi_common/models.py:584–594`:

```python
class Event(pydantic.BaseModel):
    """A third-party service event recorded in ClickHouse."""

    id: str = pydantic.Field(default_factory=nanoid.generate)
    project_id: str
    recorded_at: datetime.datetime = pydantic.Field(
        default_factory=lambda: datetime.datetime.now(datetime.UTC),
    )
    type: str = ''
    third_party_service: str
    attributed_to: str | None = None
    metadata: dict[str, typing.Any] = {}
    payload: dict[str, typing.Any] = {}
```

Add a small unit test asserting `Event(project_id='p',
third_party_service='gh').id` is a non-empty string and `.type ==
''`.

### 2. imbi-api — add `event_type_selector` to webhook CRUD

- Update the Pydantic webhook models (request, response, and the
  `from_graph_record` factory) to carry `event_type_selector: str |
  None = None`.
- Update every Cypher query in `endpoints/webhooks.py` that touches
  the `IMPLEMENTED_BY` edge — CREATE, _FETCH_WEBHOOK_QUERY, list,
  PATCH/SET helpers — so the new property is read and written.
- Update the diff allowlist / read-only-paths sets so PATCH can
  modify the selector.
- Tests:
  - Create a webhook with an `event_type_selector` and read it back
    — value round-trips.
  - PATCH a webhook to set/clear the selector — value updates.
  - List endpoint surfaces the selector for both edges that have it
    and edges that don't (None).
  - Existing webhook tests that don't pass the new field continue
    to pass (default `None`).

### 3. imbi-ui — surface the selector in the admin form

- Add a text input + help text in
  `src/components/admin/webhooks/WebhookForm.tsx` mirroring the
  existing selector inputs.
- Add the field to the read-only display in `WebhookDetail.tsx`.
- Run `npm run codegen:fetch` (user-driven; this requires a running
  local API with the imbi-api change applied) to regenerate
  `src/types/api-generated.ts`.
- `npm run lint && npm run format:check && npm run build` to verify.

### 4. imbi-gateway — wire ClickHouse into the lifespan

Create `src/imbi_gateway/lifespans.py`:

```python
"""Lifespan hooks for imbi-gateway."""

import contextlib
import logging
from collections import abc

from imbi_common import clickhouse

LOGGER = logging.getLogger(__name__)


@contextlib.asynccontextmanager
async def clickhouse_hook() -> abc.AsyncGenerator[None]:
    """Initialize and manage the ClickHouse connection."""
    result = await clickhouse.initialize()
    if result is False:
        raise RuntimeError('ClickHouse initialization failed')
    async with contextlib.aclosing(clickhouse):
        yield
```

In `src/imbi_gateway/app.py`:

```python
from imbi_gateway import app_status, lifespans, notifications
...
lifespan=lifespan.Lifespan(
    graph.graph_lifespan, lifespans.clickhouse_hook,
),
```

### 5. imbi-gateway — emit events from `process_notification`

In `src/imbi_gateway/notifications.py`:

- Add imports: `from imbi_common import clickhouse, models` (alongside
  the existing `from imbi_common import graph`). Keep the existing
  `import jsonpointer` — the resolver reuses it.
- Add a private helper near the existing `_resolve_user_id`:

  ```python
  def _resolve_event_type(
      selector: str | None,
      body: object,
      headers: typing.Mapping[str, str],
  ) -> str:
      """Resolve the event-type label per the configured selector.

      - None/empty selector -> ''.
      - Selector starting with '/' -> JSON pointer against `body`;
        on miss, log and return ''.
      - Otherwise -> case-insensitive header lookup; if the header
        is absent, return the selector itself as a literal label.
      """
      if not selector:
          return ''
      if selector.startswith('/'):
          try:
              resolved = jsonpointer.JsonPointer(selector).resolve(
                  body,
              )
          except jsonpointer.JsonPointerException:
              LOGGER.warning(
                  'event_type_selector %r failed to resolve',
                  selector,
              )
              return ''
          return '' if resolved is None else str(resolved)
      header_value = headers.get(selector)
      if header_value:
          return header_value
      return selector
  ```

- In `process_notification`, immediately after the `user_id = await
  _resolve_user_id(...)` call (currently line 167–172) and before
  `for record in records:` (line 173):

  ```python
  event_type = _resolve_event_type(
      sel.get('event_type_selector'),
      body,
      request.headers,
  )
  try:
      await clickhouse.insert(
          'events',
          [
              models.Event(
                  project_id=graph.parse_agtype(
                      record['project_id'],
                  ),
                  type=event_type,
                  third_party_service=service['slug'],
                  attributed_to=user_id,
                  metadata={
                      'webhook_id': webhook_id,
                      'headers': dict(request.headers),
                  },
                  payload=body if isinstance(body, dict) else {},
              )
              for record in records
          ],
      )
  except Exception:  # noqa: BLE001
      LOGGER.exception(
          'Failed to record webhook events in ClickHouse',
      )
  ```

The existing handler-execution loop runs unchanged afterwards.

### 6. Lint / format / type-check

In each touched submodule: `just format && just lint && just test`.

## Tests

### imbi-common

- `tests/test_models.py`: `Event` has a non-empty default `id` and a
  default `type == ''`; explicit values round-trip.

### imbi-api

- `tests/endpoints/test_webhooks.py` (or wherever webhook tests
  live):
  - CREATE webhook with `event_type_selector='x-github-event'` →
    GET returns the same value.
  - CREATE without the field → GET returns `None`.
  - PATCH adding/removing the selector → reflected on subsequent
    GET.
  - List endpoint includes the selector for each row.
  - Pre-existing webhook tests (no selector) still pass.

### imbi-ui

- Component test (or Storybook story) for `WebhookForm` covering
  the new input rendering and submission. (Match the existing test
  style for the form.)

### imbi-gateway

In `tests/test_notifications.py` (extend existing tests; if absent,
create the file using `unittest.IsolatedAsyncioTestCase` per the
imbi-gateway test convention). Patch
`imbi_common.clickhouse.insert` per test.

- **Happy path, multiple matching projects.** One filter passes,
  two project rows. Assert `clickhouse.insert` called once with
  `'events'` and a list of two `Event` instances; verify
  `project_id`, `type`, `third_party_service`, `attributed_to`,
  `metadata['webhook_id']`, `metadata['headers']`, and `payload`
  reflect the fixture.
- **Selector resolution — JSON pointer.**
  `event_type_selector='/action'`, body has `{'action': 'opened'}`
  → `events.type == 'opened'`.
- **Selector resolution — JSON pointer miss.** Selector points at
  a non-existent path → `events.type == ''`, warning logged, insert
  still happens.
- **Selector resolution — header present.**
  `event_type_selector='X-GitHub-Event'`, request header
  `X-GitHub-Event: deployment_status` → `events.type ==
  'deployment_status'`.
- **Selector resolution — header absent, literal fallback.**
  `event_type_selector='SonarQube Notification'`, no matching
  header → `events.type == 'SonarQube Notification'`.
- **Selector unset.** `event_type_selector` is None →
  `events.type == ''`.
- **No insert when no rule matches.** All filter results falsy →
  early return at the existing line 143; assert
  `clickhouse.insert` is never called.
- **No insert when no project matches.** Empty result from the
  matching-projects Cypher → early return at line 160; assert
  `clickhouse.insert` is never called.
- **Insert failure does not break handlers.** Patched
  `clickhouse.insert` raises; assert handlers still run, exception
  is logged, response code is unchanged (202 on success path).
- **Anonymous webhook.** `_resolve_user_id` returns `None`; the
  resulting `Event` has `attributed_to=None`.
- **Headers passthrough.** Custom request headers appear in
  `metadata['headers']` (lower-cased per FastAPI convention).

## Verification

Run inside each affected submodule (in this order):

```bash
# imbi-common
cd imbi-common && just format && just lint && just test

# imbi-api
cd ../imbi-api && just format && just lint && just test

# imbi-ui
cd ../imbi-ui
npm run lint
npm run format:check
npm run codegen:fetch   # against a running imbi-api
npm run build

# imbi-gateway
cd ../imbi-gateway && just format && just lint && just test
```

End-to-end spot check (after `just serve` brings up the API and the
gateway alongside Docker compose):

1. **Configure a webhook with the new selector.** In the imbi-ui
   admin → Third-party services → (your service) → Webhooks → edit
   one of the webhooks. Set
   `event_type_selector` = `x-github-event` (or `/action` for a
   JSON-pointer config, or `SonarQube Notification` for the literal
   case). Save.
2. **POST a sample webhook** (deployment_status payload from
   `payloads/`):
   ```bash
   curl -fsS -X POST \
     -H 'Content-Type: application/json' \
     -H 'X-GitHub-Event: deployment_status' \
     -H 'X-Hub-Signature-256: sha256=...' \
     --data-binary @payloads/deployment_status.json \
     http://localhost:8001/notifications/<webhook_id>
   ```
   Expect `202 Accepted` (handler matched and project matched).
3. **Verify the row landed in ClickHouse.** Query the read endpoint:
   ```bash
   curl -fsS \
     -H 'Authorization: Bearer <token>' \
     "http://localhost:8000/organizations/<slug>/projects/<id>/events" | jq
   ```
   Expect one entry per matched project with `type == 'deployment_status'`,
   `payload` matching the request body, `metadata.headers`
   containing the lower-cased header keys, and `attributed_to` set
   if the GitHub creator id resolved to a user.
4. **Open the project activity feed** in the UI. The new event row
   should render with body "deployment status" (UI replaces hyphens
   with spaces) under the project's day header, sorted alongside any
   `operations_log` entries by timestamp.
5. **Negative checks (manual):**
   - Body that no `WebhookRule.filter_expression` evaluates truthy
     on → expect `204 No Content` and **no** new row in `events`.
   - Body whose project identifier doesn't match any `Project`
     `EXISTS_IN` edge → expect `204` and no new row.
   - Webhook config without an `event_type_selector` → row inserted
     with empty `type` (UI body is blank but row is otherwise
     visible — confirms the legacy fallback works).

## Out of scope / deferred

- Filtering sensitive headers. None of the documented webhook
  sources include `Authorization` / `Cookie`-style headers in their
  payloads; revisit if a future webhook source does.
- Refactoring `process_notification` itself (it has a
  `# TODO(daves)` comment about pushing the per-project graph query
  into imbi-api — that's its own plan).
- Backfilling `event_type_selector` for existing webhook rows. The
  default is `NULL` and the gateway treats that as "store empty
  type". Operators set the selector on the webhooks they care about
  via the admin UI after this lands.
- Improving the activity-feed UI to also strip underscores (so a
  `type='deployment_status'` would render as "deployment status"
  even without kebab-casing). Not needed for SonarQube
  ("SonarQube Notification") or for selectors that already produce
  kebab-case values.
