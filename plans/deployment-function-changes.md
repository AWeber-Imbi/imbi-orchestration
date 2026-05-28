# Decouple deployment action handlers from GitHub specifics

## Context

`imbi_gateway.actions.create_release` and `actions.add_deployment_event`
currently read GitHub-shaped fields directly out of the webhook body
(via `_DeploymentEvent` / `_DeploymentStatusEvent` Pydantic models) and
ignore their `handler_config` argument. This bakes the GitHub payload
shape into the gateway and prevents the same handlers from servicing
non-GitHub webhooks.

The feature
(`features/deployment-function-changes.md`) moves all payload-shape
knowledge into the per-rule `handler_config` JSON: JSONPointers select
fields from the body, and a CEL expression computes the release
version. The handlers themselves become payload-agnostic.

## Decisions

User-confirmed during planning:

- **Release `links` array**: dropped entirely. The current
  `github_deployment` link goes away; if links are needed later they
  can be re-introduced as configuration.
- **Deployment-event `note`**: configuration-driven via an optional
  `note_selector` JSONPointer. Omitted from the API call when not
  configured.
- **Deployment status**: `status_selector` JSONPointer reads the raw
  state from the body; the existing `_STATUS_MAP` (GitHub state →
  Imbi status) is retained as a static mapping. This keeps the
  current behavior (unknown states log + skip) while removing the
  hard-coded `payload.deployment_status.state` access.
- **Config validation**: Pydantic models, following the
  `UpdateProjectRules` (TypeAdapter) pattern in `update_project`.
  `pydantic.ValidationError` is allowed to propagate — FastAPI error
  handling will catch it at the call site later.
- **CEL compile errors**: not caught inside the handler — they
  propagate to `_run_handlers`, which already wraps every handler
  call in `try/except Exception` and logs.

## Critical files

### `imbi-gateway/src/imbi_gateway/actions.py`

- Add two Pydantic models for the new handler configs:
  - `CreateReleaseConfig` — `title_selector: JsonPointer`,
    `version_expression: str`
  - `AddDeploymentEventConfig` — `environment_selector: JsonPointer`,
    `version_expression: str`, `status_selector: JsonPointer`,
    `note_selector: JsonPointer | None = None`
- Add a small helper to compile + evaluate a CEL expression against
  a body and return the string result. Mirrors the
  `WebhookRule.evaluate_condition` pattern in `notifications.py:38-56`
  but does not swallow errors.
- Rewrite `create_release` (currently lines 235-275):
  - Validate `handler_config` via `CreateReleaseConfig.model_validate_json`.
  - Resolve `title` via `config.title_selector.resolve(body)`.
  - Compute `version` via the CEL `version_expression`.
  - Build `create_body` with only `version`, `title`, and (when
    present) `created_by` — no `links`.
- Rewrite `add_deployment_event` (currently lines 278-316):
  - Validate `handler_config` via
    `AddDeploymentEventConfig.model_validate_json`.
  - Resolve `state` via `config.status_selector.resolve(body)` and
    look it up in `_STATUS_MAP`; unknown → log + skip (preserve
    current behavior).
  - Resolve `environment` via `config.environment_selector.resolve(body)`.
  - Compute `version` via the CEL `version_expression`.
  - Build the deployment body with `status` plus `note` (only when
    `note_selector` is configured and resolves).
- Delete the now-unused payload models: `_DeploymentCreator`,
  `_Deployment`, `_DeploymentEvent`, `_DeploymentStatusEvent`.
- `_STATUS_MAP` stays.

### `imbi-gateway/tests/test_actions.py`

- `_DEPLOYMENT_BODY` and `_STATUS_BODY` keep their GitHub-ish shape
  (since CEL/pointer expressions in the test configs read those
  fields). Update them only if needed for clarity.
- Add module-level config constants:
  ```python
  _CREATE_RELEASE_CONFIG = (
      '{"title_selector": "/deployment/ref",'
      ' "version_expression": "deployment.ref"}'
  )
  _DEPLOYMENT_EVENT_CONFIG = (
      '{"environment_selector": "/deployment/environment",'
      ' "version_expression": "deployment.ref",'
      ' "status_selector": "/deployment_status/state"}'
  )
  ```
