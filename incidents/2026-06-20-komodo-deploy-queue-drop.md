# Komodo Silently Dropped a Queued Deploy After Periphery Restart

**Date:** 2026-06-20
**Severity:** P2 Medium
**Status:** Partially Resolved — symptom fixed, root cause unconfirmed
**Affected:** Komodo Core, Komodo Periphery, observability stack (Loki)
**Duration:** ~9 hours between push and discovery (commit pushed mid-morning, gap found and worked around same afternoon)

---

## Summary

A git push containing a new bind-mount for the observability stack's Loki container was confirmed pulled into Periphery's working copy within minutes, matching the GitOps engine's normal poll-and-deploy behavior observed earlier the same day. However, a verification check roughly 50 minutes later found the target container had *not* been recreated — it was still running the pre-push configuration. The engine's own dashboard showed the stack as healthy and up to date, masking the gap. The container was brought up to date with a manually-scoped Compose command, restoring correct state, but the underlying reason the queued deployment never executed remains unconfirmed.

---

## Timeline

| Time (approx) | Event |
|------|-------|
| Morning | Git push lands a new bind-mount for Loki's alerting rules directory |
| +3 min | Periphery's working copy confirmed at the new commit (git log matches) |
| +50 min | Verification check finds the Loki container still running pre-push config — never recreated |
| Same session | Periphery's own logs show only three events all day: a startup, one unrelated transient pull failure six hours earlier, and one restart roughly 90 minutes *before* the push — then total silence afterward, with no deploy activity logged for any commit since |
| Same session | Manual fix applied: `docker compose up -d` scoped to the single affected container |
| +1 min | Fix verified — new container creation timestamp, mount present, dependent service (Loki's alert rule loader) successfully reading the new path |

---

## Root Cause

**Confirmed:**
- The git working copy for the affected stack was at the correct, up-to-date commit.
- The compose file on disk correctly contained the new mount definition.
- The running container had *not* been recreated to pick up that mount — a clear gap between "config is correct on disk" and "running state matches config."
- Periphery (the per-stack deployment agent) logged no deploy activity at all for either of two commits pushed that day, despite confirmably having pulled both.

**Unconfirmed hypothesis:** Periphery restarted roughly 90 minutes before the push landed. The leading theory is that the engine's core process queues a deploy action after detecting a git change, and if the corresponding Periphery instance is mid-restart or briefly unavailable when that action is dispatched, the queued action is dropped rather than retried — with the core process's own dashboard then showing the stack as healthy based on the *last successfully completed* deploy, not the *most recently intended* one.

This has not been independently verified. The restart's own cause is itself unconfirmed — it doesn't line up cleanly with any backup or scheduled job known to run at that hour, and no corresponding error or log entry from another component points to a clear trigger.

---

## Remediation

Manually re-issued the deploy for the single affected container, scoped narrowly to avoid disturbing the other seven containers in the same stack:

```bash
docker compose -f <stack-compose-path> up -d obs-loki
```

Verified the fix rather than assuming it from exit code alone:
- New container creation timestamp confirmed (old container fully replaced, not just restarted)
- New bind-mount confirmed present in the running container's mount list
- The dependent service's own internal state (its rule-loading subsystem) confirmed it could now read files at the new mount path, where it had previously logged a "no such file or directory" error
- Confirmed zero collateral impact — all seven other containers in the same stack retained their original, untouched uptimes

---

## Prevention

- **Open follow-up — not yet done:** root-cause the Periphery restart itself. The working theory (interrupted mid-deploy-dispatch) is unverified and the restart's trigger is unidentified. Until this is understood, there's no way to know if or when it will recur.
- **Open follow-up — not yet done:** determine whether the deploy engine has any retry or reconciliation behavior for a dropped/queued action, or whether a missed dispatch is permanently lost until the next unrelated change triggers a fresh deploy cycle. This determines whether the failure mode is "rare and self-healing" or "silent and sticky."
- **Mitigating control already in place from a related incident the same day:** an alerting rule was added specifically to surface log lines containing deploy-pull failures from the deployment agent, which would have caught *part* of this chain (the pull side) but not the silent-drop-after-successful-pull behavior actually observed here. Worth noting as a partial, not full, safety net for this exact failure mode.

---

## Lessons Learned

1. **A green dashboard reflects the last successful action, not the most recently intended one.** The deploy engine's own UI showed the stack as up to date because it was reporting against the last deploy it actually completed — with no visible distinction between "nothing has changed since" and "something changed and didn't get applied." Any GitOps tool's health view needs to be read with this distinction in mind, not taken as proof of current-state-matches-desired-state.

2. **"It pulled the commit" and "it deployed the commit" are two separate guarantees, and only one of them was actually being verified.** Confirming the git working copy was up to date felt like sufficient evidence of a successful rollout — it wasn't. The only thing that caught the gap was checking the *running container's* actual creation timestamp, several steps further down the chain than where verification initially stopped.

3. **Restarts that aren't actively investigated accumulate as unexplained variance, and unexplained variance is where silent failures hide.** Three separate restarts surfaced across this single day's work, two of which were eventually traced to known, intentional causes — this third one wasn't, and it's the one that turned out to coincide with an actual functional gap. Treating "the system came back up fine" as equivalent to "nothing here needs investigating" would have let this pass unnoticed.

---

*Environment: KONYKS-SERVER (Unraid) · GitOps deployment engine (Core + Periphery) · Loki/Prometheus/Grafana/Alertmanager observability stack · homelab-incident-reports*
