# Komodo Silently Dropped a Queued Deploy After Periphery Restart

**Date:** 2026-06-20
**Severity:** P2 Medium
**Status:** Resolved — known limitation, mitigated via external monitoring
**Affected:** Komodo Core, Komodo Periphery, observability stack (Loki)
**Duration:** ~9 hours between push and discovery (commit pushed mid-morning, gap found and worked around same afternoon)

---

## Summary

A git push containing a new bind-mount for the observability stack's Loki container was confirmed pulled into Periphery's working copy within minutes, matching the GitOps engine's normal poll-and-deploy behavior observed earlier the same day. However, a verification check roughly 50 minutes later found the target container had *not* been recreated — it was still running the pre-push configuration. The engine's own dashboard showed the stack as healthy and up to date, masking the gap. The container was brought up to date with a manually-scoped Compose command, restoring correct state.

Root cause was confirmed the same day via a controlled test (see Update section below): Komodo's auto-update mechanism is delta-only and does not self-heal. A failed deploy is permanently dropped until a new commit triggers a fresh cycle.

---

## Timeline

| Time (approx) | Event |
|------|-------|
| Morning | Git push lands a new bind-mount for Loki's alerting rules directory |
| +3 min | Periphery's working copy confirmed at the new commit (git log matches) |
| +50 min | Verification check finds the Loki container still running pre-push config — never recreated |
| Same session | Periphery logs show: `type: ComposePull \| error: Stopped after repo pull failure` at 03:14 UTC — the failed deploy, logged as a WARN |
| Same session | Manual fix applied: `docker compose up -d` scoped to the single affected container |
| +1 min | Fix verified — new container creation timestamp, mount present |

---

## Root Cause

**Confirmed (2026-06-20 follow-up test):**

Komodo's `auto_update` is **delta-only**: it triggers a `ComposePull`/`DeployStack` only when `latest_hash` changes (new commit detected on remote). It never re-deploys based on a mismatch between `deployed_hash` and `latest_hash`. When a deploy fails — whether due to a git pull error, a Periphery restart interruption, or a Core API hang — the `deployed_hash` stays at its old value permanently, the Komodo dashboard shows "running" with no error indicator, and the gap persists indefinitely until either a new commit arrives or a manual redeploy is triggered.

**Confirmed evidence from original incident:**
- Periphery WARN at 03:14:58 UTC: `type: ComposePull | error: Stopped after repo pull failure`
- After this WARN: zero subsequent deploy activity logged in Periphery for the remainder of the day
- No retry was ever attempted

**What the original failure was NOT:** This was not a "queued deploy being dropped when Periphery restarted." The Periphery restart and the failed deploy were separate events. The failure was a `ComposePull` that stopped at the git-pull stage with an explicit error. Komodo's response to that error was: log a WARN, move on, never retry.

---

## Remediation

**Immediate (2026-06-20 morning):** Manually re-issued the deploy for the single affected container, scoped narrowly to avoid disturbing the other seven containers in the same stack:

```bash
docker compose -f <stack-compose-path> up -d obs-loki
```

Verified fix: new container creation timestamp, new bind-mount confirmed, dependent service confirmed reading the new path.

**Structural (2026-06-20 afternoon):** See Mitigation section below.

---

## Update — 2026-06-20 Controlled Test

To definitively answer whether Komodo self-heals after an interrupted deploy, a controlled test was run the same day.

**Setup:** Added a trivial label change to the observability stack compose file (commit `4d83a40`), pushed to GitHub, triggered `DeployStack` via the Komodo API, and observed behavior across multiple poll cycles.

**What happened:**

| Time (UTC) | Event | Evidence |
|---|---|---|
| 19:47:25Z | `DeployStack` triggered for commit `4d83a40` | curl connected to Core, sent headers |
| 19:47:25Z–20:05:58Z | Core's entire HTTP API blocked for **18 minutes** | New read requests hung; Periphery logged zero activity — deploy was silently swallowed |
| 20:05:58Z | Periphery restarted manually | Core API immediately responsive |
| 20:07–20:19Z | 2 full poll cycles observed | 6 state snapshots; `deployed_hash` = null throughout; Periphery: zero new log entries; obs-loki container start time unchanged |
| 20:19Z | Watch complete | `deployed_hash: null`, `latest_hash: 4d83a40`, obs-loki unchanged |

**Verdict:** Komodo does not retry. Across two full 5-minute poll cycles with `deployed_hash=null` and `latest_hash=4d83a40`, Core never re-triggered a deploy. The gap persisted indefinitely.

