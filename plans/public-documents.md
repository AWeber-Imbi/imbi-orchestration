# Public non-project documents (`public-documents`)

Feature: `imbi-orchestration/features/public-documents.md`

## Context

Every document GET in imbi-api requires a bearer token plus the
`document:read` permission. **ProjectType-attached** documentation
(plus the org-wide index filtered down to it) should be readable
without authentication by default, with a per-document **private**
flag to exclude individual documents and an org-level kill-switch
that disables anonymous access for the whole org.
Project-attached and **User-attached** documents must never be
exposed anonymously — the org-wide index and the attachment-agnostic
single-document GET currently resolve documents regardless of
attachment, so they need explicit filtering. There is no optional-auth path anywhere in the
auth layer today; this feature introduces one.

## Decisions (user-confirmed)

1. **Boolean + org kill-switch**: per-document `is_private: bool`
   (default `false`) plus `Organization.public_documents_enabled:
   bool` (default `true`). Effective anonymous visibility =
   `org.public_documents_enabled && !doc.is_private`.
2. **User-attached documents are NOT public** — only
   ProjectType-attached documents are anonymously readable. The
   user-documents routes keep `require_permission('document:read')`
   unchanged.
3. **API-only anonymous access**: no anonymous UI routes. UI scope is
   the private toggle, a Private badge, and the org admin switch.
4. **Redact emails for anonymous callers**: `created_by` /
   `updated_by` are nulled in anonymous responses;
   `created_by_name` (display name) is kept. Authenticated responses
   unchanged.

Stated defaults (not asked; flag at review if wrong):

- Anonymous requests get a 404 (not 403) for private/project-attached
  docs; an org with the kill-switch off yields empty lists / 404s
  in-query rather than a 401 — no existence leak, no extra query.
- An **invalid/expired token still 401s** — it never degrades to
  anonymous.
- An **authenticated principal lacking `document:read`** gets the
  public view (filtered, 200) on these GETs rather than 403 — a doc
  readable by the anonymous world cannot be forbidden to a logged-in
  user. Email redaction applies only when truly anonymous.
- Writes (POST/PATCH/DELETE) and all project-document routes keep
  their existing auth exactly as-is.
- No new permission strings are needed (`auth/seed.py` untouched).
- No data migration: Cypher filters use `IS NULL OR` so vertices
  written before the new properties existed behave as
  public / enabled, matching the pydantic defaults.
- Comments on public documents stay auth-only; uploads embedded in
  public docs are still served auth-only (anonymous viewers may see
  broken images) — recorded as follow-ups, not in scope.
- No rate-limit decorators added (none exist on any route today;
  slowapi keys by IP and covers anonymous if ops adds limits later).

## Critical files

### imbi-common (`/Users/daves/Source/projects/imbi-development/imbi-common`)

| File | Change |
|---|---|
| `src/imbi_common/models.py:277` | `Organization(Node)` is an empty subclass — add `public_documents_enabled: bool = True` |
| `docs/` | imbi-common is a published library; update the docs page that covers the models (per repo CLAUDE.md) |

### imbi-api (`/Users/daves/Source/projects/imbi-development/imbi-api`)

| File | Change |
|---|---|
| `src/imbi_api/auth/permissions.py` | New `get_optional_user` dependency: no credentials → `None`; credentials present → existing `_authenticate_token` path (invalid token still 401s). Mirrors `get_current_user` (line 617) |
| `src/imbi_api/endpoints/documents.py` | `is_private` on create/response/patch; public filter fragment; optional-auth on the three public GETs (org index, org single-doc, project-type list); anonymous email redaction |
| `src/imbi_api/endpoints/organizations.py:413-434` | Add `n.public_documents_enabled = {public_documents_enabled}` to both `_persist_organization` SET queries + params (line 436). Create/GET paths are model-driven and pick the field up automatically |
| `tests/endpoints/test_documents.py` | New anonymous/private/kill-switch cases (existing mock-graph pattern, base at line 15) |
| `tests/endpoints/test_organizations.py` | New-field round-trip through PATCH |
| `pyproject.toml` | TEMP `[tool.uv.sources]` pin to local `../imbi-common` during dev; swap to git-branch pin for PR CI; remove once imbi-common publishes (established pattern) |

