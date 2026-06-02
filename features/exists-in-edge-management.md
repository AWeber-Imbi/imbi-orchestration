# Plugin-managed EXISTS_IN edges + a Doctor plugin type

A project's relationship to a third-party service (GitHub, SonarQube,
Sentry, …) is modelled today as a `(:Project)-[:EXISTS_IN]->
(:ThirdPartyService)` edge carrying an `identifier` and a
`canonical_link`. We want these edges to be **owned and maintained by
plugins** rather than hand-entered, and we want a way to **detect and
flag the edges that are wrong** (a lot of them are — see Migration
state below). Each service has its own identifier + URL for a project,
e.g.:

| Third-Party Service | Identifier       | Canonical (API) URL                                  |
|---------------------|------------------|------------------------------------------------------|
| GitHub              | 134741           | `https://api.aweber.ghe.com/repositories/134741`     |
| SonarQube           | conv:account     | `https://sonarqube.aweber.io/api/.../component?...`  |
| Sentry              | 4506270533746688 | `https://aweber-communications.sentry.io/api/0/...`  |

The GitHub plugin currently maintains none of this on the edge — it
writes a `github-repository` entry into `Project.links` via the
`LinkWriteback` side-channel (`imbi-common/.../plugins/base.py:252`,
persisted by `imbi-api/.../endpoints/_helpers.py:218-282`). We want to
move the canonical relationship onto the `EXISTS_IN` edge and treat the
human-facing dashboard URL as a *separate*, optional project link.

## Goals

1. A **LifecyclePlugin** creates and maintains the `EXISTS_IN` edge for
   its service — the `identifier` and the **canonical API URL** (the
   one that returns JSON), not the dashboard URL.
2. The same plugin optionally maintains a **dashboard URL** as a
   project link (via the existing `LinkWriteback` mechanism).
3. A new **Doctor plugin type** validates a project's `EXISTS_IN`
   edges: the canonical URL resolves, the dashboard URL resolves, and
   the identifier on the edge matches the identifier reported by the
   canonical URL's payload.
4. The **API** surfaces edge data on the project response:
   `EXISTS_IN.identifier` merged into the existing `identifiers` map,
   and canonical URLs in a new map — both keyed by ThirdPartyService
   slug.

## Canonical URL = API URL, dashboard URL = project link

The canonical URL on the edge must be the **API** URL (returns JSON),
because the Doctor needs to GET it and read the identifier back out.
The **dashboard** URL (human-facing, e.g.
`https://aweber.ghe.com/apis/account`) belongs in
`Project.links`, written by the plugin, keyed by ThirdPartyService
slug. The two are different and today are frequently conflated.

## Current state (for the planner)

- **Edge + CRUD.** `EXISTS_IN` is real today with a dedicated CRUD path
  (`imbi-api/.../endpoints/webhooks.py:708-884`) and models
  `ExistsInCreate` / `ExistsInResponse`
  (`imbi-api/.../domain/models.py:1268-1286`). The edge property is
  named **`canonical_link`** — we want it to mean the API URL and
  probably rename it `canonical_url` (decision below). Persistence is a
  `(project, service_slug)`-keyed `MERGE`; reuse this path, not the
  generic single-edge `plugin_edges.py` machinery (it's single-edge and
  one-plugin-per-rel-type, neither of which fits multi-service
  `EXISTS_IN`).
- **Plugin↔service binding.** Lifecycle plugins are attached to a
  `ThirdPartyService` by an explicit operator action — the
  `(:ThirdPartyService)-[:HAS_PLUGIN]->(:Plugin)` edge
  (`imbi-common/.../models.py:336-353`). This is the binding that tells
  a plugin which TPS its `EXISTS_IN` edge should point at.
