# GitHub Deployment Webhook → Imbi Releases

## Context

The `imbi-gateway` already accepts inbound GitHub webhooks and dispatches them through `WebhookRule` handlers per `tasks/deployment-webhook.md`. The two stubs that drive deployment processing — `actions.create_release` and `actions.add_deployment_event` (`imbi-gateway/src/imbi_gateway/actions.py:120,128`) — are empty. We need them to:

1. **`create_release`** — handle GitHub `deployment` events: ensure an Imbi `Release` exists for `/deployment/ref` via `POST /organizations/{org}/projects/{id}/releases/`.
2. **`add_deployment_event`** — handle GitHub `deployment_status` events: append a deployment edge event via `POST /organizations/{org}/projects/{id}/releases/{version}/environments/{env_slug}`, with `/deployment/url` as the note.

The user also wants the Imbi user attribution to be solved generically rather than ad-hoc per handler: the `Webhook → ThirdPartyService` `IMPLEMENTED_BY` edge already carries `identifier_selector` (a JSON Pointer that locates the project identifier in a payload). We mirror that pattern with two new edge properties — `user_subject_selector` (JSON Pointer to the external subject in the payload) and an optional `identity_plugin_slug` override — and resolve the Imbi user inside `process_notification`. Handlers receive the resulting `user_id` as a new `str | None` parameter and remain unaware of identity-plugin mechanics.

## Decisions (user-confirmed)

- Env slug is taken **verbatim** from `/deployment/environment`. No mapping table.
- New endpoint shape: **`GET /users/by-identity?plugin_slug=&subject=`** on `users_router`. Returns `UserResponse`, 404 when no match.
- GitHub `deployment_status.state` mapping: `queued`/`pending` → `pending`, `in_progress` → `in_progress`, `success` → `success`, `failure`/`error` → `failed`, `inactive` → `rolled_back`. Unknown states logged + skipped (no event recorded).
- Plugin slug for user lookup: **edge override + graph fallback, both filtered to identity-type plugins**. If the IMPLEMENTED_BY edge carries `identity_plugin_slug`, use it. Otherwise iterate every identity plugin attached to the ThirdPartyService via `(tps)-[:HAS_PLUGIN]->(p:Plugin)` (filter to `plugin_type='identity'`), call the lookup once per slug, collect the distinct user ids returned. 0 → `user_id=None`; 1 → that id; **≥2 distinct users → log error and pass `user_id=None`** (handlers still run).
- `ActionFunction` gains a new `user_id: str | None` parameter; `update_project` accepts and ignores it for backwards-compatibility.

## Critical files

### imbi-api
- `src/imbi_api/identity/repository.py:281` — `find_user_by_subject` already exists; reuse it as the new endpoint's only graph call.
- `src/imbi_api/endpoints/users.py` — add `get_user_by_identity` route.
- `src/imbi_api/domain/models.py` — extend `WebhookCreate`/`WebhookUpdate`/`WebhookResponse` (and `WebhookResponse.from_graph_record`) with `user_subject_selector: str | None = None` and `identity_plugin_slug: str | None = None`.
- `src/imbi_api/endpoints/webhooks.py` — write/read the two new edge properties:
  - `_FETCH_WEBHOOK_QUERY` and `list_webhooks` query: `RETURN ... impl.user_subject_selector AS user_subject_selector, impl.identity_plugin_slug AS identity_plugin_slug`.
  - `create_webhook` Cypher (line 264): include both new properties on the `IMPLEMENTED_BY` edge create.
  - `patch_webhook` patchable dict (line 442) and write Cypher (line 549): include both.
  - `_UPDATE_RETURN_TAIL_WITH_TPS` / `_UPDATE_RETURN_TAIL_NO_TPS`: add new columns.

### imbi-gateway
- `src/imbi_gateway/notifications.py`:
  - `ActionFunction` typedef (line 17): `Callable[[str, str, Any, str | None, str], Awaitable[None]]` (4th = `user_id`, 5th = `handler_config`).
  - Webhook resolution Cypher (line 67): also `RETURN impl.user_subject_selector`, `impl.identity_plugin_slug`, and `[(tps)-[:HAS_PLUGIN]->(p:Plugin) WHERE p.plugin_type='identity' | p.plugin_slug] AS identity_plugin_slugs`.
  - New helper `_resolve_user_id(body, sel, identity_slugs, http_client) -> str | None` that implements the override + fallback + multi-resolution logic above. Calls `GET /users/by-identity?plugin_slug=&subject=` (404 → `None`).
  - `_run_handlers`: forward `user_id` as the 4th positional argument.
