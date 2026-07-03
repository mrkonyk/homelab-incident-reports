# KomodoDeployLag on observability/seerr — A PAT-Rotation Race, Not a Repeat of Prior Deploy-Drop Bugs

**Date:** 2026-07-03
**Severity:** P2 Medium
**Status:** Resolved
**Affected:** observability stack (Grafana, Loki, Prometheus, Alertmanager, cAdvisor, Alloy), seerr stack, `do_push_rotation.sh`, `deploy-all-lagging` Komodo Procedure
**Duration:** ~20 hours from first failed deploy (2026-07-02 19:19:56 UTC) to independently-verified closure (2026-07-03, live fault-injection test)

---

## Summary

A `KomodoDeployLag` HIGH alert fired for the `observability` stack, reporting `deployed_hash`/`latest_hash` mismatch for 880+ minutes with `auto_update=true`. Because this alert exists specifically to catch two previously-documented failure modes — a Cloudflare edge block on the GitHub webhook (found and fixed 2026-06-29/07-01) and a silent Komodo poll-based deploy drop with no logged error (2026-06-20, unresolved) — the first task was determining which of those two this was, or whether it was something new.

It was something new. Direct evidence from Komodo Core's own logs showed the GitHub webhook fired, was authenticated, and dispatched the `deploy-all-lagging` batch procedure within the same second the triggering commit landed — ruling out a Cloudflare regression. The batch itself then failed loudly for exactly two of 24 stacks (`observability`, `seerr`) with an explicit `HTTP 401 / Invalid username or token` git authentication error — a fully logged, diagnosable failure, not the silent no-op that characterized the June 20 incident.

Root cause: a race between `do_push_rotation.sh`'s PAT propagation and Komodo Periphery's own independent git activity in the same window. Two stack checkouts held a stale token at the exact moment a webhook-triggered deploy batch ran against them, ~38 minutes after a PAT rotation had completed elsewhere. No retry mechanism existed to catch the drop afterward, so both stacks stayed stuck until manually redeployed.

Fixed in two parts: an immediate manual redeploy of both stacks (with functional verification, not just "container is up"), and a new step 5 in `do_push_rotation.sh` that hash-compares every stack checkout's token against the freshly-rotated canonical value after the existing sweep and redeploys any stack still found stale. The new step 5 was verified three ways: a passive check that the fix held under normal webhook traffic, a real end-to-end PAT rotation with independent hash-verification of all 25 checkouts, and a live fault-injection test that staled one real stack's token mid-rotation to confirm the detection-and-fix path actually fires under a genuine race, not just in isolated scratch fixtures.

---

## Timeline

| Time | Event |
|------|-------|
| 2026-07-02 ~18:41 EDT | `periphery.config.toml` modified — a routine PAT rotation via `do_push_rotation.sh` completes elsewhere in the fleet |
| 2026-07-02 19:19:47 EDT | Commit `569ce10` pushed to `homelab-infra` (unrelated hermes-agent bind-mount change) |
| 2026-07-02 19:19:56 EDT (23:19:56 UTC) | GitHub webhook fires from `140.82.115.242`; Komodo Core logs `Successfully authenticated incoming webhook`; `deploy-all-lagging` Procedure batch dispatches `DeployStack` to 24 stacks in the same second |
| same instant | 22/24 stacks deploy successfully. `observability` and `seerr` fail: `git fetch`/`git pull` return `HTTP 401` / `Invalid username or token. Password authentication is not supported for Git operations.`; both leave `compose.yaml` missing post-failure |
| 2026-07-02 19:19:56 EDT → 2026-07-03 ~11:00 EDT | No retry attempted by Komodo (confirmed via `ListUpdates` — no update record for either stack between the failure and manual intervention). `deployed_hash` stuck at `5c7461d` for both while `latest_hash` continues advancing with unrelated commits |
| 2026-07-03 (session start) | `KomodoDeployLag` HIGH alert investigated. Confirmed via Komodo Core logs: webhook fired + authenticated, Procedure batch ran immediately — ruling out a Cloudflare regression of the 2026-06-29/07-01 fix. Confirmed via `GetUpdate` logs on the two failed stacks: explicit, fully-logged 401 auth errors — ruling out a repeat of the 2026-06-20 silent poll-drop |
| 2026-07-03 | Root cause identified: both stacks' `.git/config` tokens hash-matched canonical *at investigation time*, meaning something had already re-synced them after the failure — but nobody had re-triggered a deploy since. `periphery.config.toml` mtime (18:41) predating the failure (19:19:56) by ~38 minutes pointed to a rotation-race window |
| 2026-07-03 | Manual recovery: `DeployStack` triggered for `observability` and `seerr`; both converged `deployed_hash == latest_hash`; functional checks performed (see Remediation) |
| 2026-07-03 | Fix proposed and implemented: new step 5 in `do_push_rotation.sh`, committed `6b33b7f`, pushed to `main` |
| 2026-07-03 | Passive verification: the `scripts/hermes/` push itself triggered a clean 24-stack batch; `observability`/`seerr` converged to `6b33b7f` with zero drift and zero unexpected container recreation |
| 2026-07-03 | Live rotation test: a genuinely new PAT rotated end-to-end via the (now 6-step) script; independently hash-verified all 25 stack checkouts against canonical; confirmed via Komodo Core logs and `ListUpdates` that zero deploy/webhook activity occurred during the 11-second rotation window — a valid clean pass, but not proof the detection path fires under real contention |
| 2026-07-03 | Live fault-injection test: scratch copy of the script with a pause inserted between step 4 and step 5; real rotation run; `wallos`'s `.git/config` deliberately staled with a synthetic token during the pause to simulate the race; step 5 detected the mismatch, corrected it, and redeployed `wallos` — independently re-verified by hash and by `GetStack`/container health, with a full-fleet sweep confirming no collateral impact |

