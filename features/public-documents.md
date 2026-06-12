# Public non-project documents

Non-project documentation should be readable without authentication
by default. Project-attached **and user-attached** documentation
stays auth-only; this feature only changes documents attached to a
**ProjectType**, and the org-wide document index / generic
single-document GET filtered down to those.

Today every document GET in imbi-api requires a bearer token plus the
`document:read` permission (`require_permission('document:read')` in
`src/imbi_api/endpoints/documents.py`). There is no anonymous path
anywhere in the auth layer.

## Intent

- Anonymous (unauthenticated) requests to these read endpoints
  succeed, returning only public project-type-attached documents:
  - `GET /{org_slug}/documents/` and
    `GET /{org_slug}/documents/{document_id}` (org-wide router)
  - `GET /{org_slug}/project-types/{type_slug}/documents/`
- A single document can be marked **private** (`is_private`) to
  exclude it from anonymous access while remaining visible to
  authenticated users with `document:read`.
- Project-attached and user-attached documents must never be exposed
  anonymously — including through the org-wide index and the
  attachment-agnostic single-document GET, which currently resolve
  documents regardless of where they are attached.
- All writes (POST/PATCH/DELETE) stay authenticated; nothing changes
  there beyond the new flag being settable.

## Data notes

- Follow the `is_pinned` precedent for adding a boolean property to
  the Document vertex (default handling for rows written before the
  property existed).
- Anonymous reads must 404 (not 403) on private, project-attached,
  or user-attached documents so existence isn't leaked.
- An invalid/expired token should still 401 rather than silently
  degrading to anonymous.
- Anonymous responses redact author emails (`created_by`,
  `updated_by`) but keep `created_by_name`.

## Org-level kill-switch

`Organization.public_documents_enabled: bool` (default `true`). When
`false`, nothing in the org is anonymously readable regardless of
per-document flags. Effective anonymous visibility:
`org.public_documents_enabled && !doc.is_private`.

## UI

- imbi-ui gets an affordance to set / clear the private flag on a
  document (create/edit form), and a Private badge in document lists
  and the reader.
- The org admin form gets a switch for `public_documents_enabled`.
- **API-only anonymous access**: the UI itself stays behind login;
  anonymous UI reader routes are a deferred follow-up.

## Scope

- imbi-common: `public_documents_enabled` on the `Organization`
  model (+ library docs).
- imbi-api: optional-auth dependency, anonymous filtering in the
  document queries, the per-document flag, org persistence of the
  kill-switch, anonymous email redaction.
- imbi-ui: flag toggle + private badge; org settings switch;
  hand-written document types in `src/types/index.ts` (user owns the
  `npm run codegen:fetch` step).
- Out of scope / deferred: anonymous UI reader routes; comments on
  public documents stay auth-only; uploads embedded in public
  documents are still served auth-only (anonymous viewers may see
  broken images); rate limits on the public endpoints.

See `plans/public-documents.md` for the plan of record.
