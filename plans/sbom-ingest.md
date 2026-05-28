# SBoM Ingestion — Implementation Plan

## Context

Builds for Imbi-managed projects already emit CycloneDX SBoMs as a CI
side-effect (npm & python pipelines, command lines in
`features/sbom-ingest.md`). Today those artifacts are thrown away.
We want to ingest them so we can answer:

1. *What components and versions are deployed in a given environment?*
2. *Which projects use a given component?*
3. *Which projects use a given version of a given component?*

The ingest path is **CI build → Gateway webhook → API graph
upsert → UI Dependencies tab**. Imbi Gateway already owns
project-from-payload resolution and forwarding to the Imbi API; the
Imbi API owns CycloneDX parsing and graph mutation; the UI surfaces
the result on the project page's *Dependencies* tab, which today is
a placeholder (`imbi-ui/src/components/ProjectDetail.tsx:1054-1056`).

This is greenfield — no existing CycloneDX/SBoM/purl code lives in any
of imbi-api, imbi-common, or imbi-gateway today (confirmed by repo-
wide search).

## Decisions

Defaults locked unilaterally per "work without stopping for
clarifying questions." Each is reversible; flagged here so the user
can redirect any that don't fit before implementation starts.

1. **Intake: existing webhook + new action handler.** The CI job
   POSTs its SBoM envelope to an Imbi Gateway notification URL
   (`POST /notifications/{webhook_id}`) the same way GitHub does
   today. A new `ingest_sbom` action joins `create_release` and
   `add_deployment_event` in
   `imbi-gateway/src/imbi_gateway/actions.py`. No new gateway
   endpoint, no new dedicated route — the existing webhook-rule +
   plugin-registry path handles dispatch, authentication, and
   project resolution.
2. **Webhook envelope shape (gateway-facing).** Build job posts the
   *raw* GitHub-Actions fields and lets the handler derive the
   release identity via CEL — no precomputed `version` field:
   ```json
   {
     "repository": "<org/repo>",
     "ref_name":   "<github.ref_name — branch/tag>",
     "sha":        "<github.sha — full 40-char SHA>",
     "sbom":       { /* CycloneDX 1.7 JSON */ }
   }
   ```
   `repository` flows through the existing identifier-selector +
   `Project-[:EXISTS_IN]->ThirdPartyService` lookup. The release
   identity is derived by ``IngestSbomConfig.version_expression``
   (CEL) — typical configurations are conditional, e.g.:
   ```text
   ref_name == "main" ? substring(sha, 0, 7) : ref_name
   ```
   so that builds from `main` (testing-environment deploys) land
   under the short SHA — matching the deployment image tag —
   while builds from a release branch / tag land under that
   ref's name. Without conditional logic this is awkward to
   express via a JSON pointer; the CEL form is symmetric with
   `CreateReleaseConfig.committish_expression` /
   `version_expression` (`actions.py:202-215`) and reuses the same
   `_evaluate_cel` helper. `committish_expression` (CEL) is also
   used by the auto-create branch; the bare expression `"sha"`
   passes the full SHA through and the gateway truncates to the
   7 lowercase hex chars the API expects. `sbom_selector` and
   `title_selector` stay as JSON pointers because they point at
   stable payload locations — there's nothing conditional about
   them. Resolved tag → `Release` via
   `ImbiClient.list_releases(tag=...)`, mirroring how
   `add_deployment_event` resolves releases today
   (`actions.py:380-385`). `sbom` is the verbatim CycloneDX payload.
3. **API URL: `PUT /…/releases/{release_id}/sbom`.** Existing release
   routes are keyed by `release_id` (`releases.py:687, 712, 1110`);
   the SBoM PUT follows the same convention. The gateway does the
   tag→id resolution before calling, matching the deployment-event
   pattern. The feature file's "release version in the URL" intent
   is honored — `release_id` *is* the release's version identity in
   Imbi.
4. **CycloneDX 1.7 only.** Reject other `specVersion` values with
   `415 Unsupported Media Type`. The user explicitly wants the
   migration; no transitional 1.5+1.7 acceptance window. Producers
   on 1.5 update their CI commands as part of cutting over.
