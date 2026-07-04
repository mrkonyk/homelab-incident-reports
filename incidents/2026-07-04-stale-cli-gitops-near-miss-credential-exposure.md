# Stale CLI Version, GitOps Near-Miss, and Cascading Credential Exposure

**Date:** 2026-07-04
**Severity:** P1 High
**Status:** Resolved
**Affected:** Claude Code sandbox container, Komodo GitOps auto-deploy (all managed stacks), Unraid API authentication, Hermes-Agent Strava integration
**Duration:** Single extended session (~4 hours from first symptom to full resolution)

---

## Summary

A routine question about Claude Code's context-window behavior on a newly-adopted model uncovered a chain of unrelated issues: a stale CLI version silently misreporting context capacity, two live bugs in the GitOps auto-deploy pipeline (one of which put the entire deploy mechanism for every managed stack at risk of accidental deletion), a credential exposure caused by an API endpoint echoing rendered environment variables in plaintext, an incorrect root-cause theory about that exposure that was pursued, tested, and then reversed after direct verification contradicted it, and a second, unrelated credential exposure caused by an imprecise file read during the investigation itself. All items were resolved and independently verified; no data loss or service outage occurred, but two of the findings (the GitOps deletion risk and the credential exposure) carried real blast radius if they had gone unnoticed.

---

## Timeline

| Time (relative) | Event |
|------|-------|
| T+0:00 | Context-usage display on a Sonnet-class model observed stuck near 100% and not resolving |
| T+0:20 | Root cause traced to a Claude Code CLI version several releases behind the minimum required for native support of the newer model's context window |
| T+0:45 | CLI updated; issue confirmed resolved |
| T+1:00 | Follow-up GitOps audit requested to verify auto-deploy health across all managed stacks |
| T+1:30 | Two live bugs found: a git-pull blocker on one stack, and a Procedure missing from tracked resource definitions with a pending delete state |
| T+1:45 | Both fixed; a stale-cache gotcha in the sync mechanism caught before declaring the fix complete |
| T+2:00 | End-to-end verification via a real deploy triggered through the full webhook chain |
| T+2:15 | During verification, an API call intended only to confirm state instead echoed a live credential in plaintext |
| T+2:20 | Credential rotated immediately; blast-radius search launched for any other copies |
| T+2:45 | A second, older exposure of the same credential found on persistent storage, independent of the session's own leak |
| T+3:00 | Root-cause theory formed for why the credential appeared unmanageable through normal tooling; theory tested via a scoped service restart |
| T+3:15 | Restart completed cleanly but did not resolve the issue — theory was wrong |
| T+3:20 | Correct explanation found via direct verification: the credential was not orphaned, it was a live, currently-relied-upon key that had simply been reused rather than issued fresh |
| T+3:30 | Credential deleted outright after confirming zero live dependents |
| T+3:40 | Unrelated second credential exposure discovered — a file read during the investigation spilled into an adjacent, unrelated secret block |
| T+3:45 | Second exposure rotated: credential regenerated at the source, full reauthorization flow completed manually outside the coding session |
| T+4:00 | All items independently verified; incident closed |

---

## Root Cause

**1. Context misreporting.** The Claude Code CLI running inside the sandbox container was several minor versions behind the release that added native awareness of a newer model's much larger context window. The older CLI had no entry for the new model in its internal context-size table and fell back to a legacy, much smaller default — causing it to report usage percentages well past 100% and to trigger compaction far earlier than necessary. The Dockerfile's install step for the CLI was unpinned (would normally resolve to the latest release on every build), but Docker's layer caching meant the install step had not actually re-run since a build well before the newer model's release, so the image had been silently frozen on an old version despite looking, on paper, like it tracked "latest."

**2. GitOps auto-deploy bugs.**
- One managed stack had an untracked certificate file sitting in its deploy-directory git clone, which blocked `git pull` and therefore blocked that stack's auto-deploy path entirely.
- A Procedure central to the entire auto-deploy mechanism (used to redeploy any stack that falls behind) existed and was actively relied upon, but had never been added to the repository's tracked resource definitions. This meant the GitOps sync tooling saw it as something that *should not exist* and flagged it for deletion. Had a routine sync operation run without that discrepancy being caught, the Procedure — and with it, the auto-deploy path for every managed stack — would have been removed.

**3. First credential exposure.** An API call made to verify a stack's deployed state returned a field containing the stack's fully-rendered environment variables, including a live API key, in plaintext in the response. This is a property of the platform's API surface, not a misconfiguration on the container itself — the field is designed to show the effective runtime configuration, which necessarily includes resolved secret values.