- **Per-TPS config exists.** The plugin instance's `Plugin.options`
  ("Default Options" in the TPS → Plugins admin tab, screenshot in the
  feature discussion) is written via
  `imbi-api/.../endpoints/service_plugins.py` and flows into
  `PluginContext.assignment_options` via the three-tier merge in
  `imbi-api/.../plugins/resolution.py`. The generic Doctor reads its
  config (identifier JSON-pointer, dashboard link key, content-type)
  from here.
- **Analysis fan-out.** Analysis (Doctor) plugins already attach to a
  TPS via `HAS_PLUGIN` and run for every project that `EXISTS_IN` that
  TPS (`imbi-api/.../plugins/resolution.py:432-587`,
  `AnalysisPlugin.analyze` at `imbi-common/.../plugins/base.py:1165`).
  The Doctor builds on this path.
- **Context has no TPS concept.** `PluginContext`
  (`imbi-common/.../plugins/base.py:286-311`) carries `project_links`
  but nothing about `EXISTS_IN` edges, identifiers, or the plugin's own
  service slug. This needs to change for both the write and read sides.
- **Project response.** `ProjectResponse`
  (`imbi-api/.../endpoints/projects.py:214-258`) returns `links` and
  `identifiers` but never traverses `EXISTS_IN`; `_RETURN_FRAGMENT`
  (`projects.py:845-874`) needs an `OPTIONAL MATCH ...EXISTS_IN...`
  + `collect` (the traversal pattern already exists at
  `resolution.py:466`).

## Proposed shape (to be firmed up in planning)

### LifecyclePlugin → EXISTS_IN edge

The lifecycle hooks already fire on create/update/relocate/delete. They
should report the edge state as **pure data** (plugins never touch the
DB; the host persists). Introduce a sibling to `LinkWriteback` rather
than overloading it, since the edge write and the links-map write have
different sinks:

```python
class ServiceWriteback(pydantic.BaseModel):
    service_slug: str                     # EXISTS_IN target (which TPS)
    identifier: str                       # EXISTS_IN.identifier
    canonical_url: str                    # EXISTS_IN canonical (API) URL
    dashboard_links: dict[str, str] = {}  # merged into Project.links
    remove: bool = False                  # tear the edge (+ links) down
```

- `ctx` gains a field to carry one or more of these. The host persists
  via the `(project, service_slug)` MERGE for the edge and merges
  `dashboard_links` into `Project.links`.
- `remove: bool` is new capability — today `LinkWriteback` can only set,
  never delete. Project deletion fires teardown hooks driven off the
  project's `EXISTS_IN` edges.

### PluginContext enrichment (read side)

