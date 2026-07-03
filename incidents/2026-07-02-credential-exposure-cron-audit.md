# Credential Exposure During Cron Audit and Remediation

**Date:** 2026-07-02
**Severity:** P0 Critical
**Status:** Resolved (all exposed credentials rotated and confirmed dead; a small number of non-urgent cleanup items remain open and tracked)
**Affected:** Hermes-Agent (GitHub, Nextcloud, Strava MCP integrations), Komodo/Periphery GitOps (git authentication across the full stack fleet), Nextcloud account authentication
**Duration:** Single day, one continuous session

---

## Summary

A routine Hermes cron-fleet audit required inspecting Hermes's config file for MCP server grants. A raw file read exposed a live GitHub PAT, a Nextcloud app-password token, and Strava OAuth tokens in plaintext into a Claude Code session transcript. Remediation of that exposure produced four further incidental exposures of its own — a secrets-manager listing command, a git diagnostic command, a mistaken credential-generation flag, and a `grep` against an unfamiliar leftover script — each involving a different tool but the same underlying pattern: reading content without first assuming it might hold live secrets. One of those incidental exposures surfaced a pre-existing, previously undetected drift between the secrets store and what was actually deployed. Remediation also uncovered a scoping gap in its own first pass: an app-level credential tied to the original exposure was left unrotated for the duration of the incident because the initial fix only addressed the more visible per-user tokens. Every exposed credential — five distinct sets in total — was independently verified dead, not merely assumed rotated. One recommendation made mid-incident (removing embedded-credential steps from an automation script) was tested via canary before being applied and was disproven, avoiding a change that would have had no effect. A small number of low-urgency cleanup items remain open and are being tracked rather than resolved unsafely.

---

## Timeline

| Stage | Event |
|-------|-------|
| Audit | Cron-fleet audit requires inspecting Hermes's config file for MCP server grants |
| Exposure 1 | A raw, unscoped read of the config file prints a GitHub PAT, a Nextcloud app-password token, and Strava OAuth tokens into the session transcript |
| Remediation begins | Decision made to rotate every exposed credential, not a partial subset |
| Exposure 2 | A secrets-manager command run in its listing form (instead of a scoped single-secret retrieval) prints three additional live GitHub PATs into the same transcript; one doesn't match what's actually deployed — a pre-existing secrets-store/production drift is discovered as a side effect |
| GitHub PATs rotated | All three tokens rotated; prior values independently confirmed revoked via API |
| Near-miss | A revocation check for one token initially runs against a placeholder value by mistake instead of the real prior value — caught before being reported, redone correctly |
| Nextcloud rotation begins | An underspecified instruction ("rotate the Nextcloud password") doesn't distinguish the account login password from a separate app-specific credential |
| Unintended side effect | The literal, more common reading of the instruction resets the actual account login password — unrelated to the credential that was actually exposed |
| Exposure 3 | A mistaken command flag during app-password regeneration prints the newly generated token into the transcript; self-caught and revoked within the same minute |
| Nextcloud resolved | The correct app-password credential is revoked and replaced, verified live; a new account login password is generated, retrieved, and its temporary storage location deleted |
| Strava, first pass | Per-user access/refresh tokens deauthorized and independently confirmed fully revoked; grant re-authorized |
| Exposure 4 | During unrelated follow-up work, a diagnostic git command against a GitOps checkout prints a live GitHub PAT embedded in the remote URL |
| Exposure 5 | Inspecting a stale leftover script from an earlier, unrelated session prints a hardcoded API credential — later confirmed to be currently active in production, not abandoned |
| Recommendation tested, disproven | A proposed fix (removing embedded-credential steps from an automation script) is tested via canary before rollout; the automation re-embeds credentials from its own native configuration on every run regardless, so the proposed change would have had no effect |
| Scoping gap found | Reconciling the full exposure list surfaces that the Strava exposure included an app-level client secret that the first remediation pass never rotated — the deauthorize/reauthorize cycle only covered per-user tokens |
| Strava, second pass | Client secret regenerated; the existing OAuth grant tested against the new secret via a live call before any service restart; deployed and verified; prior secret independently confirmed dead |
| Closeout | All five distinct credential exposures confirmed dead; three stale config backups found and removed; remaining low-urgency items (transcript cleanup, historical log entries containing the now-dead Strava secret, a directory of other stale scripts) documented and deferred rather than resolved unsafely |

---

## Root Cause

Several independent causes, one of which recurred across the incident:

1. **Unscoped reads of credential-bearing content (recurring, four distinct instances):** a raw file read, a secrets-manager listing command, a git diagnostic command, and a structural inspection of an unfamiliar script all printed live credentials into the session transcript. Four different tools, one underlying pattern — reading for structure or content without first treating the target as potentially secret-bearing and choosing a read method accordingly.

2. **Ambiguous rotation instruction:** a request to rotate "the Nextcloud password" didn't specify which of two distinct credentials was meant. The literal interpretation of the more common reading produced a real, unintended change to an unrelated credential.

3. **Incomplete first-pass scope on one credential family:** the initial Strava remediation treated the exposure as fully covered by rotating per-user OAuth tokens, without accounting for a longer-lived, app-level secret that was part of the same original exposure. This wasn't caught until a later reconciliation pass across the full incident.

