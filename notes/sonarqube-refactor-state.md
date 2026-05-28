# SonarQube webhook plugin — refactor state

Context dump for resuming the `features/sonarqube-plugin.md` work. The
authoritative plan lives at `plans/sonarqube-plugin.md`; this file
captures the *state of the working tree* and the decisions that aren't
obvious from a `git diff`.

## Where things stand

### imbi-common (working tree, uncommitted)

Modified:

- `src/imbi_common/plugins/base.py` — added `'webhook'` to
  `PluginManifest.plugin_type` Literal; added abstract base
  `WebhookActionPlugin` with abstract method
  `run_action(self, ctx: PluginContext, credentials: dict[str, str],
  external_identifier: str, action: str, action_config: str,
  payload: object)`. **No `WebhookActionContext`** — was removed.
- `src/imbi_common/plugins/registry.py` — `'webhook'` is a recognized
  `plugin_type` and `WebhookActionPlugin` is in the `expected_base`
  switch; mismatch detection covers it.
- `src/imbi_common/plugins/__init__.py` — exports
  `WebhookActionPlugin`, `get_plugin_credentials`,
  `get_plugin_configuration_keys`, `patch_plugin_configuration`.
- `tests/test_plugins/test_base.py` and
  `tests/test_plugins/test_registry.py` — cover the new abstract
  method and registry round-trip.

New:

- `src/imbi_common/plugins/credentials.py` — moved from
  `imbi_api/plugins/credentials.py`. Pure graph + auth.encryption,
  no fastapi imports. Same behaviour: `api_token`/`aws-iam-ic`
  plugins decrypt from `Plugin.plugin_configuration`;
  `oauth2`/`oidc` plugins walk `Plugin -[:USES_APPLICATION]->
  ServiceApplication`.

Released version on PyPI is **2.1.0**; my changes still need a release
bump (≥ 2.1.1) before downstream packages can install them. Local
`pyproject.toml` still says `version = "0.10.0"` — that's the
working-tree number, not the published one.