**Mechanism:** Core's poll fetches the remote `latest_hash`. If it matches the last fetched value (even if `deployed_hash` differs), Core treats it as "nothing new" and does not fire `auto_update`. The condition `deployed_hash ≠ latest_hash` is never evaluated as a trigger.

**Additional finding — Core API freeze:** The `POST /execute` (DeployStack) endpoint is synchronous — Core holds its HTTP response open until Periphery completes. A stuck deploy freezes the entire Komodo HTTP server (UI and all API endpoints) for all requests. Confirmed at 18 minutes on 2026-06-20; no known ceiling. Fix: `docker restart komodo-periphery`, which unblocks Core in under 1 second.

**Second test (same session):** After Periphery was restarted and the cleanup commit pushed, a manual `DeployStack` completed in under 5 seconds with `success: true`. The 18-minute hang was a pathological stuck state, not the normal synchronous behavior.

---

## Mitigation

**1. Fixed the misleading alert description:**
The existing `KomodoRepoPullFailure` Loki ruler rule (added earlier same day as a reactive measure) incorrectly stated "Komodo retries automatically on the next 5-minute poll." Description corrected to document the actual behavior: no retry, manual intervention required.

**Coverage note:** `KomodoRepoPullFailure` only fires when Periphery logs a `"repo pull failure"` line. It does not cover the "Core API freeze" failure mode (where Periphery logs nothing). Both modes result in a deploy drop.

**2. Added `KomodoDeployLag` alert (primary fix):**
A new cron script (`komodo-deploy-lag`, Unraid User Script, runs every 5 minutes) queries the Komodo API, tracks each Stack's `deployed_hash` vs `latest_hash`, and fires a **HIGH** (`severity: warning, tier: T1`) alert via Alertmanager → HA after the mismatch has persisted for 10 minutes. This alert fires regardless of which failure mode caused the drop — git pull error, Core freeze, Periphery restart, or any other cause. Alert auto-resolves when hashes align.

**3. Documented the manual unstick procedure:**
Added to `homelab-infra/SYSOPS.md`:
- No-op commit (`git commit --allow-empty`) to trigger auto-update without a real change
- Manual API deploy call
- Core API freeze symptom and fix (`docker restart komodo-periphery`)
- Hash verification command to confirm a deploy landed

---

## Prevention

- [x] Root-cause the Periphery restart: confirmed separate event from the deploy failure; restart cause remains unknown but not load-bearing for this incident
- [x] Determine retry behavior: **confirmed no retry** — Komodo's auto_update is delta-only
- [x] Alerting: `KomodoDeployLag` cron fires when `deployed_hash ≠ latest_hash` for >10 min
- [x] Recovery documented in SYSOPS.md
- [ ] **Open gap — Komodo Core uptime not monitored:** `KomodoDeployLag` exits quietly when the API is unreachable (correct for transient blips, wrong for an 18-minute freeze). During today's Core API freeze, the lag cron silently skipped every cycle — zero monitoring coverage for the worse failure mode. Neither UptimeKuma (24 monitors, none for Komodo) nor Blackbox (5 external endpoints, none Komodo) would have fired. Fix: add `https://komodo.konyks.biz` to Blackbox probes with a short timeout (≤10s), alerting `severity: warning` if Core stops responding. Not built today to avoid scope creep, but "Resolved — mitigated via external monitoring" covers the deploy-drop case only, not the freeze case.
- [ ] Consider opening a Komodo upstream issue for delta-vs-reconciliation behavior

---

## Lessons Learned

1. **A green dashboard reflects the last successful action, not the most recently intended one.** The deploy engine's UI showed the stack as up to date because it was reporting against the last deploy it actually completed. Any GitOps tool's health view needs to be read with this distinction in mind.

2. **"It pulled the commit" and "it deployed the commit" are two separate guarantees.** Confirming the git working copy was at the right commit felt like sufficient evidence — it wasn't. The only thing that caught the gap was checking the running container's actual creation timestamp.

3. **Restarts that aren't actively investigated accumulate as unexplained variance.** Three separate restarts surfaced across this single day's work; the one that wasn't traced was the one that coincided with an actual functional gap.

4. **Log-based alerting is incomplete for GitOps failures.** The `KomodoRepoPullFailure` log rule would not have fired on today's incident (Alloy was offline when the failure occurred). State-based polling (`deployed_hash` vs `latest_hash`) is more reliable than log-scraping for this failure mode.

---

*Environment: KONYKS-SERVER (Unraid) · GitOps deployment engine (Core + Periphery) · Loki/Prometheus/Grafana/Alertmanager observability stack · homelab-incident-reports*