### imbi-ui (`/Users/daves/Source/projects/imbi-development/imbi-ui`)

| File | Change |
|---|---|
| `src/types/index.ts:663` | `Document.is_private: boolean`; `created_by: string \| null`; `DocumentCreate.is_private?: boolean`; `Organization.public_documents_enabled: boolean` |
| `src/components/documents/DocumentsPinboardNew.tsx` | "Private" `Switch` + label in the create/edit form; include in `onSave` payload |
| `src/components/documents/useDocumentsController.ts:142` | Mirror the `is_pinned` JSON-Patch pattern for `/is_private`; include `is_private` in create payload |
| `src/components/documents/DocumentRowCells.tsx` | `Badge` (variant `neutral`) + Lucide `Lock` icon when `is_private`, alongside existing pin button (line 46) |
| `src/components/documents/DocumentsPinboardReader.tsx` | Same Private badge in the single-document reader header |
| `src/components/admin/organizations/OrganizationForm.tsx` | `Switch` for `public_documents_enabled` (default on), saved via existing `updateOrganization` JSON Patch |
| `src/types/api-generated.ts` | Document endpoints are hand-written in `types/index.ts` (snapshot predates them); user owns `npm run codegen:fetch` regeneration |

## Implementation outline (dependency order)

### 1. imbi-common

1. Add `public_documents_enabled: bool = True` to `Organization`.
   Old vertices lack the property; pydantic default covers reads
   (`db.match` validation in `patch_organization` keeps working).
2. Update the models documentation page; run `just lint` /
   `just test`.

### 2. imbi-api

1. **`auth/permissions.py`** — add:

   ```python
   async def get_optional_user(
       db: graph.Pool,
       credentials: security.HTTPAuthorizationCredentials
       | None = fastapi.Depends(oauth2_scheme),
   ) -> AuthContext | None:
       if not credentials:
           return None
       return await _authenticate_token(
           db, credentials.credentials, settings.get_auth_settings())
   ```

   (`oauth2_scheme` is already `HTTPBearer(auto_error=False)`.)

2. **`endpoints/documents.py`** — the flag:
   - `DocumentCreate.is_private: bool = False`; add
     `is_private: {is_private}` to `_DOCUMENT_CREATE_FRAGMENT`
     (line 303) and pass it in `_create_document_impl`.
   - `DocumentResponse.is_private: bool = False`;
     `created_by: str | None` (for redaction); default the field in
     `_parse_document_row` next to the existing `is_pinned`
     back-fill (line 173).
   - `DocumentUpdate.is_private: bool | None = None`; in
     `_patch_document_impl`: add to the `current` dict, resolve via
     `_resolve_patched_field`, add `n.is_private = {is_private}` to
     the SET query. It is *not* added to
     `_DOCUMENT_READONLY_PATHS`.

3. **`endpoints/documents.py`** — public reads:
   - Public filter fragment (`o`, `p`, `pt`, `u` are all in scope
     after `_ATTACHMENT_MATCH`, line 182; `pt IS NOT NULL` excludes
     both Project- and User-attached documents):

     ```cypher
     AND pt IS NOT NULL
     AND (n.is_private IS NULL OR n.is_private = false)
     AND (o.public_documents_enabled IS NULL
          OR o.public_documents_enabled = true)
     ```

   - `_list_documents_impl` and `_fetch_document` grow a
     `public_only: bool = False` parameter that appends the
     fragment.
   - Switch these GETs to
     `auth: AuthContext | None = Depends(get_optional_user)`:
     `list_documents` (line 557), `get_org_document` (line 710),
     `list_project_type_documents` (line 617). Compute
     `public_only = auth is None or not (auth.is_admin or
     'document:read' in auth.permissions)`.
   - When `auth is None`, null `created_by` / `updated_by` on each
     parsed row before model validation (keep `created_by_name`).
   - `list_user_documents` (line 643), `list_project_documents`,
     `get_document` (project router) and all writes are untouched.