5. **PUT is idempotent / replace-set semantics, with bounded
   parallel upserts bucketed by ``purl_name``.** Re-running the
   same SBoM (rebuilds, re-deploys) deletes the prior
   `USES_COMPONENT_RELEASE` edges from that release and re-creates
   the new set. Components are bucketed by ``purl_name``; each
   bucket runs sequentially and buckets parallelize via
   `asyncio.gather` under an `asyncio.Semaphore(16)` cap.
   Serial round-trips were fast enough for ~50-component Python
   SBoMs but the gateway times out before the API finishes on
   typical npm trees (500+ components), motivating parallelism.
   Within-bucket serialization is load-bearing: two parallel
   MERGEs against the same ``Component`` vertex (e.g. the
   ``pkg:npm/react-is`` shared by ``react-is@17`` and
   ``react-is@18`` in one SBoM) trigger AGE's "Entity failed to
   be updated: 3" vertex-version conflict — the second writer
   loses. Different purl_names target different vertices, so
   cross-bucket parallelism is safe. The semaphore also bounds
   connection-pool pressure (each task holds one connection at
   a time). Per-component failures (within or across buckets)
   are caught, logged at WARNING with purl + version + release
   id, and the rest of the batch continues — partial graph
   state is more useful than aborting on the first failure.
   Prior art for parallel `db.execute` against the shared pool
   lives in `imbi-api/src/imbi_api/scoring/queue.py:110` and
   `:308`. `Component` / `ComponentRelease` /
   `ComponentIdentifier` nodes are MERGE-ed, never destroyed —
   other releases on other projects keep their references.
6. **Raw SBoM is not persisted.** We store only the normalized graph
   nodes & edges. If we later need the original document, we'll add
   blob storage; not now.