- Update `CreateReleaseTests`:
  - Pass `_CREATE_RELEASE_CONFIG` instead of `'{}'`.
  - Drop the `links` assertion in
    `test_happy_path_includes_user_id_and_link` and rename it
    (e.g. `test_happy_path_includes_user_id`).
  - Add `test_title_selector_used` — asserts the resolved title
    differs from the version when `title_selector` points at a
    different field (e.g. `/deployment/description`).
  - Add `test_version_expression_evaluated` — confirms a CEL
    expression more complex than `deployment.ref` (e.g. the
    `matches`/substring example from the feature file) produces the
    expected version string.
  - Add `test_invalid_handler_config_raises_validation_error` and
    `test_invalid_version_expression_propagates` (CEL compile/eval
    error escapes the handler).
- Update `AddDeploymentEventTests`:
  - Pass `_DEPLOYMENT_EVENT_CONFIG` instead of `'{}'`.
  - `test_status_mapping_and_note` becomes `test_status_mapping`
    (no `note` is sent by default). Update the expected
    `record_deployment` call to omit `note`.
  - Add `test_note_selector_emits_note` — config includes
    `note_selector` resolving to the GitHub URL string; expected
    body now contains `note`.
  - Update `test_failure_state_maps_to_failed`,
    `test_unknown_state_skipped`, `test_release_missing_logs_warning`
    to use the new config constant.
  - Add `test_invalid_handler_config_raises_validation_error`.
- Update `StatusMapTests._capture_status` to pass
  `_DEPLOYMENT_EVENT_CONFIG`.
- No changes required in `tests/test_notifications.py` — the
  `stub_handler` signature is unchanged, and tests there pass
  `'{}'` as the config because they exercise dispatch, not the
  built-in handlers.

## Implementation outline

1. Add the two Pydantic config models and the CEL-evaluation helper
   in `actions.py`.
2. Refactor `create_release` to use `CreateReleaseConfig`.
3. Refactor `add_deployment_event` to use `AddDeploymentEventConfig`.
4. Delete the unused `_Deployment*` payload models and (if no other
   uses remain) the `pydantic.HttpUrl` import.
5. Update `tests/test_actions.py` per the list above.
6. Run `just lint` and `just test` in `imbi-gateway/`.

## Tests

Adjust existing tests to pass real configs; add coverage for:

- **`create_release`**: title selector, version CEL expression,
  invalid handler config (Pydantic `ValidationError`), invalid CEL
  expression (propagates).
- **`add_deployment_event`**: status selector mapping, optional
  `note_selector` (present and absent), invalid handler config,
  unmapped status still skips.
- **`StatusMapTests`**: continues to verify all GitHub state →
  Imbi status mappings via the `status_selector` path.

Coverage target: 90% (project requirement). All the new lines in
`actions.py` are exercised by the tests above.

## Verification

```bash
cd imbi-gateway
just lint                         # ruff + basedpyright + mypy
just test                         # full suite incl. coverage
```

Spot-check the configs match the feature file's examples by
copy-pasting them into a quick `python -c` invocation:

```bash
cd imbi-gateway
.venv/bin/python -c "
import json
from imbi_gateway import actions
cfg = actions.CreateReleaseConfig.model_validate_json(json.dumps({
    'title_selector': '/deployment/description',
    'version_expression': (
        'deployment.ref.matches(\"^[0-9]+\\\\.[0-9]+\\\\.[0-9]+\")'
        ' ? deployment.ref : deployment.sha.substring(0, 7)'
    ),
}))
print(cfg)
"
```

The end-to-end webhook path (`tests/test_notifications.py`
`ProcessNotificationTests`) already exercises rule dispatch, so the
existing integration tests catch regressions in how `handler_config`
flows from the graph row into `WebhookRule.handler_config` and on
into the action handler.
