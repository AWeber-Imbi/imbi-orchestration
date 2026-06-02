# Plan: Plugin-managed EXISTS_IN edges + a Doctor analysis plugin

## Context

A project's relationship to a third-party service (GitHub, SonarQube,
Sentry) is modelled as a `(:Project)-[:EXISTS_IN]->(:ThirdPartyService)`
edge carrying an `identifier` and a `canonical_link`. Today these edges
are hand-entered (and many migrated from v1 are wrong — pointing at
dashboard URLs, disagreeing with the `github-repository` link). The
GitHub plugin maintains none of it; it writes a `github-repository`
entry into `Project.links` via the `LinkWriteback` side-channel.

This feature makes **lifecycle plugins own the `EXISTS_IN` edge** — the
`identifier` and the **canonical API URL** (the JSON-returning one, which
for GitHub is the rename-stable `/repositories/{id}` form) — while the
human **dashboard URL** moves to `Project.links` keyed by the
ThirdPartyService slug. A new **`imbi-plugin-doctor`** package adds the
first `AnalysisPlugin`: a generic, config-driven checker that flags
edges whose canonical/dashboard URLs 404 or whose identifier disagrees
with the canonical URL's payload — the tool for finding the bad migrated
edges. The **API** surfaces edge data as a read-only `services` list on
the project response (see the Surface correction below — an earlier
identifiers-merge was reverted).

This ships as **one correlated PR set** (edge plumbing + Integrations
panel together) to avoid a half-state where the project response
surfaces edge data with no UI to manage it. See **PR Set 2** below for
the panel + the `/services` dashboard-URL fold.

### Surface correction (after real usage)

The first cut **merged** edge identifiers into the project's
`identifiers` map (edge-wins). Real usage showed the footgun: editing a
service identifier in the Identifiers card PATCHes the *node* map, but
the response re-injects the *edge* value, so the edit silently appears
to do nothing — and the merged key renders as an editable row forever.
So the merge is **dropped**:

- `identifiers` is the node property only (no edge merge); the editable
  map and the service relationship never collide.
- Edge data is surfaced as a read-only **`services`** list on
  `ProjectResponse` (`{service_slug, service_name, identifier,
  canonical_url, dashboard_url}`), replacing the earlier `canonical_urls`
  map. `dashboard_url` is read from `Project.links[slug]`.
- Service identifiers are edited in the **Integrations** panel (which
  writes the edge), not the Identifiers card.

SonarQube/Sentry plugin adoption and the plugin-backed *"Add from
service"* resolve (`on_project_assigned`) remain deferred fast-follows.

## Decisions (confirmed)

- **Doctor = a new `imbi-plugin-doctor` package** containing an
  `AnalysisPlugin` (reuse the existing `analysis` plugin_type; no new
  type, no registry changes). It checks links against identifiers.
- **Rename the edge property `canonical_link` → `canonical_url`** (it is
  now specifically the API URL). One-time graph data migration + code
  rename.
- **Canonical URL = API URL** (returns JSON); **dashboard URL = a
  `Project.links` entry** keyed by ThirdPartyService slug.
- **Map keys = ThirdPartyService slug** for identifiers, canonical URLs,
  and dashboard links. The slug is host-injected into `PluginContext`.
- **The host owns the plugin↔TPS binding.** A plugin writes back the
  edge for *its own* bound service (resolved via
  `(:ThirdPartyService)-[:HAS_PLUGIN]->(:Plugin)`); it cannot target an
  arbitrary service. So `ServiceWriteback` carries no `service_slug`.
- **Writeback stays pure data** (`ServiceWriteback`), not callables. The
  Doctor reads an enriched `PluginContext`, not a host-callable / no
  `models.Project` leak.
- **Missing/insufficient credentials → Doctor `skip`/`warn`, never
  `fail`** (analysis fan-out already returns `{}` on missing creds).
