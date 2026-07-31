# Actionable Project Doctor + per-user analysis credentials

## Intent

Project Doctor today only *diagnoses*. The GitHub doctor
(`plans/github-project-doctor.md`) emits `pass`/`warn`/`fail` findings
whose `description` tells the operator the remedy in prose ("Re-run the
lifecycle plugin to repair the edge", "Update the project link"), but
there is **no user-facing way to perform those fixes** — there is no
button to "run the lifecycle plugin", and the only auto-fix that exists
is the hard-coded blueprint-defaults path (`apply-blueprint-defaults`),
which is special-cased outside the plugin model.

This feature makes findings **actionable**:

1. An analysis plugin can attach a remediation offer to a finding and
   perform the fix when asked. The GitHub doctor's diagnoses
   (identifier mismatch, canonical-URL shape, dashboard-link mismatch,
   `github-repository` link mismatch, missing `EXISTS_IN` edge) all
   become one-click fixes.
2. The hard-coded blueprint property doctor
   (`blueprint_compliance.py` + the `apply-blueprint-defaults`
   endpoint) is reframed to flow through the same remediation
   mechanism instead of being a bespoke special case.

It also folds in a related gap discovered while debugging: analysis
plugins have **no access to the acting user's credentials**. The
GitHub doctor can only authenticate with a static operator-pasted
`access_token`; remove it and every body-dependent check degrades to
`warn` because the unauthenticated call to a private GHEC repo returns
401. Deployment/lifecycle plugins already mint a per-user token via the
identity flow; analysis should too.

## Affected services

- **imbi-common** — the `AnalysisPlugin` contract (new remediation
  types + optional `remediate()` method).
- **imbi-plugin-github** — the GitHub doctor implements `remediate()`.
- **imbi-api** — wire per-user identity into the analysis run path,
  add a remediation endpoint, fold the blueprint doctor into it.
- **imbi-ui** — per-finding "Fix" buttons + a generic "Fix all",
  replacing the bespoke "Apply Blueprint Fixes" button.

## Key shape

- `AnalysisResultItem.remediation: RemediationOffer | None`
- `AnalysisPlugin.remediate(ctx, credentials, remediation_id) ->
  RemediationResult` (optional; default raises).
- Remediation effects graph changes through the **existing write-back
  channel** (`ctx.service_writeback` / `ctx.link_writeback`, persisted
  by the host) — the same mechanism lifecycle plugins use. No new
  DB-write layer, no logic duplicated from the lifecycle plugin.
- Per-user credentials use the **sibling identity plugin attached to
  the same third-party service** (e.g. `github-enterprise-cloud` on the
  `github` TPS).

See `plans/doctor-remediation.md` for the plan of record.
