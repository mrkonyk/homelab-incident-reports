# CrowdSec LAPI Key Exposure — Detection, Rotation, and Incomplete-Purge Discovery

**Date:** 2026-06-26
**Severity:** P0 Critical
**Status:** Resolved
**Affected:** SWAG, CrowdSec, Authelia (audited alongside), `homelab-infra` git repository, 25 Komodo-managed stack checkouts
**Duration:** Single session, discovery to close — approximately 4 hours

---

## Summary

A proactive security audit of the SWAG/Authelia/CrowdSec edge auth stack discovered a CrowdSec LAPI key committed as plaintext in a stack's Docker Compose file, tracked in git since the initial GitOps migration five days prior. The key was rotated immediately, but closing the exposure fully required three escalating discoveries: the planned remediation tool (SSH deploy keys) turned out to be architecturally unsupported by the GitOps orchestrator; the planned history-rewrite tool (`git filter-repo`) was unavailable in the environment and had to be substituted; and the substitute tool's rewrite, though locally clean, left two GitHub branches with the original history still live on the remote — caught only by comparing local rewritten state against the actual remote, not by trusting the local verification alone. A residual copy of the key remains reachable through two GitHub pull-request refs that cannot be removed via standard git operations; this was assessed and explicitly accepted as a documented, near-zero-risk residual rather than escalated to a platform support process.

---

## Timeline

| Time (relative) | Event |
|------|-------|
| T+0 | Read-only audit of SWAG/Authelia/CrowdSec begins; no findings expected to be critical |
| T+~1h | Audit completes; CrowdSec LAPI key found committed as plaintext in a stack's Compose file (5 days prior) |
| T+~1h15 | Key rotated, old bouncer registration deleted — live exposure closed |
| T+~1h30 | Scoping reveals the rotation gap is systemic: 3 generations of a separate access token are embedded in `.git/config` across all 25 stack checkouts; 2 of 3 are already dead, meaning 24/25 stacks silently cannot pull from git |
| T+~2h | SSH deploy keys investigated as the permanent fix; confirmed infeasible — the orchestrator's git-provider model has no SSH transport |
| T+~2h15 | Fallback remediation applied: stale tokens swept to the current valid one across all checkouts; sweep logic added to the rotation script so the gap can't recur |
| T+~2h30 | Git history purge attempted with the planned tool; tool unavailable in the environment, substituted with an alternative mid-execution |
| T+~3h | Full-history verification finds the substitute tool's rewrite is locally clean |
| T+~3h15 | A second verification pass (checking remote state, not just local refs) finds two branches on GitHub still carry the original, unrewritten history |
| T+~3h30 | Main branch force-pushed clean; the two stale branches deleted from the remote |
| T+~3h45 | A fresh-clone test (simulating what any new clone would actually receive) finds the key is still reachable via two GitHub pull-request refs — a ref type that persists independently of branch deletion |
| T+~4h | Residual risk assessed and accepted (key is dead, repo is private); documented in ops doctrine; local cleanup and stack-checkout resets completed; closed |

---

## Root Cause

A CrowdSec LAPI key was written directly into a stack's `docker-compose.yaml` as a plaintext environment value at the time that stack was first migrated into the GitOps-managed structure, rather than being externalized to a secrets-management layer that had already been built and was in active use for other credentials. The stack predated full secrets-onboarding coverage, and onboarding wasn't yet complete across every stack at the time of migration.

A second, independent root cause was found during remediation scoping: the GitOps orchestrator's Stack-resource git integration writes an authenticated HTTPS URL — including the access token — directly into each stack checkout's local `.git/config` at deploy time. When the token is rotated at the source, this embedded copy is never refreshed; only a full re-clone rewrites it, and routine cache-refresh operations do not trigger a re-clone. This meant that two prior token rotations had already silently broken git connectivity for the majority of managed stacks, with no alerting on the failure because most stacks don't require a git-triggered redeploy often enough for the breakage to be noticed.

---

## Remediation

**Immediate — key rotation:**
1. New CrowdSec LAPI key generated; new bouncer registration confirmed pulling decisions successfully before the old one was touched.
2. Old bouncer registration deleted, invalidating the exposed key regardless of where it still appeared in git history.
3. Key externalized from the Compose file to an `env_file`-sourced location outside git tracking.