4. **Pre-existing, independently discovered drift:** one GitHub credential's value in the secrets store didn't match what was actually deployed in production — a discrepancy that existed before this incident and was only surfaced because the incident forced a direct comparison.

5. **An unverified recommendation entered the remediation plan:** a proposed script change was based on reasoning about how credentials were embedded, not on observed behavior. A later canary test showed the reasoning was incomplete — the underlying automation re-embeds credentials from its own configuration independent of the script step in question.

---

## Remediation

- **GitHub (three separate PATs):** all rotated to newly generated values, deployed to every consumer identified for each, and verified via live authenticated calls. Every prior value was independently confirmed revoked via API response, not assumed dead from rotation alone.
- **Nextcloud:** the actually-exposed app-specific credential was revoked and replaced, verified via a live call. The unintentionally-changed account login password was regenerated, delivered through a temporary, permission-restricted file, retrieved, and the file deleted. Three stale configuration backups found to contain old credential values in plaintext (discovered across two separate passes) were deleted.
- **Strava:** resolved in two passes. The first deauthorized and re-verified the per-user OAuth grant as fully revoked, then re-authorized it. The second, prompted by a later scoping review, rotated the separate app-level client secret — the existing OAuth grant was tested against the new secret via a live call *before* any service restart, to confirm continuity rather than assume it, then deployed and verified. The prior secret was independently confirmed dead via an explicit invalid-credential response, not just an expired-token response.
- **Incidental exposures during remediation:** each was self-caught at the point of occurrence. The credential printed by the secrets-manager listing command was included in the GitHub rotation above rather than treated as a separate track. The mistakenly-printed Nextcloud token was revoked within the same minute it was generated. The git-diagnostic exposure and the stale-script exposure were both confirmed to involve credentials already covered by rotations described above, or handled via the process below.
- **A currently-active credential found in a stale, unrelated leftover script** was tested for validity (it was live) and reported rather than assumed abandoned; it was rotated through its owning system's normal credential-management process once confirmed still in production use, and the stale script was deleted.

---

## Prevention

- The credential-read rule used to brief future remediation work — previously scoped only to the single file involved in the first exposure — was corrected to state the general principle (scoped, single-item reads only, for any credential-bearing tool) in operator-side reference documentation used to draft future work. This is not a change deployed to production infrastructure itself.
- The incident's specific lessons were recorded in local operator memory: each incidental exposure pattern from this session, the credential-type ambiguity that caused the unintended password change, and refreshed credential-rotation tracking.
- Open follow-up: no server-side operational doctrine document has been updated to reflect the general "scoped reads only" principle — that remains outstanding, not a completed action.
- Open follow-up: one of the three rotated GitHub PATs was generated but is not deployed anywhere — the service it was intended for has been running on a shared credential instead, a pre-existing, already-documented gap rather than a new issue introduced here.
- Open follow-up: the automation responsible for GitOps checkouts re-embeds credentials directly into git remote URLs on every run, sourced from its own native configuration — this is a structural exposure surface independent of any rotation script, and switching that automation to key-based authentication would eliminate it. Not yet investigated.
- Open follow-up: the new Nextcloud app-specific credential was deployed directly to its consuming config file but not pushed to the centralized secrets manager (the automation identity used has read-only access there) — needs manual entry to prevent the same store-vs-deployed drift that was found elsewhere in this incident.
- Open follow-up: a small number of historical, unrelated session/debug logs were found to still contain the old (now confirmed dead) Strava secret. Left untouched deliberately, out of caution around editing another system's own historical records unilaterally — recommended for in-place redaction of just the credential string when convenient, not urgent given the credential is already dead.
- Open follow-up: a directory of other stale, leftover scripts from unrelated past sessions was identified as a likely location for additional hardcoded credentials, based on the one already found there. Not yet swept — flagged as a dedicated future task.

---

## Lessons Learned

**Read-scope discipline needs to be a default assumption, not a rule attached after the fact to whichever tool just caused a problem.** The same category of mistake — an unscoped read exposing live credentials — recurred across four different tools within a single remediation session that existed specifically to fix the first instance of it. A fix scoped to one tool doesn't transfer; the operating assumption ("could this contain a secret? read narrow") has to be applied before every read of unfamiliar or credential-adjacent content, regardless of which tool is in hand.

**Remediation scope should be defined by everything a credential grants, not by its most visible artifact.** Treating an OAuth integration's exposure as fully handled by rotating the per-user tokens left a separate, longer-lived app-level secret live for the entire duration of the incident, because the first response scoped to what was easy to see rather than everything the original exposure actually contained.

**Verifying the verification step matters as much as performing the fix.** A placeholder value very nearly produced a false "confirmed dead" report on a revocation check, and a plausible-sounding fix for one exposure surface was disproven by direct testing rather than accepted on reasoning alone. Both would have shipped as settled fact without someone deliberately re-checking the check itself — the discipline that catches an incident also has to be applied to the response to the incident.

---

*Environment: KONYKS-SERVER (Unraid) · Hermes-Agent · Komodo/Periphery · Nextcloud · homelab-incident-reports*
