# Plan: UI Quality-of-Life Improvements

## Context

`features/ui-qol-improvements.md` collects three small, loosely related
papercuts on the Operations Log and Project Detail screens:

1. The **operations log row** truncates the project name on the first
   line — the project column is locked to `minmax(140px,180px)` while
   the env/release-train column is allowed `minmax(560px,4fr)`, so
   common names like `imbi-plugin-github` already get clipped on
   typical viewports.
2. There is **no path from an operations-log entry back to the project
   page**. Today the entry's project name is non-interactive text —
   you must navigate via the org sidebar / projects index.
3. The **Project Detail header** shows the project's name and slug but
   not its UUID, and you cannot copy that UUID anywhere from the UI
   even though it shows up everywhere in the URL bar and is the
   foreign key in plugin payloads and operations-log entries.

All three are pure UI changes — no API or model work — so they live
entirely in `imbi-ui`.

## Decisions

User-confirmed choices (do not re-derive during implementation):

- **Project-detail nav from ops log**: add a **"Go to Project"** button
  alongside **Duplicate** in `OperationsLogEntryDetails`, not an inline
  link on the row's project name. The row stays a `<button>` (no
  refactor); the new button uses a real router `<Link>` so
  ⌘-click / middle-click / right-click → open-in-new-tab work.
- **Project-column width**: bump the fixed range from
  `minmax(140px,180px)` → `minmax(180px,320px)` in
  `opsRowLayout.ts`. Predictable column alignment; truly extreme
  names still truncate but the existing `title=` tooltip already
  surfaces the full name.
- **Copy-ID affordance**: small icon-only ghost button next to the
  name in the project-detail header, lucide `Copy` icon, Tooltip
  reading "Copy project ID", flashes to a green `Check` for ~2 s on
  success. Copies `project.id` (the UUID-string, the same value
  embedded in `/projects/<id>` URLs). Reuses the existing
  `useClipboard` hook.

## Critical files (imbi-ui)

### 1. Wider project column

- `imbi-ui/src/components/operations-log/opsRowLayout.ts` (~L1–10) —
  change column 3 of `OPS_ROW_GRID` from `minmax(140px,180px)` to
  `minmax(180px,320px)`. Affects both
  `OperationsLogStreamRow.tsx:L132–138` and
  `OperationsLogReleaseCard.tsx:L142–148` automatically since they
  share the same grid template.

No other files change for this fix — the `truncate` class on the
project span stays (defense for very long names).

### 2. "Go to Project" button in the entry details