- Edge identifier surfaced **as-is (string)** in the read-only
  `services` list. **Not merged into `identifiers`** (see Surface
  correction) — `identifiers` stays the node-only editable map.
- `edge_labels` is unrelated and out of scope.

## Key findings from exploration (grounding)

- Lifecycle plugins resolve via `USES_PLUGIN {tab:'lifecycle'}`
  (`resolution.py:resolve_all_plugins`), which does **not** return the
  TPS — but the plugin instance is attached to exactly one TPS via
  `HAS_PLUGIN`, so an `OPTIONAL MATCH (tps)-[:HAS_PLUGIN]->(p)` yields
  the bound slug.
- Analysis plugins already discover via the TPS `HAS_PLUGIN` path and
  carry `Plugin.options` (`resolution.py:432-587`); `tps` is bound in
  that query but its slug isn't collected yet.
- The EXISTS_IN upsert/list/delete already exist in
  `imbi-api/.../endpoints/webhooks.py:708-884` (reuse the MERGE at
  782-794).
- Link-writeback machinery to mirror:
  `lifecycle_dispatch.py` capture+persist (`245-294`),
  `_helpers.py:persist_link_writeback`/`update_project_link`
  (`218-282`), `build_lifecycle_context_bundle` (`73-123`).
- `imbi_common.json_pointer.JsonPointer` (RFC-6901) exists for
  identifier extraction. **No `AnalysisPlugin` implementations exist
  yet** — Doctor is the first.
- No UI consumes `/services` / EXISTS_IN today; `ProjectResponse.links`
  & `.identifiers` are flat dicts rendered by `EditLinksCard` /
  `EditIdentifiersCard`. UI work is limited to regenerating types.

## Critical files

### imbi-common (shared contract — affects gateway/mcp; `--strict` docs)
- `src/imbi_common/plugins/base.py`
  - Add `ServiceWriteback(identifier, canonical_url,
    dashboard_links: dict[str,str]={}, remove: bool=False)` beside
    `LinkWriteback` (`:252`).
  - Add `ServiceConnection(service_slug, identifier, canonical_url)`.
  - `PluginContext` (`:286-311`): add `service_writeback:
    ServiceWriteback | None = None` (write side),
    `third_party_service_slug: str | None = None` and
    `service_connections: list[ServiceConnection] = []` (read side,
    host-injected).
- `src/imbi_common/plugins/__init__.py` — export the new public classes
  (needed for `--strict` doc reference).
- `docs/plugins/index.md` + `docs/plugins/lifecycle.md` — document
  `ServiceWriteback`, the context additions, and the host-owned binding.
- Tests: extend `tests/test_plugins/test_base.py` (msgpack round-trip of
  the new context fields, mirroring the `LinkWriteback` case at `:532`).

### imbi-api
- `src/imbi_api/endpoints/_helpers.py`
  - Add `lookup_project_exists_in(db, project_id) ->
    list[ServiceConnection]` (query mirrors `webhooks.py:729-739`, reads
    `canonical_url`).
  - Add `persist_service_writeback(db, ctx)`: if `ctx.service_writeback`
    and `ctx.third_party_service_slug` set — `remove` → DELETE edge +
    drop dashboard keys from `p.links`; else MERGE edge (reuse
    `webhooks.py:782-794`, set `identifier`+`canonical_url`) and merge
    `dashboard_links` into `p.links` (reuse `update_project_link`).
    Best-effort/logged/swallowed like `persist_link_writeback`.
- `src/imbi_api/plugins/resolution.py`
  - `resolve_all_plugins`: add `OPTIONAL MATCH
    (tps:ThirdPartyService)-[:HAS_PLUGIN]->(p)` and collect `tps.slug`;
    add `third_party_service_slug` to `ResolvedPlugin`.
  - `resolve_analysis_plugins`: collect `tps.slug` on the TPS path into
    the resolved entry so the analysis context can inject it.
