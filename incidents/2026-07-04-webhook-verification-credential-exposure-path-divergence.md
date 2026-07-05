# Webhook Chain Re-Verification, an Accidental Credential Exposure, and a Self-Caused Credential Path Divergence

**Date:** 2026-07-04
**Severity:** P1 High
**Status:** Resolved
**Affected:** Komodo webhook auto-deploy chain (verification only, no fix needed), Grafana admin credential in Infisical, Infisical machine identity permissions
**Duration:** Single extended session (~2 hours)

---

## Summary

A follow-up request to independently re-verify the webhook-based auto-deploy chain across all managed stacks — deliberately not trusting an earlier incident report's conclusion — confirmed the chain is genuinely working, with one real monitoring blind spot found (a Procedure's own success flag has failed on every automated run for days due to one already-known, unrelated container conflict, while the actual deploy mechanism it reports on works correctly). During a related check for dashboard-based alerting on that flag, a routine authentication probe against a monitoring service exposed a live credential in plaintext, caused by a tool-syntax incompatibility rather than an unsafe command pattern. The credential was rotated immediately with explicit authorization. Investigating why that credential's secrets-manager copy had drifted out of sync with its live value surfaced an incorrect root-cause trace (a prior rotation was initially believed to have silently failed) that was later corrected to the true cause: two independently-created "canonical" locations for the same secret existed in the secrets manager, and this session's own rotation — trusting the wrong one — is what actually caused the two to diverge. Both the credential and the secrets-manager structure were fully reconciled, and a broad permission grant made mid-session to unblock the fix was documented for later review as broader than strictly necessary.

---

## Timeline

| Time (relative) | Event |
|------|-------|
| T+0:00 | Independent re-verification of the webhook auto-deploy chain requested, explicitly not trusting a prior incident report's conclusion |
| T+0:15 | Full chain traced hop by hop; a live test push confirmed real end-to-end deploy across a sample of stacks |
| T+0:30 | A historical, already-resolved access-control issue on one hop found and correctly identified as no longer relevant |
| T+0:40 | A genuine monitoring gap found: an automation's own top-level success indicator had failed on every run for several days, while the mechanism it reports on was functioning correctly the entire time |
| T+0:50 | Root cause of the false failures confirmed: a single known, unrelated container naming conflict, isolated to one non-critical stack, never affecting the 22 stacks that matter |
| T+1:00 | Two verification gaps disclosed rather than glossed over: one API scope insufficient to inspect delivery history directly, one token scope insufficient to inspect a specific ruleset directly — both compensated for with strong circumstantial evidence, explicitly labeled as such |
| T+1:10 | Follow-up check launched to confirm nothing currently consumes the failing success flag directly (to rule out live false-positive alerting) |
| T+1:15 | During that check, an authentication probe against a monitoring dashboard failed due to a tool-syntax incompatibility, and the tool's own error output echoed a live credential in plaintext |
| T+1:16 | Exposure disclosed immediately and in full; rotation authorized explicitly before any action was taken |
| T+1:20 | Credential rotated live; success flag consumers confirmed clean (nothing currently wired to the failing indicator) |
| T+1:30 | A drift between the rotated credential's live value and its secrets-manager copy was investigated; an incorrect theory (a prior rotation had silently failed) was formed and reported |
| T+1:45 | On challenge, the theory was retraced and found to be built on an unverified assumption; the correct explanation was found instead |
| T+1:50 | Correct root cause confirmed: two separate, independently-created secrets-manager locations existed for the same credential; the wrong one had been trusted, and this session's own rotation had overwritten the live value without updating the correct location — which is what caused the divergence, not a prior failure |
| T+2:00 | Both locations reconciled to a single source of truth; a stale, unused location retired; a permission grant made mid-session flagged for follow-up review |

---

## Root Cause

**Webhook chain: no fault found.** The chain from the ingress edge through to stack redeploy was confirmed live and functioning, verified with a real test push rather than a configuration read alone. One historical access-control issue on the ingress hop was found in logs from several days prior, already resolved, and correctly excluded from the current findings.

**Monitoring blind spot.** A Procedure responsible for redeploying any lagging stack reports a single top-level success flag alongside its per-stack results. That flag had reported failure on every automated run for several consecutive days. The actual cause was a single, already-known naming conflict on one non-essential container, unrelated to the 22 stacks the mechanism is actually meant to keep current — every one of which converged correctly on every run. The flag's failure state was accurate to its narrow definition (not every target succeeded) but structurally misleading as an overall health indicator, since a monitoring or alerting integration built against that field rather than a more specific per-stack comparison would have produced permanent false-positive noise. Confirmed no current integration (cron reporting, dashboarding, automation rules) actually consumes that flag directly; the one existing Komodo-aware automated check already correctly compares per-stack deployed-versus-latest state instead.

**Credential exposure.** While probing whether the monitoring dashboard for the automation platform hosted any alerting tied to Komodo, an authentication attempt used a command-line flag syntax not supported by the target container's minimal shell utility. The utility rejected the unrecognized flag and echoed the full invocation, including the credential value, into its error output. This was a tool-compatibility issue, not an unsafe command pattern in itself — the credential was being passed via a standard flag-based authentication method that would have worked against a fuller-featured version of the same tool.

