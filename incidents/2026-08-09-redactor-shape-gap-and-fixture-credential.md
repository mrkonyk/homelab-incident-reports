# Redaction Filter Shape Gap, and a Real Credential Inside the Filter Itself

**Date:** 2026-08-09
**Severity:** P1 High
**Status:** Resolved
**Affected:** `redact_secrets.sh` (Claude Code hooks), Hermes-Agent configuration handling, credential guard PostToolUse hook
**Duration:** Redaction gap latent from 2026-07-26; exploited 2026-08-08; fully remediated 2026-08-09

---

## Summary

A shell-based redaction filter used as the sanctioned way to read credential-bearing configuration
files was believed to be general-purpose. It was not. Its coverage was keyed to a specific textual
shape — `VAR=value`, as found in environment files — and silently passed through the YAML `VAR: value`
form used inside the Hermes-Agent configuration file. On 2026-08-08 four live credentials were
printed in plaintext into an operator transcript while reading that file *through* the filter, with
no error and no warning. Remediation added a new pattern for the YAML shape and verified it against
the real file rather than a reconstruction. That verification then corrected a second, more important
misunderstanding about how the filter decides what to redact. Separately, the remediation work
surfaced that the filter's own test fixtures contained a real (since-rotated) credential, in a
world-readable file, that had been scrubbed from the version-controlled copy months earlier but
never from the live one — a divergence previously misread as "the repo copy is just stale."

---

## Timeline

| Time | Event |
|------|-------|
| 2026-07-26 | Pattern 7 (`VAR=value`) added to the filter and labelled detection-in-depth. Version-controlled copy scrubbed of a real fixture credential; live copy not touched. |
| 2026-08-08 ~09:00 | Four credentials printed in plaintext into an operator transcript during a ranged read of the agent config file, performed through the filter. |
| 2026-08-08 | All four credentials rotated. Transcript scrub completed; 8 occurrences replaced, integrity verified. |
| 2026-08-09 19:04 | Pattern 8 (YAML `VAR: value`) implemented and installed. Self-test 19/19 → 24/24. |
| 2026-08-09 ~21:00 | Substrate check against the real config file. Coverage model corrected. |
| 2026-08-09 22:34 | Fixture credential scrubbed from the live filter and its backups; live and version-controlled copies reconciled to byte-identical. |
| 2026-08-09 ~23:30 | PostToolUse detector diagnostic completed; residual false-alarm behaviour characterised and accepted. |

---

## Root Cause

**Primary — shape-specific coverage presented as general.** The filter's pattern set matched
credential-bearing lines by textual form. Pattern 7 handled the environment-file assignment form.
Configuration for the agent's MCP integrations stores the same secrets under YAML mapping syntax
inside per-server `env:` blocks. That is a different form, matched by nothing, so the values passed
through untouched. The filter emitted no marker when it changed nothing, so a fully redacted read
and a fully unredacted read were indistinguishable to the operator.

A compounding operator error: a documented rule requires targeted single-key extraction from that
file specifically to avoid this class of read. A ranged read was used instead. The rule was correct
and was not followed.

**Secondary — a real credential inside the test fixtures.** The filter carries self-test fixtures
that must preserve realistic shape to exercise the patterns meaningfully. One fixture had been
built from a real endpoint credential rather than a synthetic one. A scrub in July corrected the
version-controlled copy and amended it out of history, but the live executable copy was never
touched. The two copies had therefore diverged, and the hash mismatch was read as staleness rather
than investigated.

---

## Remediation

**1. Pattern 8.** Added as a second substitution block immediately after Pattern 7, deliberately
reusing Pattern 7's guards rather than introducing new ones — the same suffix exclusion list, the
same minimum-length and type guards — keyed on `:` instead of `=`, tolerating optional quoting on
both sides and arbitrary leading indentation, with the identifier left visible and only the value
replaced.

Verification was designed so that a pass could not be produced by a pattern that had quietly
stopped matching:

- Five new fixtures (unquoted, double-quoted, single-quoted at depth, plus two negatives).
- Both negatives are names containing `TOKEN` that must *survive* — a blanket matcher would redact
  them and fail.
- Result proven by diff rather than inspection: exactly 4 changed lines of 21, remaining 17
  byte-identical, line count unchanged.
- Expected post-install hash computed locally from pre-image plus patch, then matched byte-for-byte
  on the target both at staging and after install.