- `src/imbi_api/plugins/lifecycle_dispatch.py`
  - `LifecycleContextBundle` + `build_lifecycle_context_bundle`: add
    `project_exists_in` via `lookup_project_exists_in` in the
    `asyncio.gather`.
  - `_invoke_one` (`:245-294`): capture `ctx.service_writeback` in the
    `_call` closure and `persist_service_writeback` after the call,
    alongside the existing link writeback.
  - PluginContext build (`:172`): inject `third_party_service_slug`
    (from `ResolvedPlugin`) and `service_connections` (from bundle).
- `src/imbi_api/endpoints/project_analysis.py`
  - `_build_context` (`:92`): inject `third_party_service_slug` +
    `service_connections` (add `lookup_project_exists_in`).
- `src/imbi_api/endpoints/projects.py`
  - `_RETURN_FRAGMENT`: `OPTIONAL MATCH
    (p)-[ei:EXISTS_IN]->(tps:ThirdPartyService)` and `collect({slug,
    name, identifier, canonical_url})` as `service_edges`.
  - `ProjectResponse`: `_build_services` validator turns `service_edges`
    (+ the dashboard URL from `links`) into a read-only
    `services: list[ExistsInResponse]`. `identifiers` is left untouched
    (no merge); the old `canonical_urls` map is removed.
- `src/imbi_api/endpoints/webhooks.py` + `src/imbi_api/domain/models.py`
  - Rename `canonical_link` → `canonical_url` in the 3 EXISTS_IN queries
    and in `ExistsInCreate` / `ExistsInResponse` (`:1268-1286`).
- Tests: `persist_service_writeback` (upsert/remove/links-merge),
  resolution TPS-slug surfacing, lifecycle capture+persist, analysis
  context injection, `ProjectResponse` `services` surfacing,
  EXISTS_IN endpoints under the renamed property.

### imbi-plugin-github
- `src/imbi_plugin_github/lifecycle.py` — replace the 5
  `LinkWriteback(link_key='github-repository', ...)` sites
  (`:221,238,288,368,557`) with `ServiceWriteback(identifier=<repo id>,
  canonical_url=<API /repositories/{id}>, dashboard_links={<tps_slug>:
  html_url})`; on delete/relocate-away use `remove=True`. Construct the
  API base via the existing `_api_base` / `_hosts.py` helpers.
- `src/imbi_plugin_github/_repos.py` — `derive_owner_repo_from_links`:
  read the dashboard link under the TPS slug from `ctx.project_links`,
  with a transition fallback to the legacy `github-repository` key. (The
  repo `id` from `ctx.service_connections` is the durable handle; the
  id-based canonical URL is rename-stable, so renames only rewrite the
  dashboard link, not the edge.)
- Tests: writeback shape per hook; read-path fallback.

### imbi-plugin-doctor (new package)
- New repo/submodule mirroring an existing `imbi-plugin-*` layout
  (`pyproject.toml`, `src/imbi_plugin_doctor/`, registry entry-point).
- `EXISTS_IN` `AnalysisPlugin` (`plugin_type='analysis'`) with options
  model (validated `Plugin.options` → `assignment_options`):
  `identifier_pointer: JsonPointer` (extract id from canonical payload),
  `dashboard_link_key: str` (which `Project.links` key holds the
  dashboard URL; default = bound TPS slug), `content_type: str`.
- `analyze(ctx, credentials)`: resolve the connection for
  `ctx.third_party_service_slug` from `ctx.service_connections`; GET
  `canonical_url` (with creds) → `pass/warn/fail` on reachability + on
  `identifier_pointer` value == edge `identifier`; GET the dashboard URL
  from `ctx.project_links[dashboard_link_key]` → reachability. Missing
  creds / missing connection → `skip`/`warn`, never `fail`.
- Tests: pass/warn/fail/skip matrix with httpx mocked.

