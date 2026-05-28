# Imbi orchestration

This directory holds the cross-service **steering artifacts** that
coordinate work across the Imbi services (`imbi-api`, `imbi-common`,
`imbi-gateway`, `imbi-mcp`, `imbi-ui`, `imbi-plugin-github`, etc.).
It is designed to live as a sibling git submodule next to those
service submodules in a developer's workspace, so the workflow files
travel with the developer rather than being trapped inside any one
component repo.

```
<workspace>/
├── imbi-orchestration/      ← this repo (steering)
│   ├── features/
│   ├── plans/
│   └── notes/
├── imbi-api/                ← service submodule
├── imbi-ui/                 ← service submodule
├── imbi-gateway/            ← service submodule
└── …
```

When this repo is checked out as a submodule, treat its parent
directory as the workspace root. The workflow below assumes the
service submodules are siblings of `imbi-orchestration/`.

## What lives here

| Directory   | Purpose                                                                       | Lifetime              |
|-------------|-------------------------------------------------------------------------------|-----------------------|
| `features/` | Short, human-written intent for a cross-service feature                       | Until plan exists     |
| `plans/`    | Approved implementation plans, one per feature, the *plan of record*          | Permanent             |
| `notes/`    | Reference material (schemas, payloads, refactor state) shared across features | Permanent             |

`features/<slug>.md` and `plans/<slug>.md` share the same slug. A
feature's plan supersedes its description once approved, but both
files stay so future readers can trace from intent to plan.

`notes/` is the catch-all for durable reference material — graph
schemas, sample payloads, in-flight refactor state. Anything that
multiple features reference (or that's too big to inline into a
plan) belongs here.

## Feature workflow

We treat a feature as a three-stage artifact: **description → plan →
implementation**. Each stage produces a file we can refer back to.

```
features/<slug>.md      # human-written intent, refined by Q&A
plans/<slug>.md         # implementation plan, committed for review
                        # (code edits go into the relevant service
                        #  submodule, not this repo)
```

### Stage 1 — feature description (`features/<slug>.md`)

The user writes a short Markdown file in `features/` describing what
they want and why. It typically names the affected service
submodules, points at relevant graph edges / endpoints / payload
fields, and may sketch an API surface. It is *intent*, not a plan —
gaps and ambiguities are expected and get resolved in Stage 2.

### Stage 2 — interactive planning

When the user says "review `features/<slug>.md` and make a plan,"
follow this loop:

1. **Read the feature file first.** Don't start exploring before you
   know what's being asked.
2. **Explore the codebase.** Prefer `rg` (it honors `.gitignore`, so
   it skips `.venv/` and `node_modules/`). Never use `grep -r` or
   `find ... -exec grep` — they crawl through dependencies and
   produce noisy results. Search across the sibling service
   submodules from the workspace root.
3. **Look first, invent second.** Search for existing helpers,
   endpoints, edge labels, and data shapes that already fit the need.
   Cite file paths and line numbers in your reasoning so the user
   can verify. The plan should reuse what's already there before
   adding anything new.
4. **Consult `notes/` for prior context.** Before re-deriving a
   schema or payload shape, check whether a note already documents
   it. If you produce durable reference material while planning, add
   it to `notes/` rather than burying it in the plan.
5. **Surface real branch points with `AskUserQuestion`.** Use it for
   genuine forks — naming, persistence model, where new code lives,
   semantic edge cases. Group 2–4 mutually exclusive options per
   question; lead with the recommended option labelled
   "(Recommended)" only when one truly stands out. Do NOT use
   `AskUserQuestion` to ask "is the plan ready?" — that's what
   `ExitPlanMode` is for.
6. **Decide what *not* to ask.** Reasonable defaults (status maps,
   verbatim slug pass-through, error-on-unknown vs. log-and-skip)
   should be chosen and stated, not asked.
7. **Write the plan to the plan file.** While in plan mode the only
   file you may edit is the plan-mode file under
   `~/.claude/plans/...`. Build it incrementally as questions
   resolve. The plan must include:
   - A **Context** section explaining *why*.
   - A **Decisions** section listing user-confirmed choices verbatim
     so future readers don't have to re-derive them.
   - A **Critical files** map per service submodule with file paths
     (and line anchors when stable).
   - An **Implementation outline** ordered by dependency.
   - A **Tests** section listing the unit / integration cases.
   - A **Verification** section with concrete commands and
     end-to-end spot-checks (`just lint`, `just test`, sample
     `curl`s, etc.).
8. **Call `ExitPlanMode` exactly once,** when the plan is complete.
   Don't text-ask "does this look good?" — `ExitPlanMode` is the
   approval prompt.
9. **Copy the approved plan into `plans/<slug>.md`.** Once it's in
   `plans/`, it's part of the repo and considered the *plan of
   record*.

### Stage 3 — implementation hand-off

When the user says "start implementing" (or the equivalent — e.g.
"start with the imbi-api and UI changes, then move on to the
gateway"):

1. **Create discrete tasks with `TaskCreate`.** One task per logical
   unit, ordered so prerequisites are obvious. Mark each one
   `in_progress` *before* you start work and `completed` *only* when
   its lint, format, type-check, and tests all pass — never batch
   completions. If you hit an unrelated obstacle, file a follow-up
   task rather than expanding the current one.
2. **Work submodule-by-submodule.** Each Imbi service has its own
   linters, formatters, and test command (usually `just lint` and
   `just test`). Run them in the submodule you just edited before
   moving to the next. Don't ship cross-submodule churn from a
   single commit; each service submodule is its own git repo.
3. **Surgical edits only.** Don't refactor unrelated code, don't
   widen public APIs unprompted, don't add backward-compatibility
   shims for things that don't exist yet. If a follow-up cleanup is
   tempting, propose it; don't do it silently.
4. **Reuse existing helpers.** Before adding a new model, endpoint,
   or graph edge, double-check that the plan's "Critical files"
   pointers haven't pre-existed half the work. (In the deployment-
   webhook example, the `find_user_by_subject` graph helper already
   existed; only the HTTP wrapper was new.)