In parallel, PR
[#106](https://github.com/AWeber-Imbi/imbi-common/pull/106) on
`feature/imbi-api-client` adds a shared
**`imbi_common.api`** subpackage:

- `imbi_common.api.client.Imbi(httpx.AsyncClient)` — bookkeeping
  helpers (`patch_project`, `find_user_by_identity`,
  `create_release`, `record_deployment`) lifted from
  `imbi_gateway.actions.ImbiClient`.
- `imbi_common.api.settings.Settings(BaseSettings)` — env-driven
  config with prefix `IMBI_CLIENT_`
  (`IMBI_CLIENT_API_BASE_URL`, `IMBI_CLIENT_API_TOKEN` as
  `pydantic.SecretStr`, `IMBI_CLIENT_USER_AGENT`).

Once that PR ships in the next imbi-common release, every service
that talks back to imbi-api should drop its bespoke wiring and
import `from imbi_common.api import Imbi, Settings`. That includes
the gateway's `ImbiClient` and the sonarqube plugin's
`actions.py` httpx call — see the per-submodule sections below.

### imbi-api (working tree, uncommitted)

- `src/imbi_api/plugins/credentials.py` is now a 17-line shim that
  re-exports from `imbi_common.plugins.credentials`. Every existing
  callsite in imbi-api keeps working unchanged.

### imbi-gateway (working tree, uncommitted)

Modified:

- `pyproject.toml` — adds `imbi-plugin-sonarqube>=0.1.0` runtime dep
  and registers `gateway-actions =
  "imbi_gateway.plugin:GatewayActionsPlugin"` under
  `[project.entry-points."imbi.plugins"]`.
- `src/imbi_gateway/app.py` — calls
  `plugin_registry.load_plugins()` at the top of `create_app()` so
  the registry is populated before any request lands.
- `src/imbi_gateway/actions.py` — three legacy handlers
  (`update_project`, `create_release`, `add_deployment_event`) now
  take `(ctx: PluginContext, action_config: str, payload: object)`
  and pull `org_slug`/`project_id`/`actor_user_id` from `ctx`. The
  `ImbiClient` httpx wrapper is unchanged **for now** — once
  imbi-common ships `imbi_common.api` (PR #106), drop this class and
  switch the three handlers to
  `imbi_common.api.Imbi(base_url=..., token=..., user_agent=...)`
  fed from `imbi_common.api.Settings`. The `ACTIONS_IMBI_URL` /
  `ACTIONS_IMBI_TOKEN` env vars get retired in the same move
  (replaced by `IMBI_CLIENT_API_BASE_URL` /
  `IMBI_CLIENT_API_TOKEN`).
- `src/imbi_gateway/notifications.py` — major rewrite:
  - `WebhookRule.handler` is a `str` of the form
    `"<plugin_slug>:<action_name>"`. Field validator rejects missing
    `:`, empty slug, empty action. New `plugin_slug` and
    `action_name` properties.
  - Cypher now collects `{plugin_id, plugin_slug}` per `HAS_PLUGIN`
    (no more `USES_APPLICATION → ServiceApplication.encrypted_credentials`).
  - Project-resolution Cypher gains `OPTIONAL MATCH
    (p)-[:OWNED_BY]->(t:Team)` and returns `project_slug` +
    `team_slug`.
  - Dispatch chain: `_record_and_dispatch` → `_run_handlers` →
    `_invoke_plugin` → `_resolve_credentials_for_rule`. Uses
    `imbi_common.plugins.registry.get_plugin` + new
    `imbi_common.plugins.credentials.get_plugin_credentials`.
  - Plugins with empty `manifest.credentials` always receive `{}`;
    credentialed plugins skip with a warning when no `Plugin` node
    is attached to the TPS.
  - The gateway stashes `service_slug` and `service_endpoint` (from
    the matched TPS) on `PluginContext.assignment_options`. Plugins
    that need the TPS endpoint read it from there.

New:

- `src/imbi_gateway/plugin.py` — `GatewayActionsPlugin(WebhookActionPlugin)`
  with `slug='gateway-actions'`, no declared credentials.
  `run_action` dispatches `update_project` / `create_release` /
  `add_deployment_event` by `action` name to the matching function
  in `actions.py`. Raises `ValueError` for unknown actions.

Tests rewritten:

- `tests/test_actions.py` — every call site now uses
  `_ctx()` returning a `PluginContext` and passes
  `(ctx, action_config, payload)` positionally. All 38 tests pass.
- `tests/test_notifications.py` — rewritten:
  - Stub plugins (`StubWebhookPlugin`, `CredentialedStubPlugin`,
    `RaisingStubPlugin`) are injected directly into
    `plugin_registry._registry` via `_register_stub_plugins` /
    `_unregister_stub_plugins` (`asyncSetUp` / `asyncTearDown`).
  - `_attach_plugin(plugin_slug=..., credentials=...)` writes a
    `Plugin` node with `plugin_configuration` (Fernet-encrypted JSON
    of credentials) directly. No `ServiceApplication`.
  - `_StubCall` dataclass captures every `run_action` invocation;
    `HANDLER_CALLS: list[_StubCall]`.
  - New cases: credentialless plugin runs without attachment;
    credentialed plugin gets decrypted creds when attached;
    credentialed plugin is skipped when not attached; unknown
    plugin slug logs error + skips; malformed handler string fails
    rule validation. 19/19 unit tests pass; the AGE-dependent
    `ProcessNotificationTests` need `just test` (Docker) to run.

### imbi-plugin-sonarqube (new package, working tree)

Lives at `imbi-plugin-sonarqube/` in the meta-repo, not yet a
submodule.

- `pyproject.toml` — `name='imbi-plugin-sonarqube'`,
  `version='0.1.0'`, deps `httpx`, `imbi-common~=2.1`,
  `jsonpointer`, `pydantic`. Dev deps include
  `imbi-common[server,databases]~=2.1`. Entry point:
  `sonarqube = "imbi_plugin_sonarqube.plugin:SonarqubePlugin"`.
  `[tool.uv] constraint-dependencies =
  ["clickhouse-connect>=0.12rc1"]` lets the transitive prerelease
  resolve.
- `src/imbi_plugin_sonarqube/plugin.py` — `SonarqubePlugin(WebhookActionPlugin)`
  with single `api_token` credential. `run_action` raises on unknown
  actions; the only known action is
  `'update_project_from_webhook'`.
- `src/imbi_plugin_sonarqube/actions.py` — keyword-only
  `update_project_from_webhook(*, ctx, credentials,
  external_identifier, action_config)`. Reads `service_endpoint`
  from `ctx.assignment_options`, `api_token` from `credentials`;
  PATCHes imbi-api via httpx (reads `ACTIONS_IMBI_URL` /
  `ACTIONS_IMBI_TOKEN` env vars). **Once imbi-common's PR #106
  ships**, replace the bespoke httpx call with
  `async with Imbi(...) as imbi: await imbi.patch_project(...)`,
  source the base URL + token from `imbi_common.api.Settings`
  (`IMBI_CLIENT_API_*`), and drop the `ACTIONS_*` env vars. The
  SonarQube-side `client.py`/`fetch_component_measures` stays as
  is — that talks to SonarQube, not imbi-api.
- `src/imbi_plugin_sonarqube/client.py` — `fetch_component_measures`
  helper with Bearer auth + `SonarqubeClientError` for transport/HTTP
  failures.
- Tests pass 18/18 with 90% coverage.

## Decisions locked in (don't re-litigate)

1. **Storage:** SonarQube credentials live in the existing
   `Plugin.plugin_configuration` blob, set via
   `PATCH /api/organizations/{org}/third-party-services/{slug}/plugins/{plugin_id}/configuration`.
   *Not* on the `HAS_PLUGIN` edge, *not* on a `ServiceApplication`.
2. **Rule shape:** `WebhookRule.handler` is repurposed as
   `"<plugin_slug>:<action_name>"`. No new schema field. The
   imbi-api `WebhookCreate`/`WebhookUpdate`/`WebhookResponse`
   models still treat `handler` as a free-form string, so no
   imbi-api or imbi-ui changes were needed to support the new
   format — operators just write a different string.
3. **Plugin contract:** `run_action(ctx, credentials,
   external_identifier, action, action_config, payload)`. Uses
   the existing `PluginContext`. `credentials` is positional, not
   pulled by the plugin itself, matching `LogsPlugin.search` style.
4. **Built-in plugin:** the three legacy handlers were migrated
   into `gateway-actions` rather than left on a parallel dispatch
   path. Existing rules in production need their `handler` field
   rewritten as `gateway-actions:<action_name>`.
5. **TPS endpoint plumbing:** stashed on
   `PluginContext.assignment_options` as `service_slug` /
   `service_endpoint`. SonarQube plugin reads from there.
6. **`get_plugin_credentials` lives in imbi-common.** imbi-api gets
   a re-export shim so existing call sites don't move.
7. **Single shared Imbi HTTP client.** Once `imbi_common.api`
   (PR #106) is released, every service that calls back to the
   Imbi API uses `imbi_common.api.Imbi` configured from
   `imbi_common.api.Settings`. The gateway's `ImbiClient` and the
   sonarqube plugin's bespoke httpx wiring both retire, and the
   `ACTIONS_IMBI_URL` / `ACTIONS_IMBI_TOKEN` env vars are
   superseded by `IMBI_CLIENT_API_BASE_URL` /
   `IMBI_CLIENT_API_TOKEN`.

## What's verified vs. what's pending

| Check | Status |
| --- | --- |
| `ruff check` clean across imbi-common, imbi-api shim, imbi-gateway, imbi-plugin-sonarqube | ✅ |
| `basedpyright` clean on touched files (pre-existing error in `imbi_common/plugins/schemas.py:193` remains — not from this work) | ✅ |
| imbi-gateway unit tests (`tests/test_actions.py` + non-AGE classes in `tests/test_notifications.py`) | 57/57 pass |
| imbi-plugin-sonarqube tests | 18/18 pass, 90% coverage |
| imbi-common plugin tests | not run locally (postgres-bound conftest); need `just test` |
| `ProcessNotificationTests` in imbi-gateway (AGE-bound) | not run; need `just test` |
| Live wire-up of `imbi-plugin-sonarqube` as a git submodule | not done |

## Next steps (suggested order)

1. **Run the full test suites under Docker** in each submodule
   (`just test` in imbi-common, imbi-gateway, imbi-plugin-sonarqube)
   to exercise the AGE-bound paths.
2. **Cut imbi-common 2.1.1** (or 2.2.0) including the
   `WebhookActionPlugin` + moved `credentials.py` + `'webhook'`
   plugin_type. Commit + push + tag. **If PR #106 lands first**,
   roll the new `imbi_common.api` subpackage into the same release
   so downstream packages can pick up both changes in one bump.
3. **Publish imbi-plugin-sonarqube** wheel so `imbi-gateway`'s
   `imbi-plugin-sonarqube>=0.1.0` runtime dep resolves cleanly. Or,
   for now, install editable in dev: `uv pip install --no-deps -e
   ../imbi-plugin-sonarqube`. (Note: `uv run`/`uv sync` will revert
   editable installs unless you re-run after each sync.)
4. **Bump imbi-common pin** in imbi-plugin-sonarqube and imbi-api
   once 2.1.1+ is released. Update imbi-gateway's
   `imbi-common[databases,server]>=0.8.5` pin too.
5. **Migrate existing rule rows** — production `WebhookRule.handler`
   values like `'imbi_gateway.actions.update_project'` need to
   become `'gateway-actions:update_project'`. Same for the other
   two legacy handlers. The rewrite is a one-time data migration;
   no schema change.
5a. **Adopt `imbi_common.api.Imbi`** in imbi-gateway and
    imbi-plugin-sonarqube once the new imbi-common release is
    available. Rip out `imbi_gateway.actions.ImbiClient` and the
    `ACTIONS_IMBI_URL` / `ACTIONS_IMBI_TOKEN` settings; replace
    with `imbi_common.api.Settings` + `Imbi(...)`. Update env
    var names in any Helm/compose/secret manifests at the same
    time.
6. **Wire `imbi-plugin-sonarqube` as a git submodule** in
   `imbi-development` and adjust `.gitmodules`.
7. **End-to-end smoke test** against a local dev stack (see the
   "Verification" section in `plans/sonarqube-plugin.md`).
8. **PRs** — three correlated PRs per the meta-repo's Stage 4
   workflow: imbi-common, imbi-gateway, imbi-plugin-sonarqube. PR
   metadata should link to `plans/sonarqube-plugin.md`.

## Gotchas hit during the refactor (so they don't bite again)

- `uv pip install -e ../imbi-common` from within a sibling project
  appears to install but the editable .pth doesn't always stick.
  Use `uv pip install --no-deps --refresh -e ../imbi-common` and
  call binaries directly (`.venv/bin/python`, `.venv/bin/pytest`)
  instead of `uv run` — the latter triggers a sync that reverts the
  editable install. See the back-and-forth in the conversation
  history for details.
- `imbi_common/plugins/credentials.py`'s `_read_plugin_configuration`
  raises `ValueError` (not a custom error) when the blob is
  corrupted. Be careful if you wrap `get_plugin_credentials` calls
  to translate exceptions.
- The gateway's notification Cypher uses double-brace escaping for
  `{{ ... }}` because of psycopg's `SQL.format()`. New properties
  inside `{{ ... }}` maps need the same escaping treatment.
- Imbi-common's `plugin_type` Literal is duplicated in the
  registry's `expected_base` if/elif chain — if you add another
  plugin type, update both.

## Files to start reading on a fresh session

1. `plans/sonarqube-plugin.md` — the authoritative plan.
2. This file — for working-tree state + decisions.
3. `imbi-common/src/imbi_common/plugins/base.py` — `WebhookActionPlugin`.
4. `imbi-gateway/src/imbi_gateway/notifications.py` — dispatch.
5. `imbi-gateway/src/imbi_gateway/plugin.py` — built-in plugin.
6. `imbi-plugin-sonarqube/src/imbi_plugin_sonarqube/plugin.py` — the
   SonarQube plugin entry point.
7. `imbi-common/src/imbi_common/api/client.py` (on
   `feature/imbi-api-client`, PR #106) — the shared `Imbi` HTTP
   client both the gateway and the sonarqube plugin should adopt
   once it lands.