### Operational (meta-repo, not a service PR)
- One-time graph migration renaming the property:
  `MATCH ()-[ei:EXISTS_IN]->() WHERE exists(ei.canonical_link) SET
  ei.canonical_url = ei.canonical_link REMOVE ei.canonical_link` — ship
  as `scripts/migrate-exists-in-canonical-url.py` following the existing
  `scripts/migrate-boolean-attributes.py` pattern. Edge data-correction
  (dashboard-vs-API outliers) is expected to be manual, surfaced by the
  Doctor.
- Update `imbi-orchestration/notes/imbi-api-graph-schema.md` (EXISTS_IN
  property rename + that edges are now plugin-maintained).

## Implementation outline — PR Set 1 (edge plumbing) *(implemented)*

1. **imbi-common** — `ServiceWriteback`, `ServiceConnection`,
   `PluginContext` fields, exports, docs, round-trip tests. (No new
   plugin type.)
2. **imbi-api** — rename `canonical_link`→`canonical_url`; add
   `lookup_project_exists_in` + `persist_service_writeback`; surface
   TPS slug in resolution; wire lifecycle capture/persist + context
   injection; analysis context injection; `ProjectResponse` surfacing.
   Run `just lint` + `just test`.
3. **imbi-plugin-github** — emit `ServiceWriteback`; migrate read path
   with legacy fallback. Lint + test.
4. **imbi-plugin-doctor** — scaffold package + the analysis plugin +
   tests. Lint + test.
5. **imbi-ui** — regenerate `api-generated.ts` against the running API
   (user owns `npm run codegen:fetch`); no new components this set.
6. **Operational** — migration script + schema-note update (separate,
   run against the target graph; not a code PR gate).

## PR Set 2 — Integrations panel

A new **Integrations** panel in Project Settings: a table with one row
per `EXISTS_IN` edge (third-party service / identifier / API URL /
dashboard URL) and add/remove controls. It sits beside the existing
**Links** and **Identifiers** cards and is the UI home for linking an
existing project to an existing repo. (Naming: "Integrations" — note a
*separate* future discussion about renaming the admin **Third-Party
Services** section to "Integrations"; out of scope here.)

### Step 1 — fold the dashboard URL into the `/services` endpoints (do first)

The dashboard URL lives in `Project.links` keyed by the service slug,
not on the edge. Make the `EXISTS_IN` row a single coherent resource so
the panel is one call per row.

- `imbi-api/.../domain/models.py` — `ExistsInResponse`: add
  `dashboard_url: str | None`. `ExistsInCreate`: add optional
  `dashboard_url: str | None`.
- `imbi-api/.../endpoints/webhooks.py` (project-services router):
  - **list / create** queries: also return the matching
    `Project.links[slug]` as `dashboard_url` (read the links map, or
    `OPTIONAL`-join it), reusing `lookup_project_links`.
  - **create** (`POST`): when `dashboard_url` is supplied, merge it into
    `Project.links` keyed by the service slug (reuse
    `_helpers.merge_project_links`) in the same handler that MERGEs the
    edge.
  - **delete**: drop the matching `Project.links[slug]` entry alongside
    the edge (reuse `merge_project_links(remove=[slug])`).
  - Consider a `PUT`/`PATCH /{service_slug}` for editing a row in place
    (identifier / API URL / dashboard URL) so the panel can edit, not
    only add+remove.
- Tests: list returns `dashboard_url` from links; create writes the edge
  **and** the link; delete clears both.

### Step 2 — imbi-ui Integrations panel

- New component (e.g. `IntegrationsCard.tsx`) mirroring `EditLinksCard` /
  `EditIdentifiersCard`, wired into `ProjectSettingsTab.tsx`.
