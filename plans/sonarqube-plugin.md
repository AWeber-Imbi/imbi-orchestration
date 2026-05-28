# SonarQube webhook plugin

## Context

`aweber.ghe.com/apis/sonar-imbi-updater` is a v1 webhook handler that
retrieves SonarQube measurements and updates Imbi project facts via
integer fact ids and an OpenSearch project lookup. We want this
functionality inside `imbi-gateway`'s notification pipeline using the
existing project resolution (`ThirdPartyService` slug +
`identifier_selector` → `Project.EXISTS_IN`).

The first cut of this plan introduced a `WebhookActionPlugin` whose
sole entry point was a stringly-typed `run_action(action, ...)` method
that internally dispatched on the action name and re-validated an
opaque `action_config` JSON blob. That contract is being replaced
before it gets implemented:

- Each plugin should *declare* its actions as data (callable +
  configuration-model import strings), not bottleneck them through a
  switch statement.
- Action callables should share one signature and receive a
  pre-validated pydantic instance — not a JSON string to re-parse.
- imbi-api should validate `handler_config` at rule write time against
  the action's configuration model, and expose the model's JSON
  Schema to the UI so rule editors can render meaningful forms.

Operator-visible configuration is unchanged from the previous plan:

- **Third-Party Service:** SonarQube (existing TPS with
  `api_endpoint=https://sonarqube.aweber.io`).
- **Webhook → IMPLEMENTED_BY → TPS** edge:
  - `identifier_selector=/project/key`
- **`sonarqube` plugin instance** attached to the SonarQube TPS via
  `HAS_PLUGIN`; the operator pastes the SonarQube API token through
  `PATCH /api/organizations/{org}/third-party-services/{slug}/plugins/{plugin_id}/configuration`.
- **WebhookRule:**
  - `filter_expression=/branch/is_main==true`
  - `handler="sonarqube#update_project_from_webhook"`
    *(note the `#` separator — see Decisions)*
  - `handler_config=`
    ```json
    [
        {"metric": "coverage", "path": "/test_coverage"},
        {"metric": "ncloc",    "path": "/lines_of_code"}
    ]
    ```

The handler retrieves
`GET {api_endpoint}/api/measures/component?component={external_id}&metricKeys=coverage,ncloc`
using the decrypted SonarQube API token, then sends a JSON-Patch to
`PATCH /organizations/{org}/projects/{project_id}` mapping each
SonarQube metric value into the configured JSON Pointer path.

## Decisions (user-confirmed)

### Plugin contract

- **`WebhookActionPlugin.run_action` is removed.** The plugin no
  longer participates in dispatch; it advertises a catalog of actions
  and the host invokes them directly.
- **New abstract classmethod `actions() -> list[ActionDescriptor]`**
  on `WebhookActionPlugin`. Returns one entry per webhook action the
  plugin exposes. Classmethod (not instance method) because the
  catalog is static metadata; host code calls it via
  `entry.handler_cls.actions()` without instantiating the plugin.
- **`ActionDescriptor` is a pydantic model with five fields:**
  `name`, `label`, `description`, `callable`, `config_model`. The
  last two are `pydantic.ImportString`-typed so the host resolves
  them lazily at dispatch / validation time.
  - `callable: pydantic.ImportString[Callable[..., Awaitable[None]]]`
  - `config_model: pydantic.ImportString[type[pydantic.BaseModel]]`
  - `name` is the action key used in `WebhookRule.handler` after the
    `#` separator; must be unique within a plugin.
  - `label` and `description` are operator-facing UI strings; the UI
    pairs them with `config_model.model_json_schema()` to render the
    rule editor. **No extra per-field metadata is needed on
    `ActionDescriptor` itself** — `config_model` is a pydantic
    `BaseModel`, so `Field(description=..., default=...)` etc. on
    its fields already lands in the JSON Schema the UI consumes.

### Uniform action callable contract

