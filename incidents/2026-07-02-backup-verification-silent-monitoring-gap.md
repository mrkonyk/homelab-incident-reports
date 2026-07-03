# Silent Monitoring Failure: Backup Verification Never Actually Checked Anything

**Date discovered:** 2026-07-02
**Severity:** P1 High
**Status:** Structurally resolved; full pass-path validation pending the next real backup cycle
**Affected:** Weekly automated backup verification (agent-mode cron job)
**Duration:** Unknown exact start — confirmed broken for the job's entire observed run history up to discovery

---

## Summary

A weekly automated job responsible for verifying that appdata backups completed successfully had reported a healthy status for its entire run history. During an unrelated cron-fleet audit — reviewing jobs as candidates for cost and privilege-scope reduction, not investigating suspected failures — the job was selected for conversion from an LLM-agent-driven check to a plain script. That conversion required real filesystem access to the backup location, which turned out not to exist: the container running the job had never had a path to the backup folder at all. Log evidence showed the agent-mode version had been hitting failed tool calls against an unreachable path on every run, triggering an internal failure-safety guard that let the agent's turn complete normally — and that normal completion was indistinguishable, in the job's status field, from a real, verified-healthy backup. No backup had ever actually been checked. The gap was closed by giving the job real, scoped filesystem access and converting it to a script whose result reflects the actual check outcome.

---

## Root Cause

Two factors combined:

1. **Missing infrastructure access, present since the job's creation.** The container running the check had no filesystem path to the backup location, and no available tool provided an equivalent way to inspect it. This was never a regression — there is no evidence the job ever had a working path to check.

2. **A status field that measures the wrong thing for agent-mode jobs.** The completion status recorded for an LLM-agent-driven job reflects whether the agent's own turn finished without crashing — not whether the task the turn was assigned actually succeeded. When the agent repeatedly failed to reach the backup path, a built-in failure-safety guard allowed the turn to end gracefully rather than error out. That graceful ending registered as a normal, healthy completion, with no distinction in the stored status between "checked and healthy" and "never actually checked."

---

## Timeline

- An unrelated cron-fleet audit reviews this job as a strong candidate for conversion to a non-agent script, based on cost and privilege scope — no suspicion at this point that the check itself wasn't working.
- Attempting the conversion surfaces that neither available tool can reach the backup path — the access simply doesn't exist yet.
- A scoped, read-only filesystem path is added to make the conversion possible at all.
- With real access in place for the first time, log evidence from the job's full run history is reviewed and shows repeated failed access attempts followed by the safety-guard completion, on every prior run.
- The converted script is tested in isolation across multiple scenarios, then re-validated against real historical backup data through the new access path.
- The first live production run — executed through the real scheduling pipeline, not a standalone test — confirms the full pipeline (execution, alerting, delivery, state tracking) works end-to-end for the first time in the job's history. It correctly reports a stale-backup condition, which is the expected result given the day it happened to run on, not a new failure.
- Validation of the check's "healthy backup, normal size" pass path remains outstanding, pending the next real scheduled run.

---

## Remediation

- Added a narrowly-scoped, read-only filesystem path to the specific backup location needed — no broader access granted.
- Converted the job from an LLM-agent-driven check to a plain script whose exit status directly reflects the real outcome of the check.
- Tested the script in isolation across healthy, shrunk, missing, zero-byte, and no-prior-data scenarios before deployment.
- Re-validated the script's logic against real historical backup data once real access was available, rather than relying solely on synthetic test cases.
- Ran the converted job through the actual production scheduling pipeline (not a standalone test) to confirm end-to-end behavior — scheduling, execution, alert generation, delivery, and state tracking all functioning correctly for the first time.

---

## Prevention

- Documented generally, not just for this one job: for any agent-mode job, a "completed successfully" status does not by itself confirm the underlying check succeeded — it only confirms the agent's own turn didn't error out. Verifying an agent-mode monitoring or verification job means checking what the tools it called actually returned, not just its completion status.
- Recommended review of any other agent-mode jobs performing monitoring or verification work, since this failure mode is generic to the agent framework and nothing so far confirms this was the only job affected by it.
- Converting agent-mode monitoring/verification jobs to scripts with narrowly-scoped, real access — where feasible — structurally closes this class of gap, since a script's result is tied directly to what it actually checked rather than to whether an agent's reasoning turn happened to complete without error.

---

## Lessons Learned

**A monitoring job's own "healthy" status is not, by itself, evidence that the monitoring is working.** This is a second-order version of a lesson that shouldn't need relearning twice in one system: a status field is a claim, not a fact, and that applies as much to the tooling meant to catch problems as to anything else being monitored.

**This was found by accident, while working toward an unrelated goal.** The job was picked up for cost and privilege-scope reasons, not because anything appeared broken. Deliberately auditing what monitoring and verification jobs can actually reach — not just what they're configured to check — would catch this class of gap proactively instead of by chance.

**A long, unbroken history of passing checks deserves more scrutiny, not less, when nothing independently ties that history to a real underlying check ever having succeeded.** Consistency is not itself evidence of correctness if the access needed to perform the check was never confirmed to exist.

---

*Environment: KONYKS-SERVER (Unraid) · Hermes-Agent cron fleet*
