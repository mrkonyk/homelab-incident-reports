# Credential Exposure During Cron Audit and Remediation

**Date:** 2026-07-02
**Severity:** P0 Critical
**Status:** Resolved (transcript cleanup deferred, tracked)
**Affected:** Hermes-Agent (GitHub, Nextcloud, Strava MCP integrations), Komodo/Periphery GitOps, Nextcloud account authentication
**Duration:** Single day, same continuous session

---

## Summary

A routine Hermes cron-fleet audit required inspecting Hermes's config file for MCP server grants. A raw file read exposed a live GitHub PAT, a Nextcloud app-password token, and Strava OAuth tokens in plaintext into a Claude Code session transcript. The same failure mode recurred during remediation of the first exposure, when a secrets-manager table-listing command exposed three additional live GitHub PATs into the same transcript — one of which turned out to not match what was actually deployed, revealing a pre-existing, previously undetected drift between the secrets store and production. Remediation also surfaced a second, independent issue: an underspecified rotation instruction led to a real user-facing side effect — an account login password was changed as an unintended consequence of rotating what was actually a separate app-specific credential. All exposed credentials were rotated and verified dead; the account login password was recovered and secured. Transcript cleanup for the still-live session file was deferred with a tracked, condition-checked handoff rather than resolved unsafely.

---

## Root Cause

Two independent instances of the same underlying pattern, plus one unrelated instruction-ambiguity issue:

1. **Unscoped credential read (primary):** a config file known to contain live credentials was read with a raw `cat` instead of a redacted/structural read, printing a GitHub PAT, a Nextcloud app-password token, and Strava tokens into the session transcript in full.

2. **Unscoped credential read (recurrence, different tool):** during remediation of (1), a secrets-manager command was run in its table-listing form instead of a scoped single-secret retrieval, printing three additional live GitHub PATs into the same transcript. One of the three — the token thought to be currently deployed — turned out to have a different value than what was actually live in the config file, indicating the secrets store and production had already drifted apart, undetected, before this incident.

3. **Ambiguous rotation instruction (independent contributing factor):** a rotation request specified "the Nextcloud password" without distinguishing between the account login password and an app-specific password token. The actually-exposed secret was the latter, but the literal instruction led to the account login password being reset first, as an unintended real side effect unrelated to the exposure itself.

---

## Remediation

- **GitHub PATs (3 total):** all rotated. The Hermes MCP token was redeployed to config and verified via a live MCP call. A second token, used for automated git pushes, was rotated through an existing automation script that propagated the new value to every downstream consumer. A third token was regenerated but found to be undeployed anywhere — the service it was meant for has been running on the shared push token instead, a pre-existing, already-documented gap rather than a new issue. All prior token values were confirmed revoked via API (401 responses), not just assumed superseded.
- **Nextcloud:** the actually-exposed app-password token was revoked and replaced, verified via a live MCP call. Two stale config backup files also found to contain the old value in plaintext were deleted. The unintentionally-changed account login password was regenerated, written to a root-only temporary file, retrieved, and the temp file deleted. A second, smaller leak occurred mid-remediation (a token-generation command printed its own output to the transcript) and was self-caught and revoked within the same minute.
- **Strava:** the OAuth grant was fully deauthorized (verified by confirming the refresh-token flow itself now fails, not just the access token) and re-authorized via a fresh browser consent flow.
- **Transcript cleanup:** deferred. The file was confirmed to be the live, actively-written session transcript — not safe to delete mid-session. Handoff recorded for a future session to check file liveness before deleting, rather than leaving an unscoped reminder.

---

## Prevention

- The credential-read rule used to brief future sessions — previously scoped only
  to the config file involved in the first exposure — was corrected to state the
  general principle (scoped/single-item reads only, for any credential-bearing
  tool) in operator-side reference documentation used to draft future work. This
  is not a change deployed to KONYKS-SERVER itself.
- The incident's specific lessons were recorded in local operator memory: the two
  secondary-leak patterns from this session, new operational notes on the
  credential-type ambiguity that caused the unintended password change (account
  login vs. app-specific token), and refreshed credential-rotation tracking.
- Open follow-up: no server-side doctrine document has been updated to reflect the
  general "scoped reads only" principle — that remains outstanding, not a
  completed action.
- Open follow-up: the third rotated GitHub PAT is generated but not deployed —
  finishing the separation from the shared push token is optional future cleanup,
  not urgent.
- Open follow-up: the new Nextcloud app-password token was deployed to config
  directly but not pushed to the secrets manager (the automation identity is
  read-only on writes there) — needs manual entry to keep the secrets store from
  drifting again.

---

## Lessons Learned

**Read-scope discipline has to be a general rule, not a per-tool patch.** The first exposure was caused by a broad read on one tool; a documented fix for that specific tool didn't stop the same category of mistake from recurring on a completely different tool during the very session meant to clean up the first exposure. The fix that actually holds is the general principle, not the specific command.

**Credential-type ambiguity is its own failure mode.** "Rotate the Nextcloud password" was underspecified — there were two distinct credentials it could have meant, and the literal reading of the more common one produced a real, unintended change to the other. Precise naming of *which* credential matters as much as precise execution of the rotation itself.

**Secrets-store drift is invisible until something forces a full audit.** One of the exposed tokens didn't match what was actually deployed — a discrepancy that had existed silently before this incident and was only caught because the exposure forced a side-by-side comparison. A periodic secrets-store-vs-deployed-reality check would catch this kind of drift proactively instead of by accident.

---

*Environment: KONYKS-SERVER (Unraid) · Hermes-Agent · Komodo/Periphery · Nextcloud · homelab-incident-reports*