Every action is an async function with this signature, keyword-only:

```python
async def action(
    *,
    ctx: PluginContext,
    credentials: dict[str, str],
    external_identifier: str,
    action_config: SomeConfigModel,  # pre-validated pydantic instance
    payload: object,
) -> None: ...
```

- `ctx` is the standard `PluginContext` (`project_id`, `project_slug`,
  `org_slug`, `team_slug`, `actor_user_id`, `assignment_options`,
  etc.). The gateway stashes `service_slug` and `service_endpoint`
  into `ctx.assignment_options`.
- `credentials` is the decrypted blob from the `Plugin` node on the
  TPS, keyed by `manifest.credentials[*].name`. Plugins that declare
  no credentials always receive `{}`.
- `external_identifier` is the value resolved by the gateway from
  the inbound payload via `IMPLEMENTED_BY.identifier_selector`
  (e.g. a SonarQube `/project/key`). For built-in `gateway-actions`
  actions that don't use it, the parameter is simply ignored.
- `action_config` is a **typed instance of the action's
  `config_model`**, already validated by the host. The action does
  not call `model_validate_json`; the host does.
- `payload` is the raw request body.

### Handler field format

- `WebhookRule.handler` becomes `<plugin_slug>#<action_name>`.
- The `#` separator is deliberate: pydantic's `ImportString` treats
  `module:attr` (or `module.attr`) as valid; `#` is invalid Python
  import syntax so there's no chance of an accidental fall-through to
  `ImportString` resolution.
- Validator: `^[a-z][a-z0-9-]*#[a-z][a-z0-9_]*$`. Slug rule mirrors
  the existing plugin-slug shape; action name allows underscores so
  Python identifiers stay legal.
- **Only the `#` form is supported at runtime.** The three legacy
  handlers (`update_project`, `create_release`,
  `add_deployment_event`, all previously dotted-path imports under
  `imbi_gateway.actions`) move into a new built-in `gateway-actions`
  plugin and existing rows are rewritten as a one-shot data
  migration. There is no dual-format fallback.

### imbi-api validation + UI surface

- `WebhookRuleCreate` / `WebhookUpdate` gain a model validator that:
  1. Parses `handler` into `plugin_slug` + `action_name` (rejects
     malformed strings with 422).
  2. Looks the plugin up in `imbi_common.plugins.registry` (already
     loaded at startup via `startup_load_plugins`). Unknown slug ⇒
     422.
  3. Resolves `entry.handler_cls.actions()` and finds the descriptor
     by name. Unknown action ⇒ 422.
  4. Resolves the descriptor's `config_model` (lazily, by triggering
     the `ImportString` validator on the descriptor) and calls
     `config_model.model_validate(handler_config)`. ValidationError
     ⇒ 422 with the underlying pydantic error surfaced.
- **New endpoint `GET /plugins/{slug}/actions`** returns:
  ```json
  [
    {
      "name": "update_project_from_webhook",
      "label": "Update Project from SonarQube Measures",
      "description": "Patches Imbi project facts using SonarQube measurements …",
      "config_schema": { … config_model.model_json_schema() … }
    }
  ]
  ```
  The endpoint is the UI's source of truth for rendering the rule
  editor; it imports the `config_model` and emits its JSON Schema
  verbatim. No additional per-field metadata is layered on top —
  pydantic `Field(description=..., default=...)` flows through
  naturally.

### Built-in `gateway-actions` plugin

- Ships inside `imbi-gateway` and registers via entry point so it's
  always present in the gateway's plugin registry.
- Manifest: `slug='gateway-actions'`, `plugin_type='webhook'`, no
  credentials. The plugin is invoked even when no `Plugin` node is
  attached to the TPS (host fills `credentials={}`).
