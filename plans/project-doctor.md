# Project Doctor — Plugin-driven Project Analysis

## Context

The Imbi project page already surfaces a **Health & Compliance** score on the Overview tab and exposes two utility actions (`Recompute Score`, `Sync Deployments`) on the Project Settings tab. Today the score is the only source of "how is this project doing?" signal, computed from a fixed catalog of scoring-policy categories (`attribute`, `presence`, `link_presence`, `age`) that read only static project attributes / links.

The Project Doctor feature (`imbi-orchestration/features/project-doctor.md`) adds a richer, plugin-driven view:

1. A new **Doctor panel** (project tab) opened from a stethoscope icon next to the project-settings cog, colour-coded by the worst observed analysis status.
2. A new **analysis plugin type** that lets plugins (logzio, github, sonarqube, …) inspect a project and return a list of `pass | warn | fail` result items with a title and markdown body.
3. **Persistence of the latest report** per project (graph) so opening the panel is cheap; an `Analyze` button re-runs all applicable plugins.
4. A new **scoring-policy category** (`analysis_result`) so an individual analysis result can feed the existing Health & Compliance score in addition to standing alone in the Doctor panel.

The plugin association rules match the existing Imbi conventions: the analysis plugin is attached to the **project type** via `USES_PLUGIN` (operator-installed default, project-edge override), and *additionally* picked up from any **`ThirdPartyService`** the project `EXISTS_IN` when that TPS has a `HAS_PLUGIN` edge to an analysis plugin. Identity-plugin discovery in the deployment-webhook plan uses the same TPS fan-out pattern.

## Decisions (chosen defaults, not asked)

