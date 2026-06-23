# GitHub PAT Rotation Script: Believed-Staged Rollout Was Actually a Single Unscoped Write Across 24 Stacks

**Date:** 2026-06-23
**Severity:** P2 Medium
**Status:** Resolved
**Affected:** `rotate-github-pat.sh` (credential propagation tooling), 24 Komodo-managed stack git checkouts, GitHub PAT Expiry Watch cron (Hermes-Agent)
**Duration:** Single session, approximately 4 hours from initial tooling validation through close-out

---

## Summary

A self-hosted Infisical instance and a new non-interactive rotation script (`rotate-github-pat.sh`) were deployed to replace a manual four-touchpoint GitHub PAT rotation process and two previously abandoned interactive scripts. During live validation of the script's stack-remote-update function, a staged rollout plan (a two-stack canary, followed by batches of the remaining stacks) was followed procedurally across several operator-and-agent exchanges. Post-hoc verification revealed the underlying function accepted no stack-scoping parameter at all: the first non-dry-run call had already rewritten the credential in every one of the 24 affected stack git checkouts in a single unconditional pass. Every subsequent "batch" was re-verifying an already-complete write, not incrementally applying one. No negative impact occurred — every git remote authenticated successfully throughout, including three security-critical stacks that were inadvertently included in an unplanned bulk run — but the operational assumption of staged, reversible rollout was false from the first real invocation onward.

---

## Timeline

| Time | Event |
|------|-------|
| Earlier session | Two prior interactive rotation scripts abandoned after repeated failures driving non-interactive execution |
| Session start | Infisical deployed as centralized secret store; `rotate-github-pat.sh` written as the non-interactive replacement |
| T+0 | First credential rotated and verified end-to-end: restart timing confirmed via container start time vs. config write time, no raw value in any output |
| T+~3min | Second credential rotated and verified by the same standard |
| T+~5min | Third credential's rotation aborts mid-script on a stale embedded credential found in one stack's git config; manually recovered, surfacing a broader gap |
| T+~10min | Audit finds the same stale-credential pattern in 24 of 25 stack git checkouts; one stack found clean via a different, injection-based credential mechanism |
| T+~15min | New function added to handle all affected stacks; dry-run pass confirms the pattern matches across all 24 |
| T+~20min | "Two-stack canary" executed — unknowingly the first unconditional global write |
| T+~30min | A 13-stack "batch" executed without an explicit stack list, deviating from a planned isolation of three security-critical stacks; all reported healthy with no service disruption |
| T+~40min | Final 9-stack "batch" executed; every stack reports as already holding the new value *before* that run executes |
| T+~45min | Discrepancy investigated; function's actual source confirms it has no scoping logic and loops unconditionally over every matching file on every call |
| Close-out | Documentation corrected to reflect actual (global, by-design) behavior; stale references in operational docs corrected |

---

## Root Cause

The stack-remote-update function was implemented as an unconditional loop over every matching git config file under the stacks directory, gated only by a content check (does this file's remote URL contain an embedded credential in the expected pattern) — not by any stack-name or list parameter. No version of the function ever supported partial application.

The staged-rollout plan existed entirely at the conversational/process layer — canary, then batch, then batch — and was never reflected in the code being executed. The first real (non-dry-run) invocation, framed as a small canary, silently completed the rewrite across every stack with an embedded credential in one pass. Every subsequent "batch" call repeated the same unconditional loop, found the values already correct, and reported a clean re-verification — which read identically to a genuine incremental success.

A compounding factor: partway through, the executing agent's session was restarted. An earlier instruction — explicitly isolate three security-critical stacks (the authentication and reverse-proxy path) for individual, last, closer-scrutiny handling — was given in one message but never reached the agent in a form it retained; a later, separate instruction arrived as an unscoped "go-ahead." The agent used its own judgment, ran all remaining stacks including the security-critical three in a single bundled pass, and did not flag the deviation from the original plan until asked directly afterward.

---

## Remediation

1. Verified actual end state directly rather than trusting the staged narrative: confirmed all 24 affected stacks' git remotes authenticated successfully against the new credential.
2. Independently verified the three security-critical containers had no restarts and remained healthy throughout — container start timestamps predated the bulk operation by roughly a day and a half, and direct health checks returned expected results.
3. Obtained the function's actual source rather than accepting a description of its behavior, confirming the no-scoping hypothesis directly.
4. Added an explicit code comment documenting the function's global, all-at-once behavior as intentional — no legitimate use case exists for applying this kind of credential update to only some affected stacks.
5. Corrected stale credential-identifier references left in operational documentation from before this rotation.

---

## Prevention

- A documentation comment was added directly above the function in question, stating plainly that it operates globally by design and that no partial-application path exists.
- The standing operational runbook was updated to mark the new Infisical-backed script as the canonical rotation process for all three GitHub credential slots involved, with explicit deprecation notices on the two predecessor scripts that were abandoned earlier.
- **Process change:** multi-stage execution plans relayed to an executing agent need to be self-contained within each handoff — an explicit, repeated scope every time — rather than relying on instructions accumulated earlier in a session. A session restart demonstrated that staging instructions can silently fail to carry forward even when everything appears to proceed normally afterward.
- A second, now-redundant verification function (checking only one hardcoded stack, duplicating checks the new function already performs per-stack) was identified and flagged for a future decision, left unmodified pending that decision.
- Two unrelated open items were logged rather than actioned in the moment, to avoid scope creep into unrelated investigations.

---

## Lessons Learned

1. **A staged rollout that exists only in conversation, not in code, provides none of the safety guarantees it appears to.** The dry-run pass beforehand was a genuine safety check; the canary-then-batch structure layered on top of a globally-scoped function was theater — it produced an identical end state to a single unscoped call, with more reporting overhead and a false sense of incremental control.
2. **Verified outcomes can be entirely true while the process claim producing them is false.** Every individual check (credential reachable, no disruption, no raw value exposed) passed honestly at every stage — and still described a process that never actually happened the way it was reported. A result that looks suspiciously too clean (every item in a batch already correct before the batch runs) is itself a signal worth chasing down, not a reason to relax scrutiny.
3. **Session boundaries are a real failure point for multi-step plans relayed through natural language to an executing agent**, independent of model capability. An explicit instruction to handle security-critical components with extra isolation did not survive a context reset; the agent gave an honest account of the gap once asked directly, but only because it was asked. Bare continuation instructions on multi-stage operational work should be treated as fragile across any session boundary, not just trusted to persist.

---

*Environment: KONYKS-SERVER (Unraid) · Komodo/Periphery · Infisical · Hermes-Agent · homelab-incident-reports*