- `src/imbi_gateway/actions.py`:
  - Update `update_project` signature to `(org_slug, project_id, body, user_id, update_spec)` — `user_id` ignored.
  - Implement `create_release` and `add_deployment_event` (matching the new 5-arg signature).

### imbi-ui
- Add `user_subject_selector` and `identity_plugin_slug` text fields to the webhook create/edit form. The form already renders `identifier_selector` next to the third-party-service picker — drop the new fields in alongside it. Surface the same fields on the webhook detail view.

### imbi-plugin-github
- No code changes. The plugin's existing `identity` manifest is what populates `Plugin.plugin_type='identity'` so the gateway's identity-plugin filter picks it up. Operator action only: ensure it is registered + assigned to the relevant ThirdPartyService(s).

## Implementation outline

### 1. imbi-api: `GET /users/by-identity`

In `src/imbi_api/endpoints/users.py`, add:

```python
@users_router.get('/by-identity', response_model=models.UserResponse)
async def get_user_by_identity(
    plugin_slug: str,
    subject: str,
    db: graph.Pool,
    auth: typing.Annotated[
        permissions.AuthContext,
        fastapi.Depends(permissions.require_permission('user:read')),
    ],
) -> models.UserResponse:
    user_id = await identity_repository.find_user_by_subject(
        db, plugin_slug=plugin_slug, subject=subject,
    )
    if user_id is None:
        raise fastapi.HTTPException(status_code=404, detail=...)
    # Reuse the existing User->UserResponse projection used by get_user;
    # consider extracting that into a small helper if duplication grows.
    nodes = await db.match(models.User, {'id': user_id})
    ...
```

Route ordering: `/by-identity` must be declared before the catch-all `/{email}` route to avoid `email='by-identity'` capture.

### 2. imbi-api: webhook edge properties

`WebhookCreate`/`WebhookUpdate`/`WebhookResponse` gain optional `user_subject_selector` and `identity_plugin_slug`. The edge-create Cypher in `create_webhook` (line 267) currently writes one property:

```
CREATE (w)-[:IMPLEMENTED_BY {{identifier_selector: {identifier_selector}}}]->(tps)
```

Extend to:

```
CREATE (w)-[:IMPLEMENTED_BY {{
  identifier_selector: {identifier_selector},
  user_subject_selector: {user_subject_selector},
  identity_plugin_slug: {identity_plugin_slug}
}}]->(tps)
```

Same pattern in `patch_webhook` (line 549). All three FETCH/LIST queries return the new edge columns. `from_graph_record` parses them.

### 3. imbi-gateway: notification dispatch

The handler signature expansion is mechanical:

```python
ActionFunction = typing.Callable[
    [str, str, typing.Any, str | None, str], abc.Awaitable[None]
]
```

`_run_handlers` becomes `await rule.handler(org_slug, project_id, body, user_id, rule.handler_config)`.

`process_notification` extends its query (line 67 onward) to surface `impl.user_subject_selector`, `impl.identity_plugin_slug`, and a list-comprehended `identity_plugin_slugs`. After the existing project-resolution block, before `_run_handlers`:

```python
user_id = await _resolve_user_id(
    body=body,
    user_subject_selector=sel.get('user_subject_selector'),
    edge_plugin_slug=sel.get('identity_plugin_slug'),
    candidate_plugin_slugs=identity_plugin_slugs,
)
```

`_resolve_user_id`:
1. Return `None` if `user_subject_selector` is unset OR resolves to a falsy/missing value.
2. Build the candidate slug list: `[edge_plugin_slug] if edge_plugin_slug else candidate_plugin_slugs`.
3. For each slug, call `GET /users/by-identity?plugin_slug={slug}&subject={subject}` via a shared `ImbiClient` session (extend the existing client class — add a new `get_user_by_identity` method that 404s → `None`). Collect non-None ids into a `set`.
4. Length 0 → `None`; length 1 → that id; ≥2 → `LOGGER.error(...)` and `None`.

### 4. imbi-gateway: `actions.create_release`

Pydantic input model (subset of GitHub deployment payload):

```python
class _DeploymentEvent(pydantic.BaseModel):
    class Deployment(pydantic.BaseModel):
        ref: str
        url: pydantic.HttpUrl
        environment: str
    deployment: Deployment
```

Body:

```python
async def create_release(
    org_slug: str, project_id: str, body: object,
    user_id: str | None, handler_config: str,
) -> None:
    payload = _DeploymentEvent.model_validate(body)
    create_body = {
        'version': payload.deployment.ref,
        'title': payload.deployment.ref,  # or derive from handler_config
        'links': [{'name': 'GitHub Deployment',
                   'href': str(payload.deployment.url)}],
    }
    if user_id is not None:
        create_body['created_by'] = user_id
    async with ImbiClient() as client:
        resp = await client.post(
            f'/organizations/{org_slug}/projects/{project_id}/releases/',
            json=create_body,
        )
        if resp.status_code == 409:
            return  # idempotent — already created
        resp.raise_for_status()
```

