# Hermes-Agent Cron Fleet Audit — Weekly IP Scan Running Clean While Primary Scan Mechanism Had Never Executed

**Date:** 2026-06-28 to 2026-06-29
**Severity:** P2 Medium
**Status:** Resolved
**Affected:** Hermes-Agent container, ip-scan.sh (weekly hardcoded-IP scan), Unraid-MCP integration
**Duration:** MCP scan section silently non-functional for an indeterminate period prior to discovery; scheduling itself was intact throughout

---

## Summary

Following the resolution of the ghost-cron database backup gap, audit scope was extended to all
scheduled jobs running inside the Hermes-Agent container to check for similar silent failures. The
weekly IP scan script — responsible for detecting hardcoded Docker bridge IPs in container
configurations — was found to be running on schedule, exiting 0, and writing "clean" log entries on
every run. Manual execution with stderr captured revealed the script's primary scan mechanism had been
crashing and falling back to empty results on every execution. Only the secondary scan (static
config-file grepping) was actually running. No log entry, alert, or error message had ever indicated
anything was wrong.

The root cause split into two independent bugs in the MCP section of the script, each of which would
have been sufficient to produce silent failure on its own. Both are documented in detail separately.

---

## Timeline

| Time | Event |
|------|-------|
| 2026-06-28, morning | Ghost-cron backup gap resolved; audit scope expanded to cover all Hermes-Agent scheduled jobs |
| 2026-06-28 | ip-scan.sh log reviewed; all recent entries read "Weekly IP scan: clean" — no errors, no anomalies |
| 2026-06-28 | Script executed manually inside Hermes-Agent container with `2>&1` to capture stderr |
| 2026-06-28 | Stderr shows: `Unraid-MCP scan error: Expecting value: line 1 column 1 (char 0)` — a JSON parse failure that was being silently swallowed |
| 2026-06-28 | Confirmed: MCP_OUT variable is empty on every run; only Scan 2 (file-grep) was contributing findings |
| 2026-06-28 to 2026-06-29 | Root cause investigation across two independent failure classes (see ip-scan-two-failure-classes) |
| 2026-06-29 | Both bugs fixed; instrumented verification run shows all 20 containers scanned, zero JSON decode errors |
| 2026-06-29 | Full script run produces clean exit with no stderr output; log entry written normally |

---

## Root Cause

The ip-scan.sh script has two scan sections. Scan 1 uses a Python heredoc to connect to the
Unraid-MCP container, retrieve a list of running Docker containers, and inspect each one's labels
and network settings for hardcoded non-safe IPs. Scan 2 uses grep to inspect config files on mounted
volumes. The two sections are independent; Scan 2 does not depend on Scan 1.

The Python block in Scan 1 wraps its entire body in a single `try/except Exception` clause. When any
exception occurs, the handler prints `Unraid-MCP scan error: <message>` to `sys.stderr` and exits
the block. In the script's cron context, stderr is not redirected to the log file — only stdout is
captured in `MCP_OUT`. A crashing MCP section produces an empty `MCP_OUT`, which is
indistinguishable from a clean MCP section that found no issues. The script then runs Scan 2, finds
no issues there either, writes "Weekly IP scan: clean" to the log, and exits 0.

This error-reporting structure meant that any bug in the MCP section would be invisible in normal
operation. Two such bugs were present simultaneously (see separate incident for technical detail).

---

## Remediation

1. Extended audit scope to Hermes-Agent immediately after resolving the backup-gap incident on the
   same day — not deferred to a separate session.
2. Executed ip-scan.sh manually with stderr visible to observe actual runtime behavior, rather than
   inferring health from the log file.
3. Traced the silent failure to the MCP block via the empty `MCP_OUT` variable; confirmed Scan 2 was
   functioning normally.
4. Fixed both underlying MCP bugs (see ip-scan-two-failure-classes).
5. After fixing, ran the script with per-container instrumentation to confirm all 20 containers were
   reached and that the `except json.JSONDecodeError` guard added as part of the fix was not being
   triggered for any container.

---

## Prevention

- Any scan or check that catches exceptions and continues should either treat the partial failure as a
  reportable condition (writing a diagnostic to the same log path as normal output) or propagate the
  failure as a non-zero exit. The current structure — catch, print to stderr, continue — provides no
  persistent record that a scan was incomplete.
- The fleet audit pattern (checking all scheduled jobs after finding a silent failure in one) is now
  a standing practice for any incident in this category.
- Future additions to the Hermes cron fleet should include a quick manual execution with stderr
  captured as part of the initial verification, before the first scheduled run.

---

## Lessons Learned

1. **An exit-0 log entry proves a script ran to completion; it does not prove the script did what it was supposed to do.** The IP scan was executing on schedule, exiting successfully, and writing clean log entries. All of these were accurate. None of them were evidence that the MCP container scan had run. A job that silently partial-fails and continues to completion is worse than a job that fails outright, because it substitutes a false-positive "clean" for the absence of signal that a hard failure would have produced.

2. **Catching exceptions broadly and continuing is a design choice with a real cost.** The `except Exception` pattern in the MCP block was written to prevent a transient network failure from making the whole scan report an error. That's a reasonable goal. The cost is that any bug — not just transient failures — is silently absorbed. The fix retained the exception handling but made failures observable: the guard was narrowed to `json.JSONDecodeError` and the outer exception path now only fires for unexpected issues, while a structured per-container continue keeps the scan running for the other 19 containers.

3. **Expanding scope after a silent-failure discovery has a high expected return.** The MCP bug would have remained undetected indefinitely if the audit hadn't been extended beyond the backup job that prompted it. Spending the additional time to sweep all jobs in the same container cost far less than discovering a second coverage gap later by accident.

---

*Environment: KONYKS-SERVER (Unraid) · Hermes-Agent · Unraid-MCP · ip-scan.sh · homelab-incident-reports*
