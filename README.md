# imbi-orchestration

Cross-service steering for the Imbi multi-repo. This repo holds the
**feature descriptions**, **implementation plans**, and **shared
notes** that coordinate work across the Imbi service repositories
(`imbi-api`, `imbi-common`, `imbi-gateway`, `imbi-mcp`, `imbi-ui`,
`imbi-plugin-*`, etc.).

It's deliberately small: three directories of Markdown plus a
`CLAUDE.md` that tells Claude Code how to drive the
description → plan → implementation → pull-request workflow. The
code itself lives in the individual service repos; this repo
records the *why* and *how* of work that touches more than one of
them.

```
imbi-orchestration/
├── CLAUDE.md          # workflow guidance for Claude Code
├── README.md          # this file
├── features/          # short intent docs, one per cross-service feature
├── plans/             # approved implementation plans, one per feature
└── notes/             # durable reference material (schemas, payloads, …)
```

## Why a separate repo?

The steering artifacts are useful independent of any one
developer's local workspace layout. Pulling them into their own
repo lets:

- multiple developers share the same plans and notes without
  forking a personal meta-repo;
- the artifacts live alongside (not inside) any one service repo,
  so they're not biased toward the API, the UI, or any other
  service;
- the workflow guidance in `CLAUDE.md` follow the artifacts rather
  than being duplicated into every developer's setup.

## Recommended workspace layout

The intended use is to check this repo out as a **git submodule
alongside the Imbi service submodules** in your own development
workspace. A typical layout:

```
<your-imbi-workspace>/
├── .gitmodules
├── imbi-orchestration/      ← this repo (steering)
├── imbi-api/                ← service
├── imbi-common/             ← shared library
├── imbi-gateway/            ← service
├── imbi-mcp/                ← service
├── imbi-ui/                 ← service
├── imbi-plugin-github/      ← plugin
└── …
```

`CLAUDE.md` assumes that the service submodules are siblings of
`imbi-orchestration/`. When Claude Code runs in the workspace
root, it picks up the workflow from
`imbi-orchestration/CLAUDE.md` and finds the services next door.

## Getting started

### If you already have a meta-repo

From the root of your workspace meta-repo:

```sh
git submodule add <url-to-imbi-orchestration> imbi-orchestration
git submodule update --init --recursive
```

Then point your workspace's top-level `CLAUDE.md` (if you have one)
at `imbi-orchestration/CLAUDE.md` so Claude Code picks up the
workflow. A minimal forwarding stanza:

```markdown
For the cross-service feature workflow (description → plan →
implementation → correlated PRs), read
`imbi-orchestration/CLAUDE.md`. The directories `features/`,
`plans/`, and `notes/` referenced there live inside that
submodule.
```

### Starting from scratch

Create a directory for the workspace, then add this repo plus the
service repos you care about as submodules:

```sh
mkdir my-imbi-workspace && cd my-imbi-workspace
git init
git submodule add <url-to-imbi-orchestration> imbi-orchestration
git submodule add <url-to-imbi-api>           imbi-api
git submodule add <url-to-imbi-ui>            imbi-ui
git submodule add <url-to-imbi-gateway>       imbi-gateway
# …add the other services as needed
git commit -m "Bootstrap Imbi workspace"
```

Add a short top-level `CLAUDE.md` that points at
`imbi-orchestration/CLAUDE.md` and you're ready to drive features
through the workflow.

## Adding a feature

The full procedure lives in [`CLAUDE.md`](./CLAUDE.md). In short:

```
1. Write imbi-orchestration/features/<slug>.md.
2. Ask Claude: "review features/<slug>.md and make a plan."
3. Iterate via AskUserQuestion until the plan reflects your intent.
4. Approve the plan (Claude calls ExitPlanMode).
5. Copy the approved plan to imbi-orchestration/plans/<slug>.md
   and commit it here.
6. Ask Claude: "start implementing — order the work as you see fit"
   (or specify the order yourself). Code changes land in the
   service submodules, not in this repo.
7. Ask Claude: "create correlated pull requests for the changes."
8. Once each component PR is merged, bump the submodule pointers
   in your parent workspace.
```

## Conventions

The full convention list is in [`CLAUDE.md`](./CLAUDE.md). The
short version:

- Search with `rg`, never `grep -r`.
- Use `Read` / `Edit` / `Write` rather than shell `cat` / `sed`.
- Python: call `.venv/bin/<tool>` directly; don't `source` the
  venv.
- `curl` always with `-f` / `--fail`.
- Parse JSON with `jq`, never `grep`.
- One feature branch slug shared across service submodules
  (`feature/<slug>`).
- Atomic commits per service; one PR per service; cross-link the
  PRs.

## License

BSD 3-Clause License, see LICENSE.