- Three actions, each pointing at a config model:
  - `update_project` → `UpdateProjectConfig` (a
    `pydantic.RootModel[list[UpdateProjectRule]]` wrapping the
    existing per-row `UpdateProjectRule`).
  - `create_release` → existing `CreateReleaseConfig`.
  - `add_deployment_event` → existing `AddDeploymentEventConfig`.
- All three callables are refactored to the uniform signature; their
  current internal `model_validate_json` calls are dropped (the host
  validates now).
- Rule data migration: existing rows whose `handler` field looks like
  `imbi_gateway.actions:update_project` (etc.) are rewritten in
  place to `gateway-actions#update_project` before the gateway's
  next deploy. See **Implementation outline** §5.

### Smaller defaults (decided without asking)

- The host resolves `ImportString`s **lazily** — `ActionDescriptor`
  validates them at construction time (so misconfigured plugins fail
  loud at import), but the gateway / api call `descriptor.callable`
  /`descriptor.config_model` at dispatch / validation time rather
  than caching imported objects globally.
- SonarQube measure values are passed through verbatim into the JSON
  Patch (no client-side numeric coercion).
- Missing metric in the SonarQube response → log a warning and skip
  that mapping; other patches still ship.
- `imbi-plugin-sonarqube` mirrors `imbi-plugin-logzio` layout: `uv`,
  `httpx`, `respx` for tests, 90% coverage gate, single-quote ruff
  rules, BSD-3 license, Python ≥3.14.
- Single credential field `api_token` on the manifest.
- `prerelease` constraint for `clickhouse-connect` (transitive of
  `imbi-common[databases]`) lives in
  `[tool.uv] constraint-dependencies` so its dev-deps resolve.
- The gateway lists `imbi-plugin-sonarqube` in its base runtime
  dependencies so the plugin is importable in the gateway process.
- The same plugin packages must be installed in **both** imbi-api
  and imbi-gateway processes. imbi-api needs the config models to
  validate rule writes and serve `/plugins/{slug}/actions`;
  imbi-gateway needs both the callable and the config model to
  dispatch. Plugin packages should keep their action / config
  modules dependency-light so importing them in imbi-api is cheap;
  heavier subpackages (e.g. `client.py`) are imported only by the
  action body, on demand.

## Critical files

### imbi-common

- `src/imbi_common/plugins/base.py`
  - Add `class ActionDescriptor(pydantic.BaseModel)` with fields:
    `name: str`, `label: str`,
    `description: str | None = None`,
    `callable: pydantic.ImportString[…]`,
    `config_model: pydantic.ImportString[type[pydantic.BaseModel]]`.
  - Extend `PluginManifest.plugin_type` Literal with `'webhook'`.
  - Replace the existing draft `WebhookActionPlugin` (which has an
    abstract `run_action`) with a class that has only:
    ```python
    class WebhookActionPlugin(abc.ABC):
        manifest: PluginManifest

        @classmethod
        @abc.abstractmethod
        def actions(cls) -> list[ActionDescriptor]: ...
    ```
- `src/imbi_common/plugins/registry.py`
  - Continue recognising `'webhook'`. Existing subclass check stays.
  - Add a load-time guard: if a `WebhookActionPlugin` subclass
    returns an empty list from `actions()`, log a warning (plugin is
    inert).
  - Add a uniqueness guard on action `name` within a single plugin's
    `actions()`; duplicates → skip plugin with an error.
- `src/imbi_common/plugins/credentials.py` (new file) — moved from
  `imbi_api/plugins/credentials.py`; exposes
  `get_plugin_credentials`, `get_plugin_configuration_keys`, and
  `patch_plugin_configuration`. Pure graph + auth.encryption code,
  no fastapi imports.
- `src/imbi_common/plugins/__init__.py` — re-exports the new symbols
  (`ActionDescriptor`, `WebhookActionPlugin`,
  `get_plugin_credentials`).

### imbi-api

- `src/imbi_api/plugins/credentials.py` — re-export shim importing
  from `imbi_common.plugins.credentials`. Keeps every existing
  imbi-api callsite untouched.