Note: passing `user_id` (a nanoid) into a string field that is conventionally an email is an acceptable transitional hack — it preserves the user attribution end-to-end, and the API response surfaces the value verbatim. A follow-up can teach `ReleaseCreate` to resolve a `created_by_user_id` server-side.

### 5. imbi-gateway: `actions.add_deployment_event`

```python
class _DeploymentStatusEvent(pydantic.BaseModel):
    class Deployment(pydantic.BaseModel):
        ref: str
        url: pydantic.HttpUrl
        environment: str
    class DeploymentStatus(pydantic.BaseModel):
        state: str
    deployment: Deployment
    deployment_status: DeploymentStatus

_STATUS_MAP = {
    'queued': 'pending', 'pending': 'pending',
    'in_progress': 'in_progress',
    'success': 'success',
    'failure': 'failed', 'error': 'failed',
    'inactive': 'rolled_back',
}
```

Body:

```python
async def add_deployment_event(
    org_slug: str, project_id: str, body: object,
    user_id: str | None, handler_config: str,
) -> None:
    payload = _DeploymentStatusEvent.model_validate(body)
    status = _STATUS_MAP.get(payload.deployment_status.state)
    if status is None:
        LOGGER.warning(
            'Unmapped deployment_status.state %r — skipping',
            payload.deployment_status.state,
        )
        return
    url = (
        f'/organizations/{org_slug}/projects/{project_id}'
        f'/releases/{payload.deployment.ref}'
        f'/environments/{payload.deployment.environment}'
    )
    async with ImbiClient() as client:
        resp = await client.post(url, json={
            'status': status,
            'note': str(payload.deployment.url),
        })
        if resp.status_code == 404:
            LOGGER.warning(
                'Release %r missing for project %s; status %r dropped',
                payload.deployment.ref, project_id, status,
            )
            return
        resp.raise_for_status()
```

Note `user_id` is unused on the deployment event — `record_deployment` has no `created_by` field. Listed in the signature only so the dispatcher signature is uniform.

### 6. imbi-gateway: `ImbiClient`

Add two methods alongside `patch_project` (line 84):

- `get_user_by_identity(plugin_slug, subject) -> dict | None` — GET, 404 → None.
- `post_release(org_slug, project_id, body) -> Response` — POST, surfaces 201 / 409 / errors.
- `post_deployment_event(org_slug, project_id, version, env_slug, body) -> Response` — POST.

These keep URL formatting and logging in one place and make the handler bodies short.

## Tests

- **imbi-api**:
  - `tests/endpoints/test_users.py` (or equivalent): `/users/by-identity` returns the user; 404 when subject absent; 404 when slug absent; permission enforced.
  - `tests/endpoints/test_webhooks.py`: round-trip create/patch/get with non-null `user_subject_selector` and `identity_plugin_slug`.
- **imbi-gateway**:
  - `tests/test_notifications.py`: extend the existing fixture to include the new edge fields; tests for (a) selector unset → `user_id=None`, (b) edge override slug used directly, (c) graph fallback resolves to a single plugin, (d) graph fallback finds the same user from two plugins → still passes that user, (e) graph fallback finds two different users → logs error and passes `None`.
  - `tests/test_actions.py`: `create_release` and `add_deployment_event` happy paths, 409 idempotency on release create, status mapping table, unknown-state skip, missing release on event → warning, no exception.

## Verification

1. `just lint` and `just test` clean in both `imbi-api` and `imbi-gateway`. Coverage gates remain ≥90% in `imbi-gateway`.
2. End-to-end against a local dev stack:
   - `just serve` `imbi-api`; create an org, third-party service, attach the `github` plugin, create a webhook with `identifier_selector=/repository/full_name`, `user_subject_selector=/deployment/creator/id`, `identity_plugin_slug=github`, and rules pointing at the two new handlers.
   - `just serve` `imbi-gateway` with `ACTIONS_IMBI_TOKEN` pointing at the API.
   - `curl -f -X POST http://localhost:<gw>/notifications/<webhook_id>` with a fixture GitHub `deployment` payload → 202; verify `GET /api/organizations/{org}/projects/{id}/releases/` shows the new release.
   - Repeat with `deployment_status` payloads (`in_progress`, then `success`); verify `GET .../releases/{version}/environments/{env}` shows two events with `current_status=success` and the GitHub deployment URL in the latest event's note.
3. UI smoke: open the webhook edit screen, set/clear the two new fields, save, confirm round-trip.