---

## Root Cause

**The failure was a PAT-rotation race, not a webhook or polling defect.**

`do_push_rotation.sh` propagates a new GitHub PAT to five targets in sequence: `periphery.config.toml`, Komodo's MongoDB `GitProviderAccount` record, the main repo checkout's remote, and a sweep of all 25 Komodo-managed stack checkouts' `.git/config` files. This propagation is not atomic across targets, and Komodo Periphery independently performs its own git operations (auto-update polling, webhook-triggered deploys) on its own schedule, using whatever token value happens to be current in `GitProviderAccount`/`.git/config` at the moment it acts.

On 2026-07-02, a rotation completed around 18:41 EDT. A webhook-triggered deploy batch ran roughly 38 minutes later, at 19:19:56 EDT. In that window, 22 of 24 pattern-matched stacks already held the new token (most likely because the rotation script's sweep had already reached them, or Periphery had independently re-synced them correctly). Two — `observability` and `seerr` — did not, and failed the batch with an explicit git authentication error. The most likely mechanism (consistent with a previously-documented gap, see Prevention) is that Periphery re-cloned or reset these two stacks' remotes using a token value from a point in time that predated the rotation's completion, and the rotation script's own sweep either hadn't reached them yet or was itself overwritten by Periphery's action shortly after.

Once the two stacks failed, nothing re-tried them. Komodo's `auto_update` mechanism governs Docker image-digest drift, not git-commit drift; the 5-minute resource poll only refreshes `latest_hash` for display and never triggers a deploy. The only paths that redeploy a stack are a manual `DeployStack` call or a webhook hitting the Procedure's listener URL — and a webhook batch only re-attempts stacks whose `latest_hash` has moved again, which does nothing for a stack that's already failed and whose only problem is a stale credential, not a stale commit. The two stacks were stuck indefinitely until someone manually intervened.

### What this was NOT

- **Not a regression of the 2026-06-29/07-01 Cloudflare fix.** That incident (documented in `SYSOPS.md`, "Komodo webhook auto-deploy — root cause chain") found GitHub's webhook deliveries were being blocked by a dormant "Only Canada" geo-restriction custom rule predating the effort by years, fixed with an ordered geo-bypass + WAF-skip rule pair for the webhook path. This time, Komodo Core's own log shows `Successfully authenticated incoming webhook` at the same second the triggering commit landed, from a genuine GitHub Hookshot IP (`140.82.115.242`). The webhook chain worked correctly end-to-end; the failure was entirely downstream of Cloudflare, inside Komodo's own deploy execution.

- **Not a recurrence of the 2026-06-20 silent poll-drop** (`2026-06-20-komodo-deploy-queue-drop.md`). That incident's defining characteristic was silence: Periphery confirmed the pull, the dashboard showed the stack as healthy, and no error was ever logged — the gap was only found by checking the running container's actual creation timestamp. This incident's failure was loud and immediately diagnosable: `GetUpdate` on the failed stacks showed a stage-by-stage log culminating in an explicit `HTTP 401` / `Invalid username or token` error and a "file doesn't exist after writing stack" validation failure. Nothing about this failure required inference — it was captured, structured, and attributable to a specific git-auth failure the moment anyone looked.

This is a third, previously-undocumented failure mode specific to the interaction between credential rotation and Komodo's git-based deploy mechanism.

---

## Remediation

### Immediate: manual redeploy with functional verification

`DeployStack` was triggered manually for both `observability` and `seerr` once their `.git/config` tokens were confirmed (via `sha256` hash comparison, not visual inspection) to already match the current canonical PAT. Both completed successfully and converged `deployed_hash == latest_hash`.

Verification went beyond "container is up":
- **Grafana:** `GET /api/health` → `200`, `{"database":"ok","version":"13.0.2"}`
- **Loki ruler API:** `GET /loki/api/v1/rules` (direct, bypassing the auth proxy to isolate Loki itself) → `200`
- **Loki auth proxy:** `GET /ready` through the nginx sidecar with no credentials → `401` — the *correct* response, confirming the basic-auth gate added 2026-07-02 was still enforcing
- **Seerr:** `GET /` → `307` redirect to `/login` → follow redirect → `200`; container `StartedAt` unchanged, confirming no unnecessary recreation

### Structural: `do_push_rotation.sh` step 5

Added a post-sweep verification pass that closes the race rather than trying to prevent it (no reliable way exists to detect "a deploy is about to land" from the rotation script's side, since Komodo exposes no queryable lock/queue state — see Prevention). After the existing checkout sweep:

1. Re-derive the new token's `sha256` and compare it against every stack checkout's current token hash.
2. For any mismatch, re-apply the correct token to that stack's `.git/config`.
3. Trigger `DeployStack` for the corrected stack and poll `GetUpdate` (up to 20s) for `status=Complete` + `success=true` before moving on.
4. `mariadb`/`redis` are excluded from the auto-redeploy action (token still gets fixed) — consistent with their existing exclusion from unattended auto-deploy in the `deploy-all-lagging` Procedure's DB-tier policy.
5. Print a summary: stacks checked, stale, redeployed, skipped, and needing manual follow-up.

Before touching the real script, the detection-and-fix logic and the DB-tier exclusion were both tested in isolation against scratch fixtures with fake tokens — confirming correct mismatch detection, correct selective fix (leaving already-current tokens untouched), and correct DB-tier skip behavior — prior to any real credential handling.

Committed as `6b33b7f`, pushed to `main`.

---

## Verification

Three separate passes were run before considering this closed, specifically because a fix that "looks right" and a fix that's been shown to catch a real fault under real conditions are different claims.

**Pass 1 — Passive check.** The `do_push_rotation.sh` commit itself triggered a real webhook batch. `observability` and `seerr` converged cleanly to the new commit hash with no drift. Cross-checked the full 24-stack batch: 23 succeeded, the one failure (`music-assistant`) was the pre-existing, already-documented container-name-conflict issue, unrelated to this fix. The one container recreation observed (`Hermes-Agent`) was traced to an unrelated upstream `:latest` image digest change, not to anything in this commit.

**Pass 2 — Live rotation, independently audited.** Confirmed no new PAT was already staged in Infisical before proceeding (checked by hash comparison — avoided assuming). Once a genuinely new PAT was staged, ran the real (unmodified) script end-to-end. Rather than trusting the script's own "25/25 updated, 0 stale" report, independently re-extracted and hashed every stack checkout's token myself and confirmed a 100% match against a freshly re-extracted canonical value. Checked Komodo Core logs and `ListUpdates` for the exact rotation window and found zero concurrent deploy/webhook activity — meaning this pass, while clean, had nothing to race against and is not on its own evidence that step 5's detection logic works under contention.

**Pass 3 — Live fault injection, the actual positive-detection proof.** A scratch copy of the script had a pause inserted between step 4 and step 5 (synchronized via a sentinel file rather than a guessed sleep duration, after a first attempt where the injection window was missed entirely due to checking the wrong output stream). Ran the real script; during the pause, deliberately overwrote `wallos`'s real `.git/config` with a synthetic stale token to simulate a Periphery re-clone landing in the gap. Step 5 correctly reported `MISMATCH: wallos still on a stale token post-sweep — re-applying` followed by `wallos: redeploy complete, success`. Independently verified: re-hashed `wallos`'s token (match), confirmed the synthetic value was gone, confirmed `deployed_hash == latest_hash`, confirmed the container was running and untouched (`StartedAt` unchanged, since only the credential was ever wrong), and swept all 25 checkouts to confirm zero collateral effect from the test. No token value appeared in any script output at any point across all three passes (checked programmatically, not just visually).

---

## Prevention / Follow-up

- [x] Immediate stacks recovered and functionally verified, not just container-up-checked
- [x] Rotation-race gap closed with a post-sweep verification-and-redeploy step
- [x] Fix verified against a genuine injected race, not only scratch fixtures
- [ ] **Open — no queryable lock/queue state from Komodo.** The alternative approach considered (pause rotation while a deploy is in flight) was rejected because Komodo exposes no API to query "is a deploy currently running or about to." `procedure_locks()` is an in-memory per-resource mutex with no external visibility (confirmed by source reading during the 2026-07-01 webhook investigation). Step 5's after-the-fact detection is a compensating control, not a true fix for the underlying non-atomicity. If Komodo ever exposes deploy/lock state via API, revisit whether a pre-rotation check is worth adding alongside step 5, not instead of it.
- [ ] **Open — revocation status of the PAT active during the race not tracked as part of this incident.** The rotation script's own trailing checklist already reminds the operator to revoke the prior token in GitHub; this incident didn't specifically confirm that happened for the token active during the race. Worth a spot-check next time PATs are audited.
- [ ] Consider whether other scripts that write credentials to multiple independent targets (Redis shared password, Immich Postgres, Grafana admin) have a similar non-atomic-propagation race against a service that reads its own copy on its own schedule. Not investigated here — flagged as a pattern worth checking, not a confirmed second instance.

---

## Lessons Learned

1. **"The alert fired" is not the same claim as "it's the bug we already know about."** This alert exists because of two prior incidents, and the investigation could easily have stopped at "probably the Cloudflare thing again" or "the June 20 poll-drop, still unresolved." Both were wrong. Checking Komodo Core's own logs directly — rather than reasoning from the alert's history — took minutes and immediately ruled out both prior explanations in favor of a third one nobody had looked for.

2. **A loud, fully-logged failure and a silent one are different bugs even when the symptom looks identical.** `deployed_hash != latest_hash` was the same alert condition in this incident, the June 20 incident, and the June 29 one. The actual failures — a rotation race with an explicit 401, a poll mechanism with no retry and no log line, and a Cloudflare rule blocking the request before it ever reached Komodo — are three unrelated mechanisms that happen to trip the same alert. Treating the alert as diagnostic rather than as a pointer to "go look" would have led to the wrong fix.

3. **A negative test result needs its timing checked before it counts as evidence.** The live rotation pass reported zero corrections needed — technically a "pass," but only meaningful once the timing was checked and showed zero concurrent deploy activity during that window. Without that check, a clean rotation could be misread as proof the fix works, when it only proves nothing went wrong when there was nothing to go wrong against. The fault-injection pass, not the clean rotation, is what actually demonstrates the fix.

4. **Trusting a script's own summary output is not the same as verifying its claim.** In both the live rotation and the fault-injection test, the script's own reported counts turned out to be accurate — but that was established by independently re-deriving the same numbers from the underlying state (`.git/config` file contents, `GetStack` hashes, container timestamps), not by reading the script's stdout. A script that lies about its own success is indistinguishable from one that doesn't, unless something outside the script checks.

---

*Environment: KONYKS-SERVER (Unraid) · Komodo Core/Periphery v2.2.0 · GitHub webhook → `deploy-all-lagging` Procedure · `do_push_rotation.sh` · Infisical secrets manager · observability stack (Grafana/Loki/Prometheus/Alertmanager) · Seerr · homelab-incident-reports*
