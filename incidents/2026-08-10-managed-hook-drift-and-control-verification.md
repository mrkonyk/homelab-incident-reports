# Undetected Drift Across a Managed Hook Chain, and the Controls Built to See It

**Date:** 2026-08-10
**Severity:** P2 Medium
**Status:** Resolved, with named follow-ups
**Affected:** Claude Code managed hook chain (PreToolUse guard, PostToolUse hook, SessionEnd scrub), homelab-infra repository checkout, ollama stack
**Duration:** Drift latent since 2026-07-10; discovered and remediated 2026-08-09 to 2026-08-10

---

## Summary

A remediation session for a redaction filter surfaced a broader problem: nothing on this
infrastructure could detect when a security control diverged from its version-controlled or
managed counterpart. Investigation found four separate instances of that class — a filter whose
live and repository copies had silently diverged in both directions, a credential guard whose
repository copy was a pre-hardening ancestor replicated into twenty-six deployment clones, two
inert hook copies that would activate in a stale state if a policy flag ever changed, and a
detection tripwire that had sat staged and unseeded since July. Separately, the repository
checkout had quietly become a managed resource that force-rebases the working tree, and had
already rewritten local commits twice. Remediation built two drift checkers with distinct
comparison models, reconciled every divergent pair, seeded and verified the tripwire, and removed
the second writer from the checkout.

---

## Timeline

| Time | Event |
|------|-------|
| 2026-07-10 | Repository copy of the credential guard receives its last commit; the live guard continues to evolve independently. |
| 2026-07-24 | A detection tripwire is staged for the PostToolUse hook. It is never seeded. |
| 2026-07-26 | Redaction filter is adopted into the repository at its declared canonical path. Live and repository copies begin diverging. |
| 2026-08-08 | A UI-created stack is linked to the repository, silently converting the working checkout into a managed resource. |
| 2026-08-09 ~03:53 | First managed pull of that checkout rewrites local commit hashes and embeds a credential in the remote URL. |
| 2026-08-09 | Redaction filter remediation begins; a hash mismatch between live and repository copies is investigated rather than dismissed as staleness. |
| 2026-08-09 | Live filter found to contain a real, since-rotated credential in a test fixture. Scrubbed; copies reconciled. |
| 2026-08-10 | Drift checker built for the server side; scope discovery finds the guard divergence and the inert copies. |
| 2026-08-10 | Second checker built for the workstation side; inert copies synced; tripwire seeded and verified firing. |
| 2026-08-10 | Stack unlinked from the checkout, restoring single-writer ownership. |

---

## Root Cause

**Nothing compared a live control against its declared source.** Each control carried a header
naming a canonical repository path, but no process ever verified that the two matched. Divergence
was therefore invisible by construction, and when a hash mismatch did surface it was initially read
as "the repository copy is stale" rather than investigated. Two of the four instances had persisted
for over a month.

**The most serious instance was structurally invisible.** The repository copy of the credential
guard is roughly a fifth the size of the running one and predates every hardening layer added
since — no shell tokenizer, no fail-closed library precondition, no known-emitter gate, no
capability allowlist, and no audit logging at all. It is the pre-tokenizer version defeated by
documented bypasses. Deployment tooling replicates that repository path into twenty-six per-stack
clones, so the obsolete artifact sits in twenty-six locations that look authoritative. No install
path from those clones to the running guard currently exists, but the risk is latent rather than
absent: it opens the moment anyone adds a bootstrap step that installs the guard from source.

**Two inert copies would have activated in the wrong state.** A policy flag restricts execution to
managed hooks only, which silently suppresses copies in user, project and local settings. Those
copies were never updated. One was a guard roughly half the size of the enforcing version; the
other, until recently, was a transcript-scrubbing hook that writes in place with no backup and no
logging, and which had previously corrupted documentation text in a live transcript. If the policy
flag were ever cleared, enforcement would silently regress.

**A second writer appeared on the working checkout without a decision being made.** Linking a
UI-created stack to the repository caused the deployment agent to begin pulling that checkout with
a forced rebase. Two such pulls had already rewritten local commit hashes, and an earlier one had
destroyed an uncommitted documentation rewrite. Nothing announced the change of ownership; it was a
side effect of an unrelated configuration action.

---

## Remediation

**Two checkers, because the files live on two machines.** The server-side checker compares each
live file against its repository counterpart. The workstation-side checker could not use that model
— the files it watches have no repository side — so it uses two modes: paired comparison between a
managed file and its inert copy, plus a pinned expected hash for the managed side of each pair and
for managed-only files with no counterpart.