- `imbi-ui/src/components/operations-log/OperationsLogEntryDetails.tsx`
  - Add `Link` import from `react-router-dom` and `ExternalLink` (or
    keep `ExternalLink` if already imported — it's already used for
    the entry's `record.link` block at L113–122).
  - The entry already carries `record.project_id` (used to build
    `duplicateInitialValues.project_id`) — reuse it as the link
    target: `to={`/projects/${record.project_id}`}`.
  - Render a sibling button **before** the Duplicate button in the
    bottom action row (`flex justify-end pt-2` at L162) so the
    primary navigation action sits to the left of the
    "create-another" action. Use
    `<Button asChild size="sm" variant="outline">` wrapping the
    `<Link>` so it inherits the same outline style as Duplicate.
  - Icon: `ExternalLink` (already imported) — semantically matches
    "go to that thing".
  - Label: `Go to Project`.

No type/changes to `OperationsLogRecord`; `project_id` is already on
the record (confirmed via existing `duplicate_initial_values`
construction at `OperationsLogEntryDetails.tsx:L53–62`).

### 3. Copy-ID button in the project header

- `imbi-ui/src/components/ProjectDetail.tsx`
  - Add `useClipboard` import from `@/hooks/useClipboard` and `Copy`,
    `Check` to the lucide imports already at L7–15.
  - In the header `<div className="mb-1 ml-[-18px] flex items-center gap-3">`
    (around L664), add a third child after the archived badge:
    a `TooltipProvider` → `Tooltip` → `TooltipTrigger asChild` →
    icon-only ghost `<Button size="sm" variant="ghost">`. Tooltip
    content: `Copy project ID`. Click handler: `void copy(project.id)`
    — no `key` argument needed since this is a single-button hook
    instance.
  - Render `copied ? <Check className="size-4 text-green-500" /> : <Copy className="text-secondary size-4" />`
    (matches `ScoringPolicyManagement.tsx:L235–239`).
  - `aria-label`: `Copy project ID` so the icon-only button is
    accessible.

Reuses, no new utilities:

- `useClipboard` (`imbi-ui/src/hooks/useClipboard.ts`) — already
  handles writeText, flash timeout, and cleanup. Same pattern as
  `BlueprintDetail`, `ScoringPolicyManagement`, `SettingsApiKeys`,
  `ApplicationSecretsPanel`, `RevealSecret`.
- shadcn `Button`, `Tooltip*` components already imported in both
  edited files.

## Implementation outline

Ordered so each step is independently verifiable:

1. **Column width fix** (smallest, lowest risk) — single one-line
   change in `opsRowLayout.ts`. Visual smoke-test only.
2. **"Go to Project" button** — add to `OperationsLogEntryDetails`.
   Verifies independently: open any entry's expanded details on the
   `/operations-log` page, click the new button, confirm it lands on
   `/projects/<that project's id>`. Also confirm ⌘-click opens a new
   tab.
3. **Copy-ID button** — add to `ProjectDetail.tsx` header. Verifies
   independently: visit any project, click the copy icon, confirm
   the icon swaps to the green check for ~2 s and the clipboard
   contains the UUID that matches the URL.

Each step is a single, focused commit; the meta-repo only tracks the
submodule pointer once everything has landed on `imbi-ui` `main`.

## Tests

Existing test files relevant to the touched code:

- `imbi-ui/src/components/operations-log/__tests__/renderOpsLogTemplate.test.ts`
- `imbi-ui/src/components/operations-log/__tests__/parseDescription.test.ts`
- `imbi-ui/src/components/__tests__/RecentActivity.test.tsx`

None of those exercise the row layout, the entry-details action row,
or the project-header copy button. Two new (small) tests are
warranted:

- **`OperationsLogEntryDetails.test.tsx`** (new) — render the
  component with a record, assert the rendered link has
  `href="/projects/<project_id>"` and the accessible name
  `Go to Project`. Wrap in `MemoryRouter`.
- **`ProjectDetail.test.tsx`** copy-button test — if a project-detail
  test file already exists, add a case; otherwise inline-snapshot the
  header: mock `navigator.clipboard.writeText`, click the copy
  button, assert `writeText` was called with `project.id` and the
  icon flipped to `Check`. (Skip if no project-detail test scaffold
  exists — the existing patterns in `BlueprintDetail` /
  `ScoringPolicyManagement` are already well-tested for the
  `useClipboard` flow itself.)

No regression test needed for the column-width tweak — purely visual.

## Verification

Run inside `imbi-ui`:

```bash
npm run lint
npm run format:check
npm run build
npm test -- src/components/operations-log
```

End-to-end spot-checks against a running stack (`just serve` in
`imbi-development`):

- **Truncation**: open `/operations-log` with `imbi-plugin-github` or
  another ≥18-char project among the entries; the project name in
  row 1 should render in full at viewport widths ≥ 1200 px and
  truncate gracefully (with title tooltip) below that.
- **Navigate**: expand any entry → click `Go to Project` → land on
  `/projects/<id>` (overview tab). ⌘-click opens the project in a
  new tab.
- **Copy ID**: open any project's detail page → click the new copy
  icon → icon flashes to a green check for ~2 s → paste into a
  scratch buffer and confirm it matches the UUID in the URL.
- **Embedded ops-log unchanged**: open a project's
  `Operations Log` tab; the project filter is hidden, and the
  `Go to Project` button (in the entry details) navigates to the
  same project it was filtered to — confirm the button still works
  and the row still expands/collapses on click.