- Table columns: service (select from the org's `ThirdPartyService`s),
  identifier, API (canonical) URL, dashboard URL; add + remove rows
  against `…/projects/{id}/services/`.
- Regenerate `api-generated.ts` (`npm run codegen:fetch`; user owns the
  regen step) after the Step 1 schema change.
- Tests: per the UI's component-test conventions.

> The plugin-backed *"Add from service"* button (paste `owner/repo` →
> the lifecycle plugin GETs it and fills identifier + API URL +
> dashboard URL via `on_project_assigned`) is a **fast-follow** within
> this feature, not a prerequisite — manual entry ships first. See
> Deferred.

## Tests

- imbi-common: msgpack/`model_validate` round-trip of new context
  fields; `ServiceWriteback` defaults.
- imbi-api: `persist_service_writeback` upsert + `remove` + dashboard
  merge; resolution surfaces `third_party_service_slug` for lifecycle &
  analysis; lifecycle dispatch captures+persists the writeback; analysis
  `_build_context` injects connections; `ProjectResponse` returns the
  `services` list (with `dashboard_url` from links) and leaves
  `identifiers` untouched; EXISTS_IN endpoints fold `dashboard_url`
  (create writes the link, delete clears it) under `canonical_url`.
- imbi-plugin-github: each hook emits the right `ServiceWriteback`;
  read-path TPS-slug + legacy fallback.
- imbi-plugin-doctor: pass/warn/fail/skip matrix (URL 404, identifier
  mismatch, missing creds, missing connection).

## Verification

- Per submodule: `just lint` && `just test` (imbi-common, imbi-api,
  imbi-plugin-github, imbi-plugin-doctor). Coverage ≥ the repo's
  `fail_under`.
- End-to-end against a local API (`just serve`):
  1. Attach the GitHub lifecycle plugin to a GHEC ThirdPartyService;
     create a project. Confirm an `EXISTS_IN` edge exists with the
     numeric `identifier` and the `/repositories/{id}` `canonical_url`,
     and a dashboard link under the TPS slug in `links`.
     `curl -f .../projects/{id}` shows a `services` entry with the id,
     `/repositories/{id}` `canonical_url`, and the dashboard URL —
     and `identifiers` is unchanged.
  2. Attach `imbi-plugin-doctor` to the same TPS with
     `identifier_pointer=/id`; run analysis
     (`POST .../projects/{id}/analysis`). Confirm `pass` for a good
     edge; hand-edit the edge to a wrong identifier / dashboard URL and
     confirm `fail`/`warn`; remove creds and confirm `skip`/`warn`.
  3. Rename the repo on GitHub; confirm the dashboard link self-heals
     while `canonical_url` (id-based) stays stable.
- Migration: run the rename script against a copy of the graph; confirm
  no `canonical_link` remain and `/services` + project responses read
  correctly.
- **PR Set 2:** `POST …/projects/{id}/services/` with a `dashboard_url`
  writes both the `EXISTS_IN` edge and `Project.links[slug]`; `GET`
  returns `dashboard_url`; `DELETE` clears both. In the UI, the
  Integrations panel lists existing edges and add/remove round-trips
  against those endpoints.

## Deferred (fast-follows / future passes)

- **Plugin-backed "Add from service"** — `LifecyclePlugin.on_project_assigned`
  (+ `'assigned'` in `lifecycle_events`, an inbound `external_reference`
  context field, and a targeted host dispatch that resolves the chosen
  lifecycle plugin by its TPS `HAS_PLUGIN` rather than `USES_PLUGIN`).
  Powers an "Add from service" button in the Integrations panel: paste
  `owner/repo`, the plugin GETs it and fills identifier + API URL +
  dashboard URL. Earns its keep because GitHub's id-based
  `/repositories/{id}` canonical URL is miserable to type by hand.
- SonarQube / Sentry plugins managing their own `EXISTS_IN` edges.
- Surfacing Doctor findings in the Integrations panel; deriving
  dashboard links from `LinkDefinition.url_template`.
- Renaming the admin **Third-Party Services** section to
  **Integrations** (team discussion; separate from this feature).
- Retiring the legacy `github-repository` link key once all consumers
  read the TPS-slug-keyed dashboard link.