**Credential path divergence.** Two separate secrets-manager entries existed for the same monitoring dashboard's admin credential, created independently by two unrelated pieces of prior work. One was created once and never touched again. The other was kept correctly current through 07-02 entirely through manual command-line rotation followed by a manual push, performed directly by a person — a fetch script existed that referenced the same path, but had never actually executed even once (confirmed by the target container's restart timestamp having never moved since creation, which the script's own final step would have changed had it ever run). The script's dormancy is a real, separate finding, but it deserves no credit for that location having stayed correct; the manual process is what did. When investigating why the secrets-manager copy of the credential didn't match its live value after tonight's incident-driven rotation, the abandoned location was checked first, found stale, and initially — incorrectly — assumed to be the sole canonical location; this led to a wrong conclusion that a prior rotation attempt had silently failed. On being asked to retrace the claim precisely, the properly-maintained location was found, its version history cross-referenced against log timestamps to confirm it actually had been correctly maintained up to the point of tonight's incident, and the true cause was identified: tonight's own credential rotation changed the live value without updating the correctly-maintained secrets-manager entry, because investigation had been anchored to the wrong one from the start.

---

## Remediation

**Monitoring gap:** confirmed as a monitoring-definition issue rather than a live alerting risk; no current consumer needs correction, but documented so a future dashboard or alert built against the flag doesn't inherit the same false-positive behavior. Recommended future consumers key off per-stack version comparison, matching the one existing correct implementation.

**Credential exposure:** the exposed credential was rotated immediately following explicit authorization. Verified live against the actual service, not assumed from a successful-looking command.

**Credential path divergence:**
- The properly-maintained secrets-manager location was updated with the new value and verified by reading it back and hash-comparing against the locally-stored value — not by trusting a successful write response.
- The permission grant required to complete that write (the acting identity previously lacked write access to this path) was made using a broader project-wide role than strictly necessary for the one path involved, in order to unblock the fix promptly. The full scope of that grant was documented in full for review, since it is metadata about access levels rather than a secret itself.
- The abandoned secrets-manager location and the dormant, never-executed fetch script that happened to reference it were retired together: confirmed no live automation, cron, or other script still wired to either before removing them (historical prose in existing documentation still mentions both, left as-is since it is dead documentation rather than live wiring and retirement does not require rewriting history), archived the script (preserving its history) rather than deleting it outright, and deleted the stale secrets-manager entry, confirming its removal by behavior (a subsequent lookup returning "not found") rather than trusting a deletion response.
- Final state confirmed: a single secrets-manager location remains for this credential, verified to match the live value by hash comparison.

Throughout the exposure response and the path reconciliation, no credential value was displayed at any point — verification relied exclusively on hash comparisons, status codes, and behavioral checks (e.g., a lookup returning "not found" as proof of deletion).

---

## Prevention

- A standing rule was reinforced: when a claim of "verified" or a causal explanation is challenged, the correct response is to retrace the actual evidence from scratch rather than defend the original conclusion — this produced the correct root cause on the second pass, after the first pass's inference (though explicitly hedged as an inference rather than a fact) still turned out to be wrong.
- A standing rule was added: before passing a credential to any command-line tool via flag syntax, confirm the tool's actual supported syntax when there is any ambiguity (e.g., a minimal container shell utility versus its full-featured counterpart), since an unrecognized-flag error can echo the full invocation — including the credential — into output. Prefer methods that keep credentials out of any string that could appear in error text.
- Open follow-up: the permission grant made to unblock tonight's fix is broader (full read/write across the secrets-management project) than the single path it was needed for. This should be reviewed and narrowed to a scope limited to the specific path involved, once a suitable custom role can be defined, rather than left at its current breadth indefinitely.
- Open follow-up (informational only, not urgent): two verification gaps remain — the token/credential scopes currently available do not permit direct inspection of the ingress edge's rule ordering or the source-control platform's own webhook delivery history. Current behavioral evidence strongly supports the chain working correctly, but a future incident requiring deeper inspection of either would need broader-scoped credentials at that time.

---

## Lessons Learned

**A monitoring signal's accuracy and its usefulness are different properties, and only checking the first one leaves a trap for later.** The failing success flag was technically accurate — not every target did succeed — but it had been failing identically for days for a reason completely disconnected from the property anyone actually cares about (whether the 22 stacks that matter are current). A signal that is correct but insensitive to the distinction that matters is worse than an honest gap, because it looks like coverage exists until the day someone builds real alerting on it and discovers it's been crying wolf the whole time.

**Hedging a conclusion correctly does not make it correct, and the fix for that is checking every available source, not softer language.** The claim that a prior rotation had "silently failed" was deliberately framed as an inference rather than a fact — appropriately cautious — but it was still wrong, because the check stopped at the first plausible-looking piece of evidence instead of cross-referencing every location a canonical answer could live (in this case, documented history that was sitting in the same repository the whole time). The lesson isn't "hedge more" — hedging was already correctly applied. It's that hedged language is not a substitute for exhausting the available evidence before reporting a conclusion, even a cautious one.

**Fixing an incident can itself become the next incident's cause, and the discipline that catches this is retracing causality after the fact, not just after being challenged.** This session's own rotation — intended purely to close out a credential exposure — is what actually broke the secrets-manager synchronization it was later asked to investigate. The error was caught only because the original explanation was challenged and retraced from first evidence rather than accepted; a workflow that treats "verified" as a checkpoint to revisit under scrutiny, rather than a status that closes discussion once claimed, is what surfaces this class of self-inflicted error before it compounds further.

---

*Environment: KONYKS-SERVER (Unraid) · Komodo/Periphery GitOps · Infisical secrets management · monitoring/observability stack · homelab-incident-reports*
