# Strava MCP Health Check False Negative — Schema Validation Bug in Third-Party Package

**Date:** 2026-07-05
**Severity:** P3 Low
**Status:** Resolved
**Affected:** Strava MCP integration (strava-mcp-server dependency), Morning Briefing and Sunday Maintenance cron health checks
**Duration:** Unknown start (pre-existing), surfaced during this week's cron verification work; resolved same day

---

## Summary

A newly restored MCP server health check began reporting the Strava integration as disconnected ("Connection may have expired") despite the underlying OAuth credentials being fully healthy and successfully refreshing on schedule. Root-cause investigation traced the failure to a schema validation bug in the third-party `strava-mcp-server` npm package: it requires a `weight` field on Strava's athlete API response that Strava's real API does not always return, causing a fully successful, authenticated API call to fail client-side validation and get misreported as a connection failure. No credentials were expired, revoked, or misconfigured. Fixed by vendoring a patched fork of the dependency.

---

## Timeline

| Time | Event |
|------|-------|
| Day 1 | Multi-server MCP health check restored to a consolidated morning cron job after being dropped during an earlier consolidation |
| Day 1 | Live-fire of the restored check reports Strava as failed; all other servers pass |
| Day 1 | Initial investigation confirms the failure is reproducible on repeated calls |
| Day 1 | Deeper trace shows the underlying Strava API call succeeds every time, with a valid authenticated response returned |
| Day 1 | Root cause identified: response schema validation in the MCP server package rejects the real API response over a missing optional field |
| Day 1 | Confirmed via cross-referencing exposed-vs-current secret fingerprints that a separate, unrelated credential exposure earlier in the week had already been fully superseded by normal token rotation — ruled out as a contributing cause |
| Day 1 | Upstream checked for an existing fix: none found in the latest published release or any open pull request |
| Day 1 | Package forked, patched, and vendored; dependency repointed; fix verified live across both affected cron jobs |

---

## Root Cause

The `strava-mcp-server` package's response schema for Strava's athlete endpoint declares several fields — including `weight` — as required, non-optional values. Strava's real API omits some of these fields depending on account state or privacy settings. When a field is missing, the package's schema validation throws an error *after* a fully successful, authenticated HTTP call — and the health-check tool's generic error handling reports this as a connection problem rather than surfacing the real cause (a client-side data validation mismatch).

This meant the health check's "disconnected" status was a false negative with no bearing on actual account connectivity. The underlying access and refresh tokens were healthy throughout, refreshing successfully on their normal schedule with real authenticated responses on every call.

---

## Remediation

1. Confirmed no upstream fix exists: checked the latest published package version, the project's unreleased development branch, and existing open pull requests addressing the same field — none merged or released.
2. Forked and vendored the package rather than applying a local dependency patch, for durability across future redeploys.
3. Applied the fix defensively rather than narrowly: made the specific field causing the observed failure optional, and reviewed the same schema for other fields with an equivalent nullable-but-not-optional pattern, fixing those preemptively rather than waiting for each to surface as a separate incident.
4. Repointed the running configuration at the vendored, patched version instead of the public package.
5. Verified the fix against a live, real API response shape before deploying, then confirmed a clean result on two independent test invocations and the real scheduled cron run.

---

## Prevention

- The vendored fork is tracked in version control alongside other infrastructure configuration, with documentation of what was changed and why, so the fix survives a fresh redeploy rather than living only in a running container's local state.
- The defensive fix to sibling fields in the same schema reduces the chance of a near-identical false negative recurring the next time Strava's API omits a different optional field for this or another account.

---

## Lessons Learned

1. **A failing health check and a broken connection are not the same claim.** The check correctly detected *something* wrong, but the natural assumption — credentials are bad — was wrong. Tracing the actual API call before accepting the check's own diagnosis avoided an unnecessary credential rotation for a problem that didn't exist.
2. **A generic catch-all error message can misattribute a specific failure.** The tool's error handling collapsed a schema validation exception into the same message it would show for a genuine auth failure, actively steering troubleshooting toward the wrong cause until the actual thrown error was inspected directly.

---

*Environment: KONYKS-SERVER · Hermes-Agent · Strava MCP integration · homelab-infra*