**2. Substrate check.** The above proves the pattern against a *model* of the file. A counts-only
check was then run against the real file — no content printed, only three numbers compared. It did
not reconcile against the predicted value, and the discrepancy was instructive: the prediction
assumed Pattern 8 redacted every non-excluded line in an `env:` block. It does not. It inherits
Pattern 7's *secret-name keyword list* and only considers names that hit it. Three buckets, not two.

The four credentials that leaked were caught — but they were caught because they were conventionally
named. A high-entropy value under an unconventional name passes untouched. The check also surfaced
a near-miss worth recording: one surviving key looked like it hit a `_NAME` suffix exclusion but
did not; it survived for an entirely different reason, and the two mechanisms produce identical
visible output.

**3. Fixture credential and copy reconciliation.** The live filter's real fixture value was replaced
with the synthetic already present in the version-controlled copy — chosen over minting a new one so
the two copies would converge. Shape equivalence was established by measurement (length, character
class, anchored regex) without printing either value. The substitution was length-preserving, so the
unchanged file size independently corroborated that shape held.

Both existing backups were found to carry the value as well and were removed *after* the replacement
was verified, so a rollback existed throughout. Version-control history was confirmed clean across
all refs and paths, not merely for the file in question. The live and repository copies are now
byte-identical, verified by comparison rather than by eye — making the file header's own
canonical-path claim true for the first time since it was written.

**4. Detector diagnostic.** A separate question remained: would the new fixtures trip the credential
guard's PostToolUse detector, producing false rotation alerts for future readers? They do not,
individually or in aggregate. But a whole-file read *does* alarm. Rather than assume the new
fixtures were responsible, the cause was isolated by ablation — file minus the new block still
alarms; new block alone is clean; file minus two specific pre-existing fixture lines is clean. The
alarm is attributable solely to two older fixtures whose realistic shape is exactly what makes them
useful.

---

## Prevention

**Changed**

- Pattern 8 added to the redaction filter; self-test extended 19 → 24 including negative cases.
- Real credential removed from the live filter and from all backups of it.
- Live and version-controlled copies reconciled to byte-identical; committed.

**Documented / decided**

- The targeted-extraction rule for the agent config file **stays in force**. The substrate check
  established that "piped through the filter" remains both shape-specific and name-specific, so the
  rule that was skipped on 2026-08-08 is not superseded by the fix.
- Whole-file reads of the filter are expected to raise a rotation alert that is now a pure false
  positive. Both obvious fixes are worse than the problem — reducing fixture realism weakens the
  filter's own coverage, and teaching the detector to ignore the shape means editing a security
  control that would need its own verification pass. The accepted mitigation is procedural: nothing
  requires reading that file whole.
- Diagnostic scaffolding was retained rather than discarded. Its survival from a prior session was
  the only reason the original test method could be reused instead of reinvented.

**Open follow-ups**

- Nothing keeps the live and version-controlled copies in sync. This was a point-in-time
  reconciliation, not a control; the next live edit re-diverges silently, which is precisely how the
  July divergence arose and went unnoticed for two weeks.
- The reconciliation commit is local-only. The working copy has no wired push path, so the copy the
  file header calls canonical is not in the origin remote at all.
- Whether unconventionally-named secrets warrant coverage remains undecided. Extending the keyword
  list is another guess, and each addition makes the filter feel more general than it is.

---

## Lessons Learned

**A control that fails open and silently is worse than no control, because it is trusted.** The
filter changed nothing and said nothing, so a redacted read and an unredacted read looked identical
at the point of use. The operator's confidence came from invoking the filter, not from any signal
the filter produced. Any sanitiser that can legitimately make zero changes should say so — the
absence of a marker is what converted a coverage gap into an exposure.

**Verifying against a reconstruction of the substrate is not verifying against the substrate.**
Pattern 8 passed a purpose-built fixture suite convincingly, and the fixture suite encoded the same
mental model that produced the pattern. Only the counts-only check against the real file exposed
that the coverage rule was name-keyed rather than structural. The fixtures could not have caught
that, because the fixtures were built from the misunderstanding.

**The detector worked, and the signal was not read.** The same PostToolUse hook that produced a
false alarm in this diagnostic had, a day earlier, correctly flagged the real credential sitting in
the live filter. That firing was logged, recorded as a result, and the inference — *the live file
contains a real value* — was never drawn. The finding arrived a day later by an unrelated route.
This inverts the usual failure mode in this environment, where controls look live and are dead: here
the control was alive, correct, and unread. No code change addresses that; only the habit of
treating a fired alert as a question rather than an entry.

---

*Environment: KONYKS-SERVER (Unraid) · Claude Code hook chain · Hermes-Agent · homelab-incident-reports*