4. **`endpoints/organizations.py`** — add the new property to both
   SET branches of `_persist_organization` and its `params` dict.

5. `just format` changed files, `just lint`, `just test`.

### 3. imbi-ui

1. Types in `src/types/index.ts` (Document, DocumentCreate,
   Organization).
2. Private switch in `DocumentsPinboardNew.tsx`; wire through
   `useDocumentsController.ts` create + patch.
3. Private badge in `DocumentRowCells.tsx` and
   `DocumentsPinboardReader.tsx`.
4. Org kill-switch in `OrganizationForm.tsx`.
5. `npm run lint`, `npm run format:check`; user runs
   `npm run codegen:fetch` if they want the snapshot refreshed.

## Tests

imbi-api (`tests/endpoints/test_documents.py`, existing mock-graph
pattern — override `get_optional_user` the same way
`get_current_user` is overridden today):

- Anonymous list returns public project-type docs; the generated
  Cypher contains the public filter fragment (assert on
  `mock_db.execute` call args).
- Anonymous single GET of a public project-type doc → 200 with
  `created_by`/`updated_by` nulled, `created_by_name` intact.
- Anonymous single GET of a private doc, a project-attached doc, and
  a user-attached doc → 404 (mock returns no rows under the filter).
- Anonymous request to the user-documents list → 401 (route keeps
  required auth).
- Anonymous + org kill-switch off → empty list / 404.
- Invalid bearer token on a public GET → 401.
- Authenticated with `document:read` → unfiltered query, emails
  present (regression).
- Authenticated *without* `document:read` → 200 public view.
- Create with `is_private: true` persists the flag; PATCH
  `/is_private` flips it; PATCH with explicit `null` → 400
  (`_resolve_patched_field` behavior).
- Writes without auth still 401.

imbi-api (`tests/endpoints/test_organizations.py`): PATCH
`public_documents_enabled` round-trips; orgs created before the
field default to `true`.

imbi-common: model default test for `public_documents_enabled`.

## Verification

```bash
# per submodule
cd imbi-common && just lint && just test
cd imbi-api   && just lint && just test
cd imbi-ui    && npm run lint && npm run format:check
```

End-to-end spot-checks against `just serve` (API on :8000):

```bash
# anonymous list succeeds, excludes private + project docs
curl -f http://localhost:8000/organizations/<org>/documents/

# anonymous read of a public project-type doc, emails redacted
curl -f http://localhost:8000/organizations/<org>/documents/<id> | jq '.created_by'

# private doc 404s anonymously, 200s with a token
curl -f http://localhost:8000/organizations/<org>/documents/<private-id>   # expect 404
curl -f -H "Authorization: Bearer $TOK" .../documents/<private-id>          # expect 200

# kill-switch: PATCH org public_documents_enabled=false, anonymous list now empty
curl -f -X PATCH -H "Authorization: Bearer $TOK" \
  -d '[{"op":"replace","path":"/public_documents_enabled","value":false}]' \
  http://localhost:8000/organizations/<org>
```

UI: create a doc with the Private switch on → lock badge appears in
list and reader; toggle org switch in Admin → Organizations.

## Correlated PRs / follow-ups

- PR order: **imbi-common → imbi-api → imbi-ui** (imbi-api carries a
  TEMP uv.sources pin until imbi-common publishes — same cleanup
  pattern as prior features).
- After merges: bump submodule pointers in `imbi-development` on a
  `feature/public-documents` branch.
- Deferred follow-ups: anonymous UI reader routes; public serving of
  uploads referenced by public docs; anonymous comment reading;
  rate limits on public endpoints.