- **New plugin type literal**: extend `PluginManifest.plugin_type` with `'analysis'`. New ABC `AnalysisPlugin` with a single `async analyze(ctx, credentials) -> list[AnalysisResultItem]` method. Mirrors `LogsPlugin.search`; one method per plugin type, no `ActionDescriptor` catalog (we don't need multi-action dispatch for v1).
- **Result item shape** (in `imbi_common.plugins.base`):
  ```python
  class AnalysisResultItem(pydantic.BaseModel):
      slug: str                                          # stable per-plugin id
      title: str
      description: str                                   # markdown body
      status: typing.Literal['pass', 'warn', 'fail']
  ```
- **Plugin discovery for a project**: union of
  1. Plugins attached via `(Project)-[:USES_PLUGIN {tab:'analysis'}]->(Plugin)` (project override).
  2. Plugins attached via `(Project)-[:TYPE]->(ProjectType)-[:USES_PLUGIN {tab:'analysis'}]->(Plugin)` (type default), with the project override taking precedence per plugin slug — exactly how `resolve_plugin` already merges tiers.
  3. Plugins attached via `(Project)-[:EXISTS_IN]->(ThirdPartyService)-[:HAS_PLUGIN]->(Plugin)` filtered to `Plugin.plugin_type='analysis'`. (No `tab` filter — TPS-attached analyse-everything plugins don't carry an edge tab.)
- **Trigger model**: **synchronous in-process**, like `resync_for_project`. `POST .../analysis/run` resolves all applicable plugins, runs `analyze()` for each via `asyncio.gather`, persists the merged report, returns it. Plugin failures are captured per-plugin (`status='fail'`, `description='<error>'`) without aborting other plugins. No Valkey queue in v1; the latency-sensitive `Recompute Score` path is the precedent for moving to a queue *if* doctor runs grow slow.
- **Persistence**: keep only the **latest** report per project (the feature only requires "last analysis report"). Graph shape:
  - `(:Project)-[:HAS_ANALYSIS_REPORT]->(:AnalysisReport {id, created_at, triggered_by_user_id, overall_status})`
  - `(:AnalysisReport)-[:HAS_RESULT]->(:AnalysisResult {slug, title, description, status, plugin_slug, plugin_id})`
  - On re-run: delete the existing report subgraph for the project, then create the new one in a single transaction.
- **Overall status** = worst of all result statuses (`fail > warn > pass`); drives the doctor-icon colour.
- **Auto-analyse on first open**: when the UI mounts the Doctor panel and `GET .../analysis` returns 404, it fires the `POST .../analysis/run` mutation once. Subsequent opens read the cached report.
- **Scoring-policy integration**: new discriminator variant `AnalysisResultPolicy(category='analysis_result', result_slug: str, status_score_map: dict[Literal['pass','warn','fail'], int])`. Scoring engine resolves the result with that `slug` from the project's latest `AnalysisReport` and uses the mapped score; missing report → policy contributes `None` (same convention as other categories).
- **Permissions**: `project:read` to fetch a report, `project:write` to trigger one. Scoring-policy admin uses existing `scoring_policy:read`/`scoring_policy:write`.
- **UI placement**: in `ProjectDetail.tsx` the Doctor tab is rendered *immediately to the left of the Settings tab*, also pinned right with a Stethoscope icon. Both icon-only tabs share the right cluster (drop `ml-auto` from Settings, wrap both in a right-aligned `<div>` inside `TabsList`, or apply `ml-auto` to the doctor tab and leave Settings as-is). Doctor panel shows: utility bar (Analyze / Recompute / Sync), Health & Compliance card (breakdown expanded by default), Analysis Results sections grouped fail → warn → pass.
- **Stethoscope colour**: reuse `ScoreBadge` colour ramp but driven by `overallStatus`: `fail → bg-danger`, `warn → bg-warning`, `pass → bg-success`, no report → `text-tertiary`.
- **Tab visibility**: the Doctor tab is rendered for *every* project (even when no analysis plugins resolve) because the Health & Compliance breakdown is always meaningful. The Analyze button is hidden when zero analysis plugins resolve, with a tooltip explaining why.

## Critical files

### imbi-common

- `src/imbi_common/plugins/base.py:98-105` — extend `PluginManifest.plugin_type` literal with `'analysis'`.
- `src/imbi_common/plugins/base.py` — append:
  - `AnalysisResultItem` (model: slug, title, description, status).
  - `AnalysisPlugin(abc.ABC)` with `async analyze(ctx, credentials) -> list[AnalysisResultItem]`.
- `src/imbi_common/scoring/models.py:192-195` — extend `ScoringPolicy` discriminator union with `AnalysisResultPolicy`; extend `PolicyContribution.category` literal with `'analysis_result'`.
- `src/imbi_common/graph/schemata.toml` — register new vertex labels `AnalysisReport`, `AnalysisResult` with indexes (`AnalysisReport.project_id`, `AnalysisResult.report_id` — non-unique).

### imbi-api

- `src/imbi_api/plugins/resolution.py` — add `resolve_analysis_plugins(db, project_id) -> list[ResolvedPlugin]` that unions the three sources above (USES_PLUGIN tab='analysis' on project + project-type, plus HAS_PLUGIN via EXISTS_IN filtered to `plugin_type='analysis'`). Dedupe by `plugin_id`.
- **New file** `src/imbi_api/endpoints/project_analysis.py` — router mounted under `/organizations/{org_slug}/projects/{project_id}/analysis`:
  - `POST /run` (perm `project:write`): resolve plugins, run `analyze()` per plugin via `asyncio.gather`, replace persisted report, return `AnalysisReport`.
  - `GET /` (perm `project:read`): return the latest persisted report or 404.
- `src/imbi_api/endpoints/__init__.py` — register the new router.
- `src/imbi_api/endpoints/scoring_policies.py:84-96` — extend `_validate_policy_row` / `_JSON_PROPS_KEYS` / `_NODE_PROPERTY_KEYS` to cover the new `status_score_map` JSON prop.
- `src/imbi_api/scoring/engine.py` (or wherever per-policy evaluation dispatches by category) — implement `AnalysisResultPolicy.evaluate` lookup: fetch the project's latest `(:AnalysisResult {slug: policy.result_slug, ...})` via the `HAS_RESULT` edge, map status → score. Touch only the dispatch path; no rewrites elsewhere.
- `src/imbi_api/domain/scoring.py` — extend `PolicyCreate`/`PolicyUpdate`/response models with the new variant fields.

### imbi-ui

- `src/components/ProjectDetail.tsx:625-662` — add a `'doctor'` tab object before `'settings'`, render with `Stethoscope` icon tinted by `overallStatus`. Stack both icon-only tabs to the right.
- `src/components/ProjectDetail.tsx:877+` — render `<TabsContent value="doctor">` with the new `<ProjectDoctorTab>` component.
- **New file** `src/components/ProjectDoctorTab.tsx`:
  - `useQuery(['projectAnalysis', orgSlug, projectId])` → `getProjectAnalysis(...)`. On 404, fire `useMutation(runProjectAnalysis(...))` once.
  - Utility bar: `Analyze` (mutation), `Recompute Score` (reuse `rescoreProject`), `Sync Deployments` (reuse `useProjectDeploymentResync`). Permission checks mirror `ProjectSettingsTab` (`scoring_policy:rescore`, `project:deployment:write`, plus new `project:write` for analyze).
  - Health & Compliance card: lift the existing JSX (currently inline in `ProjectDetail.tsx:971-1014`) into a shared `<HealthCompliancePanel>` component used by both Overview and Doctor; in Doctor render with `breakdownOpen={true}` initial state.
  - Analysis Results section: group by status (fail → warn → pass), each result a `<Collapsible>` panel rendering markdown via the existing markdown component (search for `react-markdown` usage in the repo).
- `src/api/endpoints.ts` — add `getProjectAnalysis(orgSlug, id, signal)` and `runProjectAnalysis(orgSlug, id)`.
- `src/types/api-generated.ts` — regenerated by `npm run codegen:fetch` after the API ships; the user owns the regen step.
- `src/components/admin/scoring-policies/ScoringPolicyForm.tsx:48-56` — extend `CATEGORY_LABELS` / `CATEGORY_DESCRIPTIONS` with the `analysis_result` entry; add form fields (result slug picker / free text + 3 status→score inputs).

### Plugins (optional, follow-up)

- Existing analysis-capable plugins (logzio, sonarqube) can declare `plugin_type='analysis'` and implement `AnalysisPlugin.analyze` in a follow-up PR. Not required for the v1 cut — the Doctor panel can ship with no plugins assigned and still surface the Health & Compliance breakdown.

## Implementation outline

1. **imbi-common — types & schema**.
   - Add `'analysis'` to `PluginManifest.plugin_type`.
   - Add `AnalysisResultItem` and `AnalysisPlugin` ABC.
   - Add `AnalysisResultPolicy` variant + extend discriminator + `PolicyContribution.category` literal.
   - Register `AnalysisReport` and `AnalysisResult` in `graph/schemata.toml`.
   - Tests: pydantic validation round-trip, discriminator routing, plugin-type literal accepted.

2. **imbi-api — resolution**.
   - Add `resolve_analysis_plugins` to `plugins/resolution.py` returning a deduped list of `ResolvedPlugin`. Use a single Cypher query that gathers all three sources via three `OPTIONAL MATCH` blocks and a `UNION` (or `collect()` + post-merge in Python — match the style of `resolve_plugin`).
   - Tests: project-type-only, project-override wins, TPS-only via EXISTS_IN, mixed (override + TPS together), no plugins.

3. **imbi-api — endpoints**.
   - New router `project_analysis.py` with the two routes. The `POST /run` body:
     ```python
     plugins = await resolve_analysis_plugins(db, project_id)
     async def _run(rp):
         try:
             items = await rp.entry.instance.analyze(ctx, creds)
             return rp, items, None
         except Exception as exc:
             LOGGER.exception(...)
             return rp, [], exc
     outcomes = await asyncio.gather(*(_run(rp) for rp in plugins))
     report = _materialize_report(outcomes, triggered_by=auth.user.id)
     await _persist(db, project_id, report)   # delete-then-create txn
     return report
     ```
   - Persistence helper does: `MATCH (p:Project {id:...})-[:HAS_ANALYSIS_REPORT]->(r) DETACH DELETE r, r--()`, then `CREATE (p)-[:HAS_ANALYSIS_REPORT]->(r:AnalysisReport {...}) WITH r UNWIND $items AS i CREATE (r)-[:HAS_RESULT]->(:AnalysisResult i)`.
   - Tests: 404 when no report, happy path with two plugins, plugin failure surfaces as fail result, re-run replaces previous report, permissions enforced.

4. **imbi-api — scoring**.
   - Add `analysis_result` dispatch arm in the scoring engine evaluation path. Reuse the existing project-attribute fetch flow but read from `(:Project)-[:HAS_ANALYSIS_REPORT]->(:AnalysisReport)-[:HAS_RESULT {slug: policy.result_slug}]->(:AnalysisResult)`.
   - Tests: policy with `pass→100, warn→50, fail→0` mapping over each persisted status; missing report → contribution `None`.

5. **imbi-ui — Doctor tab**.
   - Lift Health & Compliance JSX into `<HealthCompliancePanel breakdownOpen={...} />`, used by Overview tab and Doctor.
   - Add `<ProjectDoctorTab>` and wire the tab/content into `ProjectDetail.tsx`.
   - Stethoscope icon coloured from `overallStatus` (or neutral when report absent). Tooltip "Project Doctor".
   - Result groups: `<Collapsible defaultOpen={status !== 'pass'}>` per item — fail/warn open by default, pass collapsed.
   - Auto-analyse on first mount when no report exists.

6. **imbi-ui — scoring-policy admin form**.
   - Extend the category map + add a form section for the new variant. Validation uses the generated TS types after `npm run codegen:fetch`.

7. **Docs**.
   - Update `imbi-orchestration/notes/imbi-api-graph-schema.md` with the new node labels and edge.
   - `imbi-common/docs/api/` — public-API surface change (new plugin type, new ABC) requires a doc page edit per `imbi-common/CLAUDE.md`.

## Tests

- **imbi-common**: discriminator round-trip; `AnalysisPlugin` ABC enforces `analyze`; `AnalysisResultItem` validates the status literal; `AnalysisResultPolicy` evaluation table-driven test.
- **imbi-api**:
  - `tests/plugins/test_resolution.py` — `resolve_analysis_plugins` matrix above.
  - `tests/endpoints/test_project_analysis.py` — full request/response, permissions (`project:read`/`project:write`), re-run idempotency, plugin failure capture, 404 on empty.
  - `tests/scoring/test_engine.py` — `AnalysisResultPolicy` integration with the scoring sweep (existing pattern in this file).
- **imbi-ui** — component-level: Doctor tab renders given a fixture report; Analyze button calls mutation; result panels render markdown; severity-coloured icon renders. Use the existing component-test patterns (search for `vitest` setup in the repo).

## Verification

1. Per submodule: `just lint && just test` clean (coverage ≥ 90% in `imbi-common`; ≥ 90% in `imbi-api` for new modules, allowing for the repo-wide ~30% baseline).
2. End-to-end local dev:
   - `just serve` `imbi-api`; create org, project type, project; install a stub analysis plugin (or use a temporary in-process plugin registered via entry point) that returns a fixed `pass`/`warn`/`fail` triplet.
   - Verify `POST /organizations/{org}/projects/{id}/analysis/run` returns the expected report; `GET ...` returns the same; running again replaces the previous report.
   - Open the project in the UI → Doctor tab → confirm the panel shows the score breakdown (expanded), the three buttons (with permission gating), and the three result panels grouped fail → warn → pass.
   - Create an `analysis_result` scoring policy targeting the fail-slug; trigger `Recompute Score`; confirm the new contribution lands in the breakdown.
3. Doctor icon colour: change a plugin to return a `fail`, re-run analyse → icon renders `bg-danger`; flip to all-pass → `bg-success`.
4. Submodule pointer bump (in `imbi-development` meta-repo) only after all three submodule PRs are merged on `main`.