The pinned half was added after an argument worth recording. The initial design used pairs alone,
on the grounds that pairs are self-maintaining and a pin adds a re-pinning ritual that can be
forgotten. The counter-argument was that a bare pair goes green when *both* sides are synced to a
downgraded version — which is precisely the failure this infrastructure had just been found to be
one careless reconciliation away from. Both were adopted: the pair catches one-sided drift, the pin
catches a synced downgrade, and because the pair check remains, re-pinning without re-syncing still
fails rather than producing a false green.

Both checkers compare by hash only and never diff. That is deliberate and documented in each
header: the watched files are credential-handling controls, so a diff would paste detection
patterns and deny-rule strings into terminals and logs, and a whole-file read of the redaction
filter is itself a documented alert trigger.

**Every divergent pair was reconciled.** The filter's live and repository copies are now
byte-identical. Both inert copies were synced to their managed counterparts rather than deleted —
deletion would have been worse, because a missing hook script exits with a code the harness treats
as a non-blocking error, converting a degraded control into no control at all.

**The tripwire was seeded and verified.** Its lineage was checked first by set comparison against
the current managed hook: every line of the running version was present in the staged file, making
it purely additive rather than a downgrade. That method was itself validated by running the same
arithmetic against a known-different version and confirming it reported a large difference.

**The stack was unlinked**, restoring the checkout to a single writer, and seven pending commits
were pushed after a shape-based credential sweep across every unpushed object.

---

## Prevention

**Built**

- Server-side drift checker with a manifest, including self-pairs so the checker and its own
  manifest are watched. The self-pairs were validated by observing them fail before the repository
  copies existed.
- Workstation-side checker with paired and pinned modes, a fourteen-case test matrix covering
  one-sided drift, synced downgrade, missing on either side, unreadable, and absent manifest, plus
  the two-step recovery path.
- Scheduled daily, quiet on agreement so it reports only on drift.

**Documented**

- The obsolete guard copy was renamed with a dated suffix and a README placed beside it explaining
  that the running guard is not versioned in the repository, that the neighbouring file is a
  pre-hardening ancestor, and that installing it would reopen documented bypasses. A matching
  warning was added to the bootstrap script.
- Seven operational rules were added to the live agent instruction file, covering the alert-tripping
  read, the name-keyed limits of the redaction filter, the post-hook's lack of a blocking exit path,
  and the two-step drift recovery.
- Authorized re-pins are now recorded out of band. A hash manifest cannot distinguish a legitimate
  re-pin from a silenced alarm; a dated written record is the only thing that can.

**Open follow-ups**

- The workstation checker does not watch itself. Neither the script nor its manifest is in its own
  pin list, and its source tree is writable by the unprivileged account and outside every audited
  path. The server-side checker has this covered; its counterpart does not.
- The server-side checker is not yet scheduled.
- A repository documentation pointer still references a section that does not exist in the file it
  names — the same defect class as the correction originally written to fix it.
- Whether the harness honours the tripwire's block directive is unverified and cannot be established
  with the offline test harness. The audit record is the durable control regardless.

---

## Lessons Learned

**A control that cannot observe itself is a claim, not a control.** Every instance in this arc was
invisible for the same reason: something asserted a relationship — this file is canonical, this copy
matches, this hook is active — and nothing ever tested the assertion. The fix was not better
controls but cheap continuous comparison, and the first thing that comparison found was that one of
its own artifacts had already drifted. The workstation checker still has this gap, which is worth
stating plainly rather than filing as resolved.

**Agreement between two copies is not correctness.** The paired-comparison model is intuitive and
self-maintaining, and it is blind to the case where both sides move together — which is the shape a
well-intentioned wrong reconciliation takes, and the shape an obsolete artifact replicated across
twenty-six locations invites. Adding a pinned expected hash cost one extra step per legitimate
update and caught a stale manifest within hours, in a case where the paired check was showing green.
The realistic adversary here is not an attacker; it is a helpful mistake, and detection has to be
designed for that.

**A test that cannot fail proves nothing, so validate the method before trusting the result.** Three
separate checks in this arc were made trustworthy by first demonstrating they could produce a
negative: the lineage comparison was run against a known-different file to confirm it reported a
difference; the detection test was preceded by a control payload proving the logging path worked, so
a subsequent zero could only mean "did not match" rather than "nothing was written"; and the drift
checker's self-pairs were watched failing before being trusted to pass. An earlier attempt that
skipped this step produced a silent zero that was indistinguishable between a broken tripwire and a
broken harness, and cost an entire verification run.

---

*Environment: KONYKS-SERVER (Unraid) · Claude Code managed hook chain · Komodo · Hermes-Agent · homelab-incident-reports*
