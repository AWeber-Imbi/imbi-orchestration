# Actionable Project Doctor + per-user analysis credentials

## Context

Project Doctor (`plans/project-doctor.md`,
`plans/github-project-doctor.md`) established the `AnalysisPlugin`
contract and the GitHub doctor. Findings are **diagnostic only**:
`AnalysisPlugin.analyze()` returns `AnalysisResultItem`s with
`slug`/`title`/`description`/`status`, and the host
(`imbi-api/src/imbi_api/endpoints/project_analysis.py`) renders them in
the Doctor panel. There is no contract for *fixing* a finding.

Two consequences motivate this feature:

1. **Findings aren't actionable.** The GitHub doctor's fail messages
   name the remedy in prose ("Re-run the lifecycle plugin to repair the
   edge") but there is no user-facing way to run the lifecycle plugin,
   and no per-finding fix. The only auto-fix is the hard-coded
   blueprint-defaults path: `check_blueprint_compliance` /
   `apply_blueprint_defaults` / `remove_stale_blueprint_properties` in
   `blueprint_compliance.py`, surfaced by the bespoke
   `POST .../analysis/apply-blueprint-defaults` endpoint and the
   "Apply Blueprint Fixes" button in `ProjectDoctorTab.tsx`.

2. **Analysis plugins can't use the acting user's credentials.**
   `resolve_analysis_plugins` never sets `identity_plugin_id`, and
   `project_analysis._credentials_for` only resolves static credentials
   (`get_plugin_credentials`: `api_token` blob or OAuth app
   `client_id`/`client_secret`). The GitHub doctor therefore depends on
   a static operator-pasted `access_token`; without it, a private GHEC
   repo returns 401 and every body-dependent check degrades to `warn`.
   Deployment/lifecycle already mint a per-user token via
   `imbi_api.identity.host_integration.attach_identity` +
   `call_with_identity_retry`.

The enabling insight for the fix model: **plugins never touch the DB.**
Lifecycle plugins effect graph changes by setting `ctx.service_writeback`
/ `ctx.link_writeback`; the host captures them
(`lifecycle_dispatch._invoke_one`) and persists via
`endpoints/_helpers.py` (`persist_service_writeback`,
`persist_link_writeback`, which wrap `_merge_exists_in`,
`update_project_link`, `merge_project_links`). Remediation reuses this
exact channel — so a plugin's `remediate()` can repair an `EXISTS_IN`
edge or a project link with no new DB-write layer.

## Decisions

User-confirmed choices (verbatim where it matters):

1. **Remediation model — "Plugin owns its fix" (symmetric
   `analyze` / `remediate`).** A finding advertises a fix; the host
   calls back into the *same* plugin to perform it. (Chosen over a
   host-interpreted command vocabulary and a delegate-to-lifecycle
   `reconcile` capability.)

2. **Remediation effects graph changes through the existing write-back
   channel**, not a new DB layer. `remediate()` recomputes the
   corrected state, sets `ctx.service_writeback` / `ctx.link_writeback`,
   and the host persists with the helpers lifecycle dispatch already
   uses. The GitHub doctor reuses `lifecycle.py`'s writeback-emission
   helper (`_record_repo`) — same package, no duplication.

3. **Per-user credentials use the sibling identity plugin on the same
   TPS.** The doctor is discovered via
   `(:Project)-[:EXISTS_IN]->(:ThirdPartyService)-[:HAS_PLUGIN]->(:Plugin)`,
   which has no edge to carry an `identity_plugin_id`; instead resolve
   the **identity** plugin attached (`HAS_PLUGIN`) to the same TPS.
   Tiebreak when >1: prefer `used_as_login`, else log and leave unset
   (falls back to the static `access_token`).

4. **Blueprint doctor folds into the same flow.** Its findings gain
   remediation offers (`use-default` → set the property's default;
   `stale` → remove it). Because the built-in check has direct DB
   access (it is host code, not a sandboxed plugin), the remediation
   endpoint routes `plugin_id == 'built-in'` to a host-side
   `remediate_blueprint(...)` that performs the single property write
   (reusing the guts of `apply_blueprint_defaults` /
   `remove_stale_blueprint_properties`, scoped to one property). The
   bespoke `apply-blueprint-defaults` endpoint is **removed**; its
   "do everything" UX is replaced by a generic `remediate-all`.

5. **`remediate-all` is best-effort per finding.** It loops over every
   fixable finding in the current report and returns a per-finding
   summary; it does not abort on the first failure.

6. **Destructive flag.** "Make Imbi match GitHub" repairs default to
   `destructive=false`; recreating a missing `EXISTS_IN` edge and
   removing a stale property default to `destructive=true` (UI shows a
   confirm).

7. **`remediate()` is idempotent.** Each implementation re-verifies the
   discrepancy against fresh state and returns `noop` when already
   correct, so a double-click or a `remediate-all` over an
   already-fixed finding is safe.

8. **A finding is fixable iff it carries a `RemediationOffer`.** The UI
   shows a Fix button only when `remediation` is present;
   `remediate()` is only invoked for such findings (a defensive default
   raises `PluginRemediationNotSupported`).

## Critical files

### imbi-common (publish first — consumers pin to it)

| File | Change |
|---|---|
| `src/imbi_common/plugins/base.py` | New `RemediationOffer`, `RemediationResult`; `AnalysisResultItem.remediation: RemediationOffer \| None = None`; `AnalysisPlugin.remediate(...)` optional method (default raises). |
| `src/imbi_common/plugins/errors.py` | New `PluginRemediationNotSupported`, `PluginRemediationFailed`. |
| `src/imbi_common/plugins/__init__.py` | Export the new public types. |
| `docs/plugins/index.md` | Document the remediation contract (analysis section). |

Reference (read-only): `base.py` `LifecycleResult`/`ServiceWriteback`/
`LinkWriteback`/`PluginContext` (writeback channel shape).

### imbi-plugin-github

| File | Change |
|---|---|
| `src/imbi_plugin_github/doctor.py` | Attach `RemediationOffer` to fixable findings in `analyze()`; implement `remediate(ctx, credentials, remediation_id)`. |
| `tests/test_doctor.py` | Remediation tests (respx). |

Reference (read-only): `src/imbi_plugin_github/lifecycle.py:557-602`
(`_record_repo` / `_repo_html_url` — reuse for writeback emission);
`src/imbi_plugin_github/_hosts.py`, `_repos.py`.

### imbi-api

| File | Change |
|---|---|
| `src/imbi_api/plugins/resolution.py` | `resolve_analysis_plugins`: also resolve the sibling identity plugin on the TPS and set `ResolvedPlugin.identity_plugin_id`. |
| `src/imbi_api/endpoints/project_analysis.py` | `_run_one` gains `auth`, wraps `analyze()` in `attach_identity` + `call_with_identity_retry`; `IdentityRequiredError` → `warn` finding. New `POST .../analysis/remediate` and `POST .../analysis/remediate-all`. Remove `apply-blueprint-defaults`. |
| `src/imbi_api/blueprint_compliance.py` | Findings carry `RemediationOffer`s; add `remediate_blueprint(db, project_id, type_slugs, remediation_id) -> RemediationResult` (single-property set/remove). |

Reference (read-only):
`src/imbi_api/identity/host_integration.py`
(`attach_identity`, `call_with_identity_retry`);
`src/imbi_api/plugins/lifecycle_dispatch.py:210-325`
(`_invoke_one` writeback capture);
`src/imbi_api/endpoints/_helpers.py`
(`persist_service_writeback`, `persist_link_writeback`,
`_merge_exists_in`, `update_project_link`, `merge_project_links`,
`lookup_project_*`).

### imbi-ui

| File | Change |
|---|---|
| `src/api/endpoints.ts` | `AnalysisResult.remediation?: RemediationOffer`; `remediateAnalysisFinding(...)`, `remediateAllAnalysisFindings(...)`; remove `applyProjectBlueprintDefaults`. |
| `src/components/ProjectDoctorTab.tsx` | Per-finding "Fix" button in `ResultPanel` (confirm when `destructive`); replace "Apply Blueprint Fixes" with "Fix all"; toast `RemediationResult.message`; invalidate the analysis query. |
| `src/types/api-generated.ts` | Regenerated by the user (`npm run codegen:fetch`). |

## Implementation outline (dependency order)

1. **imbi-common** — add `RemediationOffer` / `RemediationResult`, the
   optional `remediate()`, the two errors; export + docs. Bump version
   and publish (or use the established temp `uv.sources` pin in
   consumers during dev — see the meta-repo's pending-cleanup notes).
2. **imbi-plugin-github** — `doctor.analyze()` attaches offers;
   `doctor.remediate()`:
   - `identifier-match` / `canonical-url-shape` / `dashboard-url-match`
     → re-fetch the repo, emit
     `ServiceWriteback(identifier=str(api_id),
     canonical_url=f'{api_base}/repositories/{api_id}',
     dashboard_links={slug: html_url})` (via the shared `_record_repo`).
   - `github-repository-link-match` → emit
     `LinkWriteback(link_key='github-repository', new_url=html_url)`.
   - `exists-in` (missing edge) → recreate via the same writeback.
3. **imbi-api**
   - `resolution.resolve_analysis_plugins`: sibling-identity resolution.
   - `project_analysis._run_one`: identity hydration + 401 retry;
     `IdentityRequiredError` → `warn`.
   - `POST .../analysis/remediate {plugin_id, finding_slug,
     remediation_id}`: resolve plugin (re-derive from the analysis
     fan-out), build ctx (+identity), capture writebacks like
     `_invoke_one`, call `remediate()` under `call_with_identity_retry`,
     persist via `_helpers`, re-run analysis, return the fresh report.
     Route `plugin_id == 'built-in'` to `remediate_blueprint`.
   - `POST .../analysis/remediate-all`: loop fixable findings,
     best-effort, per-finding summary.
   - Remove `apply-blueprint-defaults`.
   - `blueprint_compliance`: attach offers + `remediate_blueprint`.
4. **imbi-ui** — types, endpoints, Fix / Fix-all buttons, remove the
   bespoke button. User regenerates `api-generated.ts`.
5. **meta-repo** — bump submodule pointers after component PRs merge.

## Tests

- **imbi-common**: `RemediationOffer`/`RemediationResult` model
  validation; `AnalysisResultItem.remediation` default `None`;
  base `remediate()` raises `PluginRemediationNotSupported`.
- **imbi-plugin-github** (`tests/test_doctor.py`, respx): each fixable
  finding's `analyze()` carries the expected offer; `remediate()`
  emits the correct `ServiceWriteback` / `LinkWriteback`; `noop` when
  already correct; `remediate()` for an unknown id raises; 401 during
  remediate surfaces as `failed` (and triggers the host retry path in
  the api tests).
- **imbi-api**:
  - `resolve_analysis_plugins` sets `identity_plugin_id` from the
    sibling identity plugin; tiebreak by `used_as_login`; unset when
    none.
  - `_run_one` hydrates identity, retries once on 401,
    `IdentityRequiredError` → `warn` finding.
  - `POST .../analysis/remediate` persists the writeback (assert the
    `EXISTS_IN` edge / link changed) and returns a refreshed report;
    routes `built-in` to blueprint remediation.
  - `remediate-all` best-effort: one failing finding doesn't block the
    rest; per-finding summary shape.
  - `apply-blueprint-defaults` route is gone (404).
  - `blueprint_compliance.remediate_blueprint` sets one default /
    removes one stale property; idempotent `noop`.
- **imbi-ui**: Fix button shows only when `remediation` present;
  destructive routes through `ConfirmDialog`; success toasts message +
  invalidates the analysis query; "Fix all" wired to `remediate-all`.

## Verification

Per submodule (run in the submodule you edited):

```bash
# imbi-common
just lint && just test            # coverage >= 90%

# imbi-plugin-github
just lint && just test            # coverage >= 85%
just test tests/test_doctor.py    # during iteration

# imbi-api
just lint && just test
just test tests/.../test_project_analysis.py

# imbi-ui
npm run lint && npm test
```

End-to-end spot-check (okteto dev env, project `dOMnDfJCBIaDAmH2GfOHn`
on the `github` TPS, GHEC Doctor instance attached):

1. With a connected GitHub identity and **no** static `access_token`,
   run analysis → body-dependent checks resolve (no longer all-`warn`),
   confirming per-user creds via the sibling identity plugin.
2. Corrupt the `EXISTS_IN` identifier in the graph; re-run → the
   `identifier-match` finding shows a **Fix** button; click → toast
   "fixed"; report refreshes to `pass`; confirm the edge was rewritten:
   ```
   kubectl exec -n daves -i -c postgres sts/postgres -- \
     psql -U postgres -d imbi -c "LOAD 'age'; SET search_path=ag_catalog,public; \
     SELECT * FROM cypher('imbi', \$\$ MATCH (:Project {id:'dOMnDfJCBIaDAmH2GfOHn'}) \
     -[e:EXISTS_IN]->(:ThirdPartyService {slug:'github'}) RETURN e.identifier \$\$) AS (id agtype);"
   ```
3. Unset a blueprint-defaulted property; run analysis → `use-default`
   finding shows **Fix**; click → property set. Confirm the bespoke
   "Apply Blueprint Fixes" button is gone and "Fix all" remediates
   every fixable finding in one action.