**4. Incorrect root-cause theory.** A copy of the same credential was found, independently of the session's own leak, sitting in a stale legacy configuration template on persistent storage — left over from before the affected service was migrated to its current GitOps-managed form, and consequently also present in several weeks of backup snapshots. Because this credential did not appear in the authentication service's own key-listing tool, the working theory was that it was an orphaned, unlisted credential that had bypassed the normal key lifecycle entirely. Reading the authentication service's actual source confirmed a plausible mechanism for this (an in-memory key cache populated once at process start, refreshed by a file-watcher that could plausibly miss a deletion event) — but the theory turned out to rest on a false premise: an earlier comparison meant to confirm the key's owner had actually been checked against a different, already-stale file, not the real source of truth. Once tested directly (querying the API for the identity attached to the key, and hash-comparing the key value against the authentication service's actual live key store), it turned out this was not an orphaned credential at all — it was a real, currently active, admin-scoped key belonging to a different service on the stack, which had simply been reused when the affected service was first stood up rather than being issued a dedicated key of its own.

**5. Second credential exposure.** While investigating where a separate service sourced its own credential from, an imprecise line-range read of a shared configuration file (intended to capture a few lines of context around one section) spilled into an adjacent, unrelated section holding a different service's OAuth credentials, printing them in plaintext. This exposure was initially misattributed as part of the first exposure's fallout before being corrected once challenged on its origin.

---

## Remediation

**Context misreporting**
```bash
docker build --no-cache -t claude-sandbox .
```
Forcing a cache-busted rebuild caused the CLI install step to re-resolve to the current release. Verified via the CLI's own version command post-rebuild.

**GitOps bugs**
- Confirmed the blocking certificate file was byte-identical to the git-tracked version before removing it, then forced a resync of the deployment tool's own bookkeeping.
- Pulled the exact resource definition the sync tool computes internally (rather than reconstructing it manually) and committed it to the tracked resource file.
- Discovered the sync tool's pending-diff state does not refresh on demand — it only updates on its own periodic poll — and waited that cycle out rather than declaring the fix complete prematurely.
- Verified via a real end-to-end test: pushed a trivial commit and confirmed it propagated through the full chain (push → webhook → redeploy) with a matching deployed hash, for the previously-broken stack specifically.

**Credential exposures**
- First exposure: the exposed key was investigated for live dependents (none found directly, but see root-cause correction above), then deleted outright once confirmed to have zero live dependents and no reason to retain admin-level scope unused.
- A leftover legacy configuration file containing a stale copy of the same credential was removed from persistent storage. A deliberate decision was made *not* to retroactively edit or scrub existing backup snapshots containing the same stale copy — fixing at the source and allowing those snapshots to age out under the existing retention schedule was judged lower-risk than editing historical backup data.
- Second exposure: the affected credential was rotated at its source (the third-party service's own developer settings, which has no self-service rotation API and required direct manual action), the old value was confirmed dead via a live API call, and a full manual reauthorization flow was completed to obtain a fresh working credential pair. The token exchange step was deliberately performed outside the coding assistant's session to avoid re-exposing the new secret through the same channel.

Before any service restart or destructive action, blast radius was mapped explicitly: active connections, scheduled job timing, and how dependent services fail (loudly with a clear error vs. silently serving stale data) were all confirmed before proceeding, and explicit go-ahead was obtained before each irreversible step.

---

## Prevention

- The missing Procedure is now part of the tracked resource definitions, closing the accidental-deletion risk permanently rather than as a one-off fix.
- A standing rule was added: files known to hold multiple services' secrets are read with a targeted search for the specific value needed, never a line-range or full-file read, since adjacent secret blocks make broad reads unsafe by default.
- A standing rule was added: any credential or secret value that appears incidentally during unrelated work must be reported with its own precise origin (what command, what file, what line) — never folded into the narrative of whatever was already being investigated.
- The unpinned CLI install step is now understood to require periodic forced rebuilds (or an explicit version pin with a deliberate bump process) rather than being trusted to "always track latest" on its own.
- Open follow-up: decide whether to pin the sandbox's CLI version explicitly going forward (consistent with how other stack images are version-pinned) or accept unpinned installs with a recurring forced-rebuild step.

---

## Lessons Learned

**An unpinned dependency is not a self-updating one.** The Dockerfile's CLI install had no version pin, which looked equivalent to "always current" — but Docker's layer caching meant it had actually been frozen at build time and never re-resolved. The same trap applies anywhere a build step assumes a mutable upstream source without a mechanism to force re-evaluation; unpinned only means "unpinned at last cache invalidation."

**A tracked-resources gap is a silent single point of failure.** The Procedure missing from source control worked perfectly in production for weeks — right up until a routine sync operation would have deleted it. Verifying that GitOps tooling agrees with reality is not the same as verifying that a system works; both checks are needed, because a system can function correctly while sitting one automated action away from losing the mechanism that keeps it functioning.

**A well-reasoned root cause is only as good as its inputs, and the discipline that catches that is direct verification, not more reasoning.** Reading the authentication service's actual source code produced a coherent, plausible explanation for the credential's behavior — but it was built on an earlier comparison made against the wrong file. The theory wasn't corrected by re-reading documentation or reasoning harder about the code; it was corrected by testing the credential's real identity directly against the system of record. The same standard was applied a second time in the same session, when an unrelated credential exposure was initially mis-narrated as connected to the first — caught only because the connection was challenged and the actual command, file, and reasoning were traced back precisely rather than accepted as stated.

---

*Environment: KONYKS-SERVER (Unraid) · Komodo/Periphery GitOps · Claude Code sandbox · Hermes-Agent · homelab-incident-reports*