Surface the project's `EXISTS_IN` connections on the context so both
the lifecycle plugin (knows which edge it owns) and the Doctor (reads
the edge to validate it) work as pure `(ctx, credentials)` functions —
no host-callable, no `models.Project` leak (keeps the contract
host-portable across api/gateway/mcp). Shape TBD, but minimally the
plugin's own service slug + the `{identifier, canonical_url,
dashboard_link}` for the service(s) the plugin is bound to.

### Doctor plugin type

A new `plugin_type='doctor'` (or fold into the existing `analysis`
type — decision below). Checks, per `EXISTS_IN` edge:

1. `canonical_url` does not 404.
2. associated dashboard URL does not 404.
3. `EXISTS_IN.identifier` matches the identifier extracted from the
   canonical URL's JSON payload.

Aim for a **generic, configurable** Doctor that can run against any TPS,
configured via `Plugin.options`: a JSON-pointer (imbi-common ships an
RFC-6901 `json_pointer` type) to extract the identifier from the
response, the `Project.links` key that holds the dashboard URL, and
expected content-type. Service-specific Doctors remain possible.

**Credentials / failure model.** Canonical API URLs require auth. The
person running analysis usually has appropriate creds, and several
TPS's also have service/server-to-server tokens. When no usable
credential is available, checks `skip`/`warn` rather than `fail`.

### API surfacing

- Merge `EXISTS_IN.identifier` into the existing `identifiers` map,
  keyed by **ThirdPartyService slug**. Decide collision precedence with
  any stored `Project.identifiers` node values (lean: edge wins,
  merged read-only at serialization).
- Add a new map (e.g. `service_urls`) of canonical URLs, keyed by TPS
  slug. `ProjectResponse` is `extra='allow'`, so additive.

## Decisions already made (confirmed in discussion)

- Canonical URL on the edge = the **API URL**; dashboard URL lives in
  `Project.links`.
- Map keys (identifiers, canonical URLs, dashboard links) are the
  **ThirdPartyService slug**, not a plugin slug. The slug must be
  visible in `PluginContext`.
- Plugin↔TPS binding is the existing operator-managed `HAS_PLUGIN`
  edge; the TPS node is operator-created (no plugin auto-creates it).
- Writeback stays **pure data** (`ServiceWriteback`), not callable
  methods. The Doctor reads from an enriched `PluginContext`, not a
  host-callable.
- Missing/insufficient credentials → Doctor `skip`/`warn`, never
  `fail`.
- **Skip `edge_labels`** — that mechanism (schema decl + generic
  single-edge CRUD) is unrelated to this feature.
- The shared-contract blast radius in `imbi-common` (PluginContext +
  writeback models, consumed by gateway/mcp, `--strict` docs) is
  expected and acceptable.

## Migration state (why the Doctor exists)

`EXISTS_IN` edges already exist from the Imbi v1 migration, but many are
**wrong**: most point at dashboard URLs, and in numerous cases the
`github-repository` link and `EXISTS_IN.canonical_link` both look like
dashboard URLs yet differ. Expect manual correction of outliers — the
Doctor is partly motivated by *finding* these. Also need a transition
for the existing `github-repository` link: the GitHub plugin's read
path (`derive_owner_repo_from_links`,
`imbi-plugin-github/.../_repos.py:68-90`) resolves `(owner, repo)` from
that link today, so it must start reading the service connection from
context in lockstep with the write cutover, and a backfill should seed
edges from current values where they're trustworthy.

## Open questions for planning

- Rename `EXISTS_IN.canonical_link` → `canonical_url`, or keep the
  existing property name?
- New `plugin_type='doctor'` vs. extending the existing `analysis`
  type (the Doctor panel + analysis fan-out already exist).
- Exact `PluginContext` shape for service connections (single bound
  service vs. full list keyed by slug).
- Precedence when an `EXISTS_IN`-derived identifier collides with a
  stored `Project.identifiers` key.
- Identifier typing: edge stores `str`; `Project.identifiers` is
  `int | str | AnyUrl`. GitHub's is numeric, SonarQube's is a string —
  coerce on the way out?

## Possible future tie-in: Link Definitions

`LinkDefinition` (`imbi-common/.../models.py:464-474`) is an
org-scoped template (`slug`, `name`, `icon`, `url_template`) whose
**`slug` is exactly the key used in `Project.links`** — the UI
(`imbi-ui/.../components/EditLinksCard.tsx`) only renders link keys that
match a definition. Since dashboard URLs will be keyed by TPS slug,
there's a natural future alignment: a ThirdPartyService could own/derive
a LinkDefinition (icon + `url_template`) so dashboard links render
consistently and could even be *derived* from the edge's identifier via
the template instead of stored. Out of scope for the first pass; worth
keeping the keying compatible so we can adopt it later.

## Affected submodules

- **imbi-common** — `PluginContext` + `ServiceWriteback`/`LinkWriteback`
  contract, possible `doctor` plugin type, docs.
- **imbi-api** — edge persistence path, context construction (lifecycle
  + analysis), `ProjectResponse` surfacing, Doctor fan-out, backfill.
- **imbi-plugin-github** — emit `ServiceWriteback`, read service
  connection from context instead of the `github-repository` link.
- **imbi-plugin-sonarqube / others** — adopt the same edge management;
  candidates for the generic Doctor.
- **imbi-ui** — render canonical/dashboard data on the project view;
  regenerate `api-generated.ts` after the API schema changes.