- `src/imbi_api/domain/models.py`
  - `WebhookRuleCreate.handler` / `WebhookRuleResponse.handler` keep
    `str` typing.
  - Add a shared validator helper (called by both `WebhookRuleCreate`
    and `WebhookUpdate`'s `rules` model_validator) that:
    1. Asserts `handler` matches the `#` shape.
    2. Looks up the plugin in the registry.
    3. Resolves the action descriptor and validates
       `handler_config` against `descriptor.config_model`.
  - Errors are raised as `ValueError` so pydantic returns 422 with
    the underlying message; the endpoint layer does not need to
    inspect them.
- `src/imbi_api/endpoints/plugins.py` (existing file — exact path TBD
  during implementation; the new endpoint joins whatever module
  already serves `GET /plugins`)
  - **New endpoint** `GET /plugins/{slug}/actions` returning a list
    of `{name, label, description, config_schema}` objects.
    `config_schema` is `descriptor.config_model.model_json_schema()`.
  - Permission gate: same as the existing plugin-list endpoint
    (authenticated read).
- `src/imbi_api/endpoints/webhooks.py`
  - No Cypher changes (handler is still a stored string), but the
    response-side mapping continues to round-trip `handler` /
    `handler_config` unchanged.

### imbi-gateway

- `src/imbi_gateway/actions.py`
  - The three handlers stay in this module but lose their internal
    `*Config.model_validate_json(...)` calls and switch to the
    uniform signature:
    ```python
    async def update_project(
        *,
        ctx: PluginContext,
        credentials: dict[str, str],
        external_identifier: str,
        action_config: UpdateProjectConfig,
        payload: object,
    ) -> None: ...
    ```
  - `UpdateProjectRules = pydantic.TypeAdapter(...)` becomes
    `class UpdateProjectConfig(pydantic.RootModel[list[UpdateProjectRule]])`
    so the host can resolve it via `ImportString[type[BaseModel]]`.
  - `CreateReleaseConfig` and `AddDeploymentEventConfig` already are
    `BaseModel` subclasses and need no shape change.
- `src/imbi_gateway/plugin.py` (new)
  - `class GatewayActionsPlugin(WebhookActionPlugin)` with
    `manifest.slug='gateway-actions'`, no credentials.
  - `actions()` returns three `ActionDescriptor`s pointing at
    `imbi_gateway.actions:update_project` (etc.) and the matching
    config models.
- `src/imbi_gateway/notifications.py`
  - Drop the `pydantic.ImportString[ActionFunction]` typing on
    `WebhookRule.handler`. Make it a `str` with a field validator
    that splits on `#`, exposing `plugin_slug` and `action_name`
    properties. Validator rejects malformed strings outright (the
    rule was already invalid before reaching here).
  - The Cypher walk for plugin info changes from
    `USES_APPLICATION → ServiceApplication.encrypted_credentials`
    to `HAS_PLUGIN → Plugin` collecting
    `{plugin_id, plugin_slug, plugin_configuration}` per attached
    plugin so the dispatcher can match the rule's `plugin_slug` to
    a stored credential blob.
  - The project-resolution Cypher gains
    `OPTIONAL MATCH (p)-[:OWNED_BY]->(t:Team)` and returns
    `project_slug` and `team_slug` so the dispatcher can populate
    `PluginContext` properly.
  - New helpers replace `_run_handlers`:
    - `_resolve_action(rule)` → `(handler_cls, descriptor)` (uses
      `registry.get_plugin(rule.plugin_slug)` and looks up the
      descriptor by name).
    - `_resolve_credentials(handler_cls, plugins_by_slug)` returns
      `{}` for plugins with no manifest credentials; otherwise
      pulls the decrypted blob from the matching `Plugin` node, or
      logs a warning and skips dispatch when none is attached.
    - `_invoke_action(descriptor, ctx, credentials,
      external_identifier, handler_config_raw, payload)` validates
      the config and calls the action. Catches and logs handler
      exceptions per the existing dispatcher contract.
- `src/imbi_gateway/app.py` — calls
  `plugin_registry.load_plugins()` during `create_app` so the
  registry is populated before any request lands. (Same pattern as
  imbi-api's `startup_load_plugins`.)
- `pyproject.toml`
  - Register `gateway-actions` under
    `[project.entry-points."imbi.plugins"]`.
  - Add `imbi-plugin-sonarqube>=0.1.0` as a runtime dependency.
- Tests — full rewrite (see **Tests**).

### imbi-plugin-sonarqube

- `pyproject.toml`
  - `name='imbi-plugin-sonarqube'`, Python ≥3.14, BSD-3.
  - Deps: `httpx`, `imbi-common~=2.4`, `jsonpointer`, `pydantic`.
  - Dev deps: `imbi-common[server,databases]~=2.4`, `basedpyright`,
    `coverage`, `pre-commit`, `pytest`, `pytest-asyncio`, `respx`,
    `ruff`.
  - `[project.entry-points."imbi.plugins"] sonarqube =
    "imbi_plugin_sonarqube.plugin:SonarqubePlugin"`.
  - `[tool.uv] constraint-dependencies = ["clickhouse-connect>=0.12rc1"]`
    so the dev resolver accepts the prerelease.
- `src/imbi_plugin_sonarqube/plugin.py`
  - `SonarqubePlugin(WebhookActionPlugin)` with manifest
    (`slug='sonarqube'`, `plugin_type='webhook'`, single
    `api_token` credential).
  - `run_action` is **removed**.
  - `actions()` returns a single `ActionDescriptor`:
    ```python
    @classmethod
    def actions(cls) -> list[ActionDescriptor]:
        return [
            ActionDescriptor(
                name='update_project_from_webhook',
                label='Update Project from SonarQube Measures',
                description=(
                    "Fetches the configured SonarQube measurements "
                    "and patches the matched Imbi project's facts."
                ),
                callable=(
                    'imbi_plugin_sonarqube.actions'
                    ':update_project_from_webhook'
                ),
                config_model=(
                    'imbi_plugin_sonarqube.actions:MetricMappings'
                ),
            )
        ]
    ```
- `src/imbi_plugin_sonarqube/actions.py`
  - `MetricMappings` becomes a `pydantic.RootModel[list[MetricMapping]]`
    (replaces the current `pydantic.TypeAdapter`).
  - `update_project_from_webhook` is rewritten to the uniform
    signature; `action_config` arrives as a `MetricMappings`
    instance (not a JSON string), so the function iterates
    `action_config.root` directly.
  - Behaviour otherwise unchanged: read `service_endpoint` from
    `ctx.assignment_options`, fetch from SonarQube, build a JSON
    Patch, PATCH imbi-api via httpx.
- `src/imbi_plugin_sonarqube/client.py` — thin httpx wrapper around
  `GET {base_url}/api/measures/component` with Bearer auth and
  structured error handling (`SonarqubeClientError`). Unchanged
  from the previous plan.

## Implementation outline (ordered)

1. **imbi-common**
   - Add `ActionDescriptor` and the new `WebhookActionPlugin`
     (classmethod `actions`, no `run_action`).
   - Move `credentials.py` into `imbi_common.plugins`.
   - Registry: add the action-name uniqueness + empty-actions guards.
   - Cut a new release (≥ 2.4.1) before downstream packages can
     resolve their deps.
2. **imbi-api**
   - Replace `imbi_api/plugins/credentials.py` with the re-export
     shim.
   - Add the `WebhookRuleCreate` / `WebhookUpdate` validator that
     parses `handler` and validates `handler_config` against the
     resolved `config_model`.
   - Add `GET /plugins/{slug}/actions`.
   - Keep all existing tests green; add coverage for the new endpoint
     and validator.
3. **imbi-gateway**
   - Convert `UpdateProjectRules` (TypeAdapter) into
     `UpdateProjectConfig` (RootModel).
   - Refactor the three actions in `actions.py` to the uniform
     signature; drop internal `model_validate_json` calls.
   - Add `GatewayActionsPlugin` in `imbi_gateway/plugin.py` and
     register it under `[project.entry-points."imbi.plugins"]`.
   - Rewrite the notification dispatcher (`_run_handlers` → action
     resolver + invoker), update the Cypher walks for plugin info
     and project resolution, and wire
     `plugin_registry.load_plugins()` into `create_app`.
   - Add `imbi-plugin-sonarqube>=0.1.0` to runtime deps.
4. **imbi-plugin-sonarqube**
   - Convert `MetricMappings` to a `RootModel`.
   - Remove `SonarqubePlugin.run_action`; add `actions()`.
   - Rewrite `update_project_from_webhook` to the uniform signature.
   - Land tests + 90% coverage.
5. **Rule data migration**
   - One-shot script (or admin command) rewrites stored
     `WebhookRule.handler` values:
     `imbi_gateway.actions:update_project` →
     `gateway-actions#update_project`, and the two siblings. Runs
     once per environment before the new gateway deploys.
   - Out of scope for the gateway-side code change; the meta-repo's
     deployment-webhook plan owns its row updates.
6. **Meta-repo** — register `imbi-plugin-sonarqube` as a submodule
   after the standalone repo exists. (Out of scope here; the user
   owns the submodule wiring.)

## Tests

### imbi-common

- `tests/test_plugins/test_base.py`
  - Instantiating a `WebhookActionPlugin` subclass that does not
    override `actions` raises `TypeError`.
  - `ActionDescriptor` resolves valid `callable` / `config_model`
    ImportStrings at construction, and raises `ValidationError` on
    bad targets (non-callable, non-BaseModel).
- `tests/test_plugins/test_registry.py`
  - A `webhook` plugin with one valid descriptor loads round-trip;
    duplicate action names within one plugin are rejected;
    `actions()` returning `[]` logs a warning and still loads (the
    plugin is just inert).
  - A `plugin_type='webhook'` class that does NOT subclass
    `WebhookActionPlugin` is rejected.

### imbi-api

- `tests/endpoints/test_plugins_actions.py` (new)
  - `GET /plugins/{slug}/actions` returns the descriptor list with
    each item's `config_schema` deep-equal to the model's
    `model_json_schema()`.
  - Unknown slug → 404; unauthenticated → 401.
- `tests/endpoints/test_webhooks.py`
  - `WebhookRuleCreate` with valid `handler` + matching
    `handler_config` accepts (201).
  - Malformed `handler` (missing `#`, empty slug, empty action) →
    422 with a clear message.
  - Unknown plugin slug → 422.
  - Unknown action name on a known plugin → 422.
  - `handler_config` that does not validate against the
    `config_model` → 422 surfacing the pydantic error.
  - Round-trip: a created rule reads back unchanged via
    `GET /webhooks/{id}`.

### imbi-gateway

- `tests/test_actions.py`
  - Each of the three legacy actions runs against a constructed
    `PluginContext` and a pre-validated config-model instance;
    asserts the expected PATCH / POST against `ImbiClient`.
  - `update_project` integrates with `UpdateProjectConfig`
    (RootModel) and produces the correct JSON-Patch list.
- `tests/test_notifications.py`
  - `WebhookRuleUnitTests`: validator rejects missing `#`, empty
    slug, empty action; the new `plugin_slug` / `action_name`
    properties match the stored string.
  - `ProcessNotificationTests` (uses `plugin_registry._registry`
    stubs):
    - Successful dispatch validates the config, builds a
      `PluginContext` carrying `org_slug` / `project_id` /
      `project_slug` / `team_slug` / `actor_user_id`, and invokes
      the callable.
    - Multi-project fan-out yields one invocation per matched
      project.
    - Plugins with declared credentials are skipped (warning) when
      no `Plugin` node is attached.
    - Plugins with declared credentials receive the decrypted blob
      stored at `Plugin.plugin_configuration`.
    - Plugins with empty `manifest.credentials` run with `{}`
      regardless of attachment state.
    - Unknown `plugin_slug` logs an error and skips dispatch
      without raising.
    - Unknown `action_name` on a known plugin logs an error and
      skips dispatch.
    - Malformed `handler_config` against the `config_model` logs an
      error and skips dispatch (the gateway must not 5xx — the rule
      should have been rejected at write time, but we defend against
      drift).

### imbi-plugin-sonarqube

- `tests/test_plugin.py`
  - Manifest shape (slug, type, credentials list).
  - `SonarqubePlugin.actions()` returns one descriptor whose
    `callable` resolves to `update_project_from_webhook` and whose
    `config_model` resolves to `MetricMappings`.
- `tests/test_client.py` — unchanged from the previous plan: happy
  path, non-2xx → `SonarqubeClientError`, transport error,
  non-JSON, URL contains `?component=` and `metricKeys=`.
- `tests/test_actions.py`
  - Happy path: 2 mappings, 2 measures returned → 2-element patch.
  - Missing metric in response → warning, partial patch.
  - Missing `api_token` in `credentials` → handler returns without
    calling SonarQube.
  - Missing `service_endpoint` on `ctx.assignment_options` →
    handler returns without calling SonarQube.
  - SonarQube returns 5xx → error logged, no PATCH issued.
  - imbi-api 5xx → warning logged on patch failure.
  - (No "malformed action_config" test — by contract the host
    validates before calling the action.)

## Verification

1. **`imbi-common`** — `just lint` and `just test` clean; coverage
   ≥ 90%. New tests cover `ActionDescriptor`, the
   `actions()`-classmethod abstract enforcement, and the registry's
   uniqueness/empty-list guards.
2. **`imbi-api`** — `just lint` + `just test` clean; coverage
   stays ≥ existing baseline. `GET /plugins/{slug}/actions` returns
   the SonarQube descriptor with a non-empty `config_schema`; create
   / update of a webhook rule rejects bad handlers / configs with
   422.
3. **`imbi-gateway`** — `just lint` + `just test` clean; coverage
   stays ≥ 90%. The AGE-dependent `ProcessNotificationTests` run
   via `just test` (which spins up Docker).
4. **`imbi-plugin-sonarqube`** — `just lint` + `just test` clean,
   coverage ≥ 90%, `uv build` produces a clean wheel.
5. **End-to-end smoke test** (manual against a local dev stack):
   - Bring up `imbi-api` and `imbi-gateway` with both plugin
     packages installed.
   - Seed an org, a SonarQube TPS (`api_endpoint=...`), attach a
     `sonarqube` plugin instance, and
     `PATCH .../configuration {"api_token": "..."}`.
   - Hit `GET /plugins/sonarqube/actions` and confirm the response
     includes `update_project_from_webhook` with a `config_schema`
     describing the metric / path array.
   - Create a webhook with `identifier_selector=/project/key` and a
     rule with `handler="sonarqube#update_project_from_webhook"`
     and the JSON config shown in **Context**. Confirm the rule
     accepts cleanly; try a deliberately bad config (e.g.
     `[{"metric": "coverage"}]` missing `path`) and confirm a 422.
   - `curl -f -X POST http://localhost:<gw>/notifications/<wid>`
     with a fixture SonarQube payload that has
     `/branch/is_main == true` and a known component key.
   - Confirm the gateway hits SonarQube (mocked or real) and that
     the matched project's `GET` now shows the updated
     `test_coverage` and `lines_of_code` facts.