5. **Don't drift from the plan without telling the user.** If the
   plan needs to change mid-flight (e.g., a graph node turns out
   not to persist a field the plan assumed), surface the trade-off
   with a short message and either pick the safer option or ask.
   Update the plan file if the change is durable.
6. **Hand off cleanly.** End-of-implementation summary should list,
   per service submodule: what changed, what tests pass, what
   coverage is now, and any deferred follow-ups (e.g.
   "ACTIONS_IMBI_TOKEN must be issued from the API admin UI before
   `just serve` works").

### Stage 4 — correlated pull requests

When the user says "create correlated pull requests for the
changes," each affected service submodule gets its own PR, but
they're branded as a set so reviewers can see the whole feature.

1. **Clarify durable choices first.** Before opening anything,
   confirm via `AskUserQuestion`: base branch per submodule, Jira
   ticket (or none), and push remote (origin vs. a fork — `imbi-api`
   in particular often pushes from a personal fork). These flow
   into commit trailers and PR metadata that's painful to undo.
2. **Use one feature-branch slug across submodules.** Typically
   `feature/<slug>` matching the `plans/<slug>.md` filename. Makes
   the set discoverable from any one submodule's history.
3. **Honor AWeber commit + branch standards.** Use the
   `git-repository:branch-creation`, `git-repository:git-commit`,
   and `git-repository:pr-creation` skills rather than hand-writing
   summaries. They enforce 60-char imperative subjects, bullet
   bodies, the `Co-authored-by` trailer, kebab-case branch names,
   and the Summary / Problem / Solution PR template.
4. **Submodule-by-submodule, in dependency order.** Default order
   is API → UI → Gateway: UI generated types follow the API, and
   the gateway integrates against both. Per submodule:
   1. Verify `git status` — any unrelated dirty files get stashed
      or reverted; only the feature's files are committed.
   2. Branch from the user-confirmed base (usually `origin/main`).
   3. Stage explicit file paths (`git add path1 path2 ...`) —
      never `git add -A` / `git add .`, which can sweep up secrets
      or generated artifacts.
   4. Commit. Atomic commits are preferred; one logical change per
      commit even if the PR ships several.
   5. Push to the agreed remote.
   6. Open the PR.
5. **Cross-reference the PRs.** Each PR description should list
   its companion PRs (`Companion PRs: ...`) and link to
   `imbi-orchestration/plans/<slug>.md`. Reviewers should be able
   to walk the set from any entry point.
6. **CI gates each PR independently.** Don't merge any of them
   until they're individually green. The order of merge is the
   same as the order of opens (API first, then UI, then Gateway)
   so downstream PRs can rebase onto a known-good upstream.
7. **Defer parent-repo pointer bumps.** If the workspace is itself
   a meta-repo that tracks submodule SHAs, don't commit pointer
   updates until each component PR is merged on its submodule's
   `main`. That's a separate, follow-up commit on the meta-repo,
   once everything has landed.
8. **Hand off the URLs.** End-of-stage summary lists every PR URL,
   plus any pending CI failures or review threads.

## Conventions (apply across submodules unless overridden)

- **Search:** `rg` only. `rg -t py PATTERN`, `rg -g '*.tsx' PATTERN`,
  `rg -l PATTERN` for filenames.
- **Reading files:** prefer `Read`. Don't `cat`/`head`/`tail` from
  the shell.
- **Editing files:** prefer `Edit`/`Write`. Don't `sed`/`awk`.
- **Python virtualenvs:** call binaries directly
  (`.venv/bin/pytest`, `.venv/bin/mypy`) instead of activating with
  `source .venv/bin/activate &&`.
- **TypeScript / UI:** the UI's `api-generated.ts` is regenerated
  by `npm run codegen:fetch` against a running API. When the API
  schema changes, update the generated types alongside the code,
  but the user owns the regeneration step.
- **Plan-mode discipline:** while plan mode is active, the *only*
  writable file is the plan file. Don't pre-write code you'll add
  later; capture it in the plan instead.
- **Curl:** always include `-f`/`--fail` so failures surface.
- **JSON parsing:** use `jq`, not `grep`.

## Adding a new feature

```
1. Write features/<slug>.md.
2. Ask Claude: "review features/<slug>.md and make a plan."
3. Iterate via AskUserQuestion until the plan reflects your intent.
4. Approve the plan (Claude calls ExitPlanMode).
5. Copy the approved plan to plans/<slug>.md.
6. Ask Claude: "start implementing — order the work as you see fit"
   (or specify the order yourself).
7. Ask Claude: "create correlated pull requests for the changes."
   Confirm base branch, Jira ticket, and push remote when prompted.
8. Watch CI on each PR; merge in API → UI → Gateway order.
9. After every component PR is merged, bump the submodule pointers
   in your parent meta-repo on its own branch and PR (if you have
   one).
```

The deployment-webhook feature (`features/deployment-webhook.md`,
`plans/deployment-webhook.md`) is the canonical example of this
workflow.