**Systemic — dead-token sweep across stack checkouts:**
1. Scoping found three generations of a separate, unrelated access token embedded in `.git/config` across all 25 git-tracked stack checkouts; only the current generation was valid.
2. **Attempted fix — SSH deploy keys:** investigated as the architecturally clean solution (no token ever written to disk in plaintext). Confirmed infeasible: the orchestrator's git-provider configuration schema supports only username/token authentication; a related boolean flag controls plaintext-HTTP vs. HTTPS, not a transport switch to SSH; direct testing confirmed no SSH key material is provisioned anywhere in the relevant agent's environment. No newer release of the orchestrator changes this. This path was closed at the feasibility-check stage, before any production stack was touched — confirmed via source inspection and live testing, not assumed from documentation.
3. **Applied fix:** all stale token references swept to the current valid token across every checkout, verified by a successful test-fetch on all 25 stacks afterward.
4. The sweep logic was added as a permanent step in the rotation script itself, so this class of gap cannot recur on future rotations without being immediately self-corrected.

**History purge:**
1. The planned tool for rewriting git history (the modern, maintainer-recommended replacement for git's legacy history-rewrite command) required a runtime dependency unavailable in the execution environment. The legacy built-in command was substituted to complete the task.
2. The substitute tool's local rewrite was verified clean via a full-history search across every local ref.
3. **A second, independent verification step** — comparing local rewritten branch state against what the remote actually held — found two branches still present on the remote with the original, unrewritten commit history. The local rewrite had correctly processed local copies of these branches, but the rewrite was never pushed to update them on the remote; this was invisible to any check that only inspected local state.
4. The main branch was force-pushed with clean history; the two stale remote branches were deleted (confirmed no open work depended on them).
5. **A third verification step** — cloning the repository fresh, as any outside party or new collaborator would — found the original key value was still retrievable through two GitHub pull-request refs. This ref type is created automatically whenever a pull request is opened and is retained by the platform independently of the source branch's existence; it cannot be removed through standard git push or branch-deletion operations.
6. This residual was assessed: the key is confirmed dead (rotated before the purge began), the repository is private, and the ref type in question is accessible only to existing repository members. The exposure was accepted as a documented residual rather than escalated to a platform support process, with an explicit trigger condition recorded for revisiting that decision.

---

## Prevention

- **Rotation script updated** to sweep all stack checkout git configurations on every future rotation, closing the root cause rather than the symptom.
- **Secrets-externalization gap flagged** in ops doctrine: the affected stack's secret now lives outside git but is not yet under the same centralized secrets-management coverage used elsewhere in the environment — recorded as an open item rather than left implicit.
- **SSH-deploy-key infeasibility documented** with the specific evidence that closed it, so it isn't re-investigated from scratch if raised again, but is revisited if the orchestrator adds support in a future release.
- **PR-ref residual documented** in ops doctrine with the exact accepted-risk reasoning and an explicit condition for when that decision should be revisited (change in repository visibility, or a future compliance requirement for provable zero-reachability).
- **Open follow-up:** complete secrets-management onboarding for the affected stack, removing the last manually-managed credential of this kind in the environment.

---

## Lessons Learned

1. **Local verification of a history rewrite is necessary but not sufficient — remote state must be checked independently, not inferred.** A history-rewrite operation correctly processed every locally-rewritten ref, including remote-tracking copies of branches. But "locally rewritten" and "pushed" are different facts, and a check that only inspects local refs cannot distinguish between a branch that's clean because it was rewritten and one that's clean because the local copy simply hadn't been compared against the actual remote yet. The catch here came specifically from querying the remote directly rather than trusting the local repository's self-report — a pattern worth generalizing to any operation where a tool's local success state doesn't guarantee the remote matches.

2. **A platform's data-retention model can make "purge" an architecturally incomplete operation, independent of how correctly the git-level work is done.** Git history can be rewritten, force-pushed, and have stale branches deleted with full correctness, and the secret can still be retrievable — because the hosting platform retains certain reference types (here, pull-request refs) outside the reach of standard git operations entirely. The operational lesson isn't "do the rewrite more carefully" — the rewrite was correct. It's that a credential exposure's blast radius needs to be evaluated against the *platform's* retention behavior, not just git's, before declaring a purge complete.

3. **A rotation process that updates the credential at its source but doesn't verify propagation to every consumer creates a silent failure mode that compounds across rotation cycles.** The root credential here was rotated correctly each time; the gap was that one class of consumer (locally-checked-out git remotes) never received the update, and nothing alerted on the resulting broken state because the failure only manifests when that specific consumer is exercised. By the time this was found, two full rotation cycles had silently broken connectivity for the majority of managed services. The fix — folding the propagation check into the rotation script itself, so it runs automatically every time rather than relying on a separate audit to catch drift — converts a one-time remediation into a structural guarantee.

---

*Environment: KONYKS-SERVER (Unraid) · SWAG · Authelia · CrowdSec · Komodo/Periphery GitOps · homelab-incident-reports*