7. **Graph model (the load-bearing decision).** Three new node
   classes plus one edge from the existing `Release`:
   - `Component` — package identity, keyed by canonical purl name
     (purl without version, e.g. `pkg:npm/foo`). Holds display
     name, ecosystem (`npm`, `pypi`, etc.), and description.
   - `ComponentRelease` — a specific version of a component, keyed
     by `(component, version)`. Holds version string and any
     CycloneDX-extracted metadata (license expression, supplier,
     hashes).
   - `ComponentIdentifier` — a single identifier of a kind+value
     pair (`kind: 'purl' | 'cpe' | 'bom-ref' | 'swid'`,
     `value: <string>`). Multiple may attach to one component.
   - Edges:
     - `Component-[:HAS_RELEASE]->ComponentRelease` (mirrors
       existing `Project-[:HAS_RELEASE]->Release` naming so graph
       readers see a consistent vocabulary).
     - `Component-[:IDENTIFIED_BY]->ComponentIdentifier`.
     - `Release-[:USES_COMPONENT_RELEASE]->ComponentRelease`
       (project's release uses this specific component version).
       Carries `ReleaseComponentEdge` properties — `scope`
       (`required` / `optional` / `excluded` / `None`) and
       `groups: list[str]` (dependency-group names). These are
       *per-release* usage facts: the same component version can
       be required by release A and a dev-group dependency in
       release B, so they live on the edge rather than on
       `ComponentRelease`. See ADR 0015 for the cdxgen-side
       semantics that populate them.
   This shape answers all three motivating queries by traversal
   alone — no edge-attribute filters needed. Choice over a single-
   `Component`-node-with-version-on-edge model: edge attribute
   filtering is awkward in AGE Cypher and would force every "what
   versions of X are in use?" query to enumerate edge bags.
8. **Component keying & dedup.** Canonical key for `Component` is
   `purl_name` (purl with version stripped). For ecosystems that
   give us a CPE but no purl, we synthesize a stable surrogate
   (`pkg:cpe/<vendor>/<product>` style) so the keying invariant
   holds. `ComponentRelease` is keyed by
   `(component.purl_name, version)`. Identifiers are MERGE-ed on
   `(kind, value)`.
9. **Initial Dependencies-tab scope: per-release listing.** Tab
   shows a release dropdown (defaulting to the project's most
   recent release with an SBoM) + an `AdminTable` of components
   and versions. Cross-release diffs, vuln overlays, and "which
   projects use X" reverse-search are out of scope for this slice.
10. **Permission model.** SBoM PUT requires `project:write` (matches
    `releases.py` mutation guards). Reads require `project:read`.

## Critical files

### imbi-common
- `src/imbi_common/models.py` — append `Component`,
  `ComponentRelease`, `ComponentIdentifier` GraphModels and the
  `Release.components` Edge annotation. Follow the existing
  `Release` shape (line 512-549) for style: `Annotated[…, Edge(…)]`
  for relationships, `pydantic.Field` for value constraints.
- `docs/api/models.md` (or equivalent) — note the new node classes;
  this is a published library.

### imbi-api
- `pyproject.toml` — declare `cyclonedx-python-lib>=11.7,<12` via
  `uv add`. (The feature file's "v3.x" reference is stale — that
  series predates CycloneDX 1.7 support; modern releases live in
  the 11.x line.)
- `src/imbi_api/endpoints/releases.py` — add two handlers:
  - `PUT /organizations/{org_slug}/projects/{project_id}/releases/{release_id}/sbom`
    (validates CycloneDX 1.7, calls normalize, replaces dep edges).
  - `GET /organizations/{org_slug}/projects/{project_id}/releases/{release_id}/dependencies`
    (returns a flat list of `{component, version, identifiers}` for
    the UI).
  - Reuse existing `_project_exists` (line 218) and `_fetch_release`
    (line 238) helpers — no new lookup primitives needed.
- `src/imbi_api/sbom.py` (new) — pure module: `parse(sbom_json) ->
  list[NormalizedComponent]` using cyclonedx-python-lib's
  `Bom.from_json()`. Validates `specVersion == "1.7"`; extracts
  purl, version, identifiers (purl/cpe/bom-ref/swid), license, and
  supplier per component; computes `purl_name` (purl w/o
  `@version`). Dedupes by `(purl_name, version)`.
- `src/imbi_api/sbom.py` (extended) — graph-application helpers
  alongside `parse()`: `replace_release_components(db, release_id,
  components)` for the PUT path and `list_release_components(db,
  release_id)` for the GET path. Cypher lives as
  ``typing.LiteralString`` constants inside the module (the
  existing `graph_sql.py` is only helpers, not a templates repo).
  Brace-escape conventions: `{{...}}` for property maps, single
  `{param}` for parameters.
- `src/imbi_api/openapi.py` — confirm the new models are picked up
  via the existing blueprint-enhanced generator; bump the
  `openapi.json` snapshot.
- `tests/endpoints/test_releases.py` (and a new
  `tests/test_sbom.py` for the normalizer).
- `tests/fixtures/sbom/` (new) — checked-in sample CycloneDX 1.7
  JSONs: one tiny synthetic, one realistic npm capture, one
  realistic pypi capture.

### imbi-gateway
- `src/imbi_gateway/actions.py`:
  - Add `IngestSbomConfig(BaseModel)` with
    `version_expression: str` (CEL) and
    `sbom_selector: JsonPointer` (the SBoM is at a stable path
    on the payload). Optional `committish_expression: str` and
    `title_selector: JsonPointer` for the auto-create branch.
  - Add `ingest_sbom()` action with the canonical signature `(ctx,
    credentials, external_identifier, action_config, payload)`.
    Flow: extract version + sbom from payload → call
    `list_releases(tag=version)` → if matched, call new
    `ImbiClient.put_sbom(...)`; if not matched, log warning and
    return (consistent with `add_deployment_event` 404 handling
    at `actions.py:400-406`).
  - Extend `ImbiClient` with `async def put_sbom(self, org_slug,
    project_id, release_id, sbom_json) -> httpx.Response` mirroring
    `record_deployment` (line 127-147) for URL + auth + error log.
- `src/imbi_gateway/plugin.py` — register `ingest_sbom` in the
  built-in action descriptor list.
- `pyproject.toml` — no new deps; cyclonedx parsing happens on the
  API side so the gateway only forwards bytes.
- `tests/test_actions.py` — extend with a new `IngestSbomTests`
  class following the `UpdateProjectTests` (line 32-128) pattern.

### imbi-ui
- `src/types/api-generated.ts` — regenerate via `npm run
  codegen:fetch` after the API endpoints land. User owns the
  regen step (project convention).
- `src/api/endpoints.ts` — add a thin wrapper for
  `listReleaseDependencies(orgSlug, projectId, releaseId)`.
- `src/components/dependencies/DependenciesTab.tsx` (new) —
  release-version dropdown (re-uses release-list endpoint already
  wired into the env train at `ProjectDetail.tsx:138-188`) +
  `AdminTable` (`src/components/ui/admin-table.tsx`) with columns
  *Component*, *Version*, *Identifiers*, *License*.
- `src/components/ProjectDetail.tsx:1054-1056` — replace
  `PlaceholderTab name="Dependencies"` with `<DependenciesTab
  project={project} />`.
- `src/components/dependencies/__tests__/DependenciesTab.test.tsx`
  (new) — Vitest + RTL, mock the endpoint via `vi.spyOn`, follow
  the `RecentActivity.test.tsx` shape.

## Implementation outline (dependency-ordered)

The three submodules can be merged independently once each is green.
API lands first because gateway tests and UI codegen depend on it.

1. **imbi-common — add graph models.** `Component`,
   `ComponentRelease`, `ComponentIdentifier`; extend `Release` with
   the outgoing `USES_COMPONENT_RELEASE` edge annotation. Update
   docs page and Module map row if a new module is added (it isn't
   here — pure model additions). `just lint && just test`.
2. **imbi-api — add `cyclonedx-python-lib` via `uv add`.** Then
   write `src/imbi_api/sbom.py` (pure parser + normalizer, fully
   unit-testable without the graph). Bring it up against the
   checked-in fixtures.
3. **imbi-api — Cypher templates** in `graph_sql.py` for the five
   operations listed above. Unit-test the SQL renderer if there's
   a precedent; otherwise rely on the endpoint integration tests.
4. **imbi-api — `PUT …/sbom` endpoint.** Wire request validation,
   `_project_exists` + `_fetch_release` precondition checks,
   `sbom.parse(...)`, then the upsert+link sequence wrapped in a
   single AGE transaction. Return `204 No Content` on success.
5. **imbi-api — `GET …/dependencies` endpoint.** Single Cypher
   traversal returning components, version, and identifiers per
   release. Response shape: `{ release_id, components: [{ name,
   ecosystem, version, license, identifiers: [{ kind, value }] }] }`.
6. **imbi-api — OpenAPI snapshot + tests.** `just test`; bump the
   committed `openapi.json` snapshot if one is checked in.
7. **imbi-gateway — `ingest_sbom` action + `ImbiClient.put_sbom`.**
   Add plugin registration. Unit tests mock `ImbiClient.put_sbom`,
   `ImbiClient.list_releases`, and `override_environment`
   (`tests/helpers.py` pattern). `just lint && just test`.
8. **imbi-ui — regenerate `api-generated.ts`** against a running
   API (user-triggered step). Add `endpoints.ts` wrapper.
9. **imbi-ui — `DependenciesTab` + integration into
   `ProjectDetail.tsx`.** Vitest spec covers empty state, populated
   state, and release dropdown selection. `npm run lint && npm
   test`.

## Tests

### imbi-common
- Unit: instantiation of each new model with happy + invalid
  payloads (e.g. missing version, empty identifier set).

### imbi-api (`tests/test_sbom.py`)
- `parse()` accepts a CycloneDX 1.7 fixture and returns the expected
  normalized list (npm and pypi fixtures).
- `parse()` raises a typed error on `specVersion != "1.7"`.
- `parse()` raises on malformed JSON / missing required CycloneDX
  fields.
- `parse()` dedupes identical `(purl_name, version)` pairs.
- `parse()` extracts purl, CPE, bom-ref, and swid into separate
  identifier records.

### imbi-api (`tests/endpoints/test_releases.py`)
- `PUT …/sbom` happy path with the small synthetic fixture; asserts
  `204` and verifies the graph mock receives the expected Cypher
  parameter shape.
- `PUT …/sbom` returns `404` for unknown project / unknown release.
- `PUT …/sbom` returns `415` on non-1.7 spec version.
- `PUT …/sbom` returns `400` on malformed payload.
- `PUT …/sbom` is idempotent: second PUT clears prior edges (mock
  asserts `CLEAR_RELEASE_COMPONENTS` runs).
- `GET …/dependencies` returns the expected response shape.
- All tests follow the existing `dependency_overrides[graph.
  _inject_graph]` mocking pattern (`test_releases.py:38-83`).

### imbi-gateway (`tests/test_actions.py`)
- `ingest_sbom` extracts version + sbom from payload and calls
  `ImbiClient.list_releases` then `put_sbom` with the expected URL.
- `ingest_sbom` logs and returns when `list_releases` finds no
  matching release (parity with `add_deployment_event:400-406`).
- `ingest_sbom` propagates non-2xx from `put_sbom` without raising
  (consistent with other action handlers).
- `IngestSbomConfig` validates the handler-config JSON.

### imbi-ui (`DependenciesTab.test.tsx`)
- Renders the empty-state when the release has no dependencies.
- Renders the table populated with mocked endpoint data.
- Switching release in the dropdown triggers a new endpoint call
  with the new `release_id`.

## Verification

Per-submodule (run in submodule directory):
- `imbi-common/`: `just lint && just test`
- `imbi-api/`: `just lint && just test`
- `imbi-gateway/`: `just lint && just test`
- `imbi-ui/`: `npm run lint && npm test`

End-to-end smoke test on the meta-repo dev stack:
1. From the meta-repo root, `just serve` (or the per-service
   equivalent) to bring up imbi-api, imbi-gateway, imbi-ui.
2. Generate a CycloneDX 1.7 SBoM for any local repo using the
   feature-file commands.
3. Wrap it in the envelope from Decision 2 and POST to the
   gateway's notification URL for that project:
   ```bash
   curl -fsS -X POST "$IMBI_GATEWAY_URL/notifications/<webhook_id>" \
        -H 'Content-Type: application/json' \
        --data @envelope.json
   ```
4. Verify the gateway log shows project + release resolution
   followed by `PUT …/sbom` → 204.
5. `curl -fsS "$IMBI_API_URL/organizations/<org>/projects/<id>/releases/<release_id>/dependencies" | jq '.components | length'`
   returns a positive count.
6. Open the project page in the UI, click *Dependencies*, confirm
   the dropdown defaults to the release just ingested and the table
   renders the expected components.
7. Re-POST the same envelope; confirm the dependency count is
   unchanged (idempotence).

## Producer guidance (cdxgen)

After the initial slice landed on `cyclonedx-python-lib` for parsing
on the API side, we evaluated the `cyclonedx-py` (`cyclonedx-bom`)
producer and found it does not handle PEP 735
`[dependency-groups]` at all — dev-vs-runtime attribution is lost
for any `uv`/`pdm` project. We switched the **producer-of-record to
[cdxgen](https://github.com/CycloneDX/cdxgen)**. ADR 0015 codifies
this; the summary that matters for producers is below.

### Property surface Imbi keys on

Imbi populates the `Release-[:USES_COMPONENT_RELEASE]->ComponentRelease`
edge's `scope` and `groups` from exactly two cdxgen-emitted signals:

- `component.scope` (CycloneDX 1.7 native field): one of
  `required` / `optional` / `excluded`. Universal across
  ecosystems. cdxgen emits this for npm dev deps, Python PEP 735
  dev-groups, Poetry groups, Go indirect deps, Maven test scope,
  etc. Absent means *unstated* — Imbi surfaces that as `None`,
  not silently defaulted to `required`.
- `component.properties[]` entries whose `name` is in the curated
  Imbi allow-list. Today that list is exactly
  `("cdx:pyproject:group",)` and covers both Python PEP 735 and
  Poetry groups (cdxgen folds both into the same property name).
  As cdxgen grows similar groupings in other ecosystems (npm
  workspaces, Maven profiles), we extend the tuple in
  `imbi-api/src/imbi_api/sbom.py`'s `_GROUP_PROPERTY_NAMES`. We
  deliberately do **not** regex-match `cdx:*:group` — that would
  catch unrelated keys like `cdx:npm:package:development` (a
  boolean, not a group name) and pollute the `groups` list.

The CycloneDX property taxonomy reference is at
<https://github.com/CycloneDX/cyclonedx-property-taxonomy>.

### Why CEL for `version_expression` (and `committish_expression`)

In GitHub Actions, the *raw* fields the workflow has at hand
are `github.ref_name` (branch or tag name) and `github.sha`
(full 40-char SHA). What Imbi keys releases on, however, is
workflow-conditional:

- Builds from `main` deploy to **testing** and are tagged on
  the deployment image by short SHA — so the SBoM has to land
  under that same short-SHA identity, or it won't match the
  release Imbi is tracking.
- Builds from a release branch or tag ship under that ref's
  name (a semver string, typically) — the ref name *is* the
  release identity.

A JSON pointer can't express that branching logic; a CEL
ternary can:

```text
ref_name == "main" ? substring(sha, 0, 7) : ref_name
```

`committish_expression` is also CEL for symmetry with
`CreateReleaseConfig` / `AddDeploymentEventConfig`, and so a
future producer that emits the committish in a non-trivial
field (or wants to fall back to a default) can express that
inline. The vast majority of producers will configure it as
the bare reference `"sha"` — full SHA in, the gateway trims to
7 lowercase hex chars on the way out. `sbom_selector` and
`title_selector` stay as JSON pointers because nothing about
them is workflow-conditional; they point at stable payload
locations.

CEL evaluation failures (a referenced field is absent, an
expression returns null) drop the SBoM with a logged warning
rather than 500ing the webhook — the auto-create branch is
designed to be forgiving of producer-side metadata gaps.

### Generating an SBoM with `cdxgen`

`cdxgen` is a Node.js tool; install with `npm install -g
@cyclonedx/cdxgen` or invoke ad-hoc with `npx @cyclonedx/cdxgen`.
Use `--spec-version 1.7` to align with Imbi's accepted version,
and `--no-include-formulation` for smaller payloads when you
don't need build-process attestations.

#### uv-based Python (project with `uv.lock`)

cdxgen picks up `uv.lock` automatically when both it and a
`pyproject.toml` are present in the working directory. The
default mode walks the lock file (transitive deps + scope +
group attribution) without invoking the build backend:

```bash
cdxgen --spec-version 1.7 \
       --type python \
       --output-format JSON \
       --output "$RUNNER_TEMP/sbom.json" \
       --project-name "$IMBI_PROJECT_SLUG" \
       --project-version "$RELEASE_TAG" \
       .
```

`--type python` short-circuits the language-detection step
(cdxgen is otherwise polyglot and may also scan a `package.json`
in the same repo). The resulting components carry
`cdx:pyproject:group` entries for any dep under
`[dependency-groups]` in `pyproject.toml` *and* any dep under
`[tool.poetry.group.X.dependencies]` in Poetry projects.

#### npm-based JS/TS (project with `package-lock.json`)

```bash
cdxgen --spec-version 1.7 \
       --type javascript \
       --output-format JSON \
       --output "$RUNNER_TEMP/sbom.json" \
       --project-name "$IMBI_PROJECT_SLUG" \
       --project-version "$RELEASE_TAG" \
       .
```

cdxgen reads `package-lock.json` directly (no install required).
`devDependencies` come through with `scope: "optional"` plus a
`cdx:npm:package:development = "true"` property. Imbi reads the
scope but ignores the boolean property — npm has no *named*
groups beyond the dev/peer/optional binary distinction, so the
`groups` list stays empty for npm components by design.

#### POSTing the envelope through the gateway

The `ingest_sbom` action consumes the envelope shape from
Decision 2 — *raw* `ref_name` and `sha` from the workflow, with
the gateway-side CEL deciding what each maps to:

```bash
jq -n \
  --slurpfile sbom "$RUNNER_TEMP/sbom.json" \
  '{repository: env.GITHUB_REPOSITORY,
    ref_name:   env.GITHUB_REF_NAME,
    sha:        env.GITHUB_SHA,
    sbom:       $sbom[0]}' \
| curl --fail -X POST \
    -H 'Content-Type: application/json' \
    --data @- \
    "$IMBI_GATEWAY_URL/notifications/$IMBI_SBOM_WEBHOOK_ID"
```

Webhook-rule `handler_config` (created in the Imbi admin UI):

```json
{
  "version_expression": "ref_name == \"main\" ? substring(sha, 0, 7) : ref_name",
  "sbom_selector": "/sbom",
  "committish_expression": "sha"
}
```

The conditional `version_expression` is the load-bearing piece:
`main` builds ship to testing under the short SHA (matching the
deployment image tag), so the SBoM has to land under that same
identity — otherwise Imbi would search by `tag="main"` and never
find a matching release. Branch / tag builds reuse the ref name
as-is. The full `sha` flows through `committish_expression`; the
gateway truncates to 7 lowercase hex chars before sending to the
API. Omitting `committish_expression` reverts to the original
log+drop-on-missing-release behavior.

## Out of scope (explicit non-goals)

- Vulnerability data (CVE/GHSA overlays, VEX). Modeling allows a
  follow-up to attach `Vulnerability` nodes to `ComponentRelease`
  later.
- "Which projects use X?" reverse-search UI. The data is reachable
  via Cypher; surface in a later pass.
- Cross-release diffs on the Dependencies tab.
- Storing the raw SBoM document.
- A 1.5↔1.7 transitional acceptance window.
- Webhook secret/signature handling specific to SBoM submissions —
  uses the existing notification webhook auth.
