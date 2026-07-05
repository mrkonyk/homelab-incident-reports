# credential_guard.sh Hardening, Round 2 — Two New Exposure Vectors Found and Closed

**Date:** 2026-07-05
**Severity:** P1 High
**Status:** Resolved
**Affected:** credential_guard.sh (PreToolUse hook), Nextcloud application password
**Duration:** Gap present since the round-1 hardening pass earlier in the week; exploited (accidentally) same day it was checked

---

## Summary

During unrelated work, a context-flag grep against a configuration file containing multiple service credentials printed a live Nextcloud application password in full — despite a hardening pass earlier in the week specifically intended to catch this class of raw credential-file access. A follow-up rotation attempt then leaked its own replacement value through a second, differently-shaped command. Investigation found the guard's library-usage check accepted merely referencing the safety library's filename as proof of safe usage, without confirming any actual sanitizing function was called, and that a generated secret written to a temporary scratch file was not recognized as credential-bearing since it fell outside the guard's static path list. Both gaps were closed, generalized beyond the exact commands that triggered them, and covered by an expanded automated test suite.

---

## Timeline

| Time | Event |
|------|-------|
| Day 1 | Unrelated infrastructure task requires inspecting a multi-service configuration file's structure |
| Day 1 | A context-flag grep against that file prints a live Nextcloud application password in full |
| Day 1 | Exposure flagged immediately; password rotation approved and carried out |
| Day 1 | First rotation attempt leaks its own newly generated replacement value through a separate command |
| Day 1 | Second rotation completed cleanly; both exposed values confirmed revoked and replaced |
| Day 1 | Given this is the second real leak since a hardening pass meant to prevent exactly this, a root-cause investigation into the guard itself is opened |
| Day 1 | Dry-run of the exact leaking command against the current guard confirms it is allowed, not blocked |
| Day 1 | Two independent gaps identified in the guard's detection logic |
| Day 1 | Fixes applied and generalized; a third bug is found via self-testing the fix, unrelated to the original two |
| Day 1 | Full regression suite (36 cases, covering both this round's fixes and every case from the round-1 hardening) run clean; committed and pushed |

---

## Root Cause

Two independent gaps, plus one self-discovered defect in the fix itself:

1. **Library-mention treated as library-usage.** The guard's evidence check for "safe usage" matched on the safety library being *sourced* in a command, without verifying that any of its actual sanitizing functions were called afterward. A command could source the library and then do something entirely unrelated and unsafe to the same file, and the guard would treat the mere mention as sufficient proof of safety.

2. **Dynamically created files were invisible to the guard.** The guard's credential-path detection relied on a static list of known credential-bearing file paths. A secret-generating command that wrote its output to an arbitrary temporary file was not covered by that list, so a subsequent command reading that same temporary file was not recognized as touching credential material at all.

3. **A regex overreach introduced during the fix itself.** While patching the two gaps above, the extraction logic used to detect "a command wrote a secret to this path" incorrectly captured stray punctuation from unrelated surrounding text as if it were a flagged file path, which then spuriously matched against nearly any later command in the session. Found and fixed during the same pass, before it reached production use.

---

## Remediation

1. Required actual evidence of a sanitizing function call, not just the library being referenced, before treating a command as safe.
2. Added session-scoped tracking: once a recognized secret-generating command writes to a given path, that path is treated as credential-bearing for the remainder of the session, regardless of whether it appears on the static path list — closing the gap for temporary/scratch files.
3. Also blocked secret-generating commands that produce output with no redirect or capture at all, a more direct variant of the same underlying risk.
4. Added the missing command (a text-extraction utility not previously covered) to the set of raw commands checked against credential-bearing paths.
5. Fixed the regex overreach found during self-testing with a length and path-prefix sanity check before accepting an extracted path as valid.
6. Built a 36-case test matrix covering every case from the original hardening pass, both real leak shapes from today reconstructed with placeholder data, and new regression cases for the self-found defect. Ran the full matrix against the pre-fix guard first to confirm it reproduced exactly the expected failures and nothing else, then again against the patched version for a clean pass.
7. Rotated both affected credential values; confirmed old values rejected and new values verified working.

---

## Prevention

- The fix is committed to the same tracked repository as the round-1 hardening, keeping the guard's full history and rationale in one place rather than scattered across untracked local state.
- The expanded test suite is retained rather than discarded, so any future change to the guard can be checked against the full accumulated case history rather than just the change under consideration.
- Session-scoped tracking of dynamically created secret files is a generalizable pattern now available for any future gap of the same shape, rather than a one-off fix tied to this specific temporary file.

---

## Lessons Learned

1. **A safety check that verifies intent rather than outcome is not a safety check.** Confirming a safety library was *mentioned* is a weaker guarantee than confirming it was *used correctly*, and the gap between those two is exactly where a real exposure occurred twice in one week.
2. **Static allowlists and denylists both eventually meet a dynamic case.** A fixed list of "known credential paths" cannot anticipate every place a secret might transiently live; tracking provenance (what a command produced and where it went) generalizes further than trying to enumerate every path in advance.
3. **Testing a fix by trying to break it yourself catches things that testing it against the reported case alone does not.** The third bug in this incident was never reported by anyone — it was found only because the fix was deliberately exercised beyond the exact scenario it was written for.

---

*Environment: KONYKS-SERVER · credential_guard.sh · credential_lib.sh · Nextcloud · homelab-infra*
