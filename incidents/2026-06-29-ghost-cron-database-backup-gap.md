# Ghost Cron Trigger Causes Silent Multi-Database Backup Outage

**Date:** 2026-06-29
**Severity:** P1 High
**Status:** Resolved
**Affected:** MariaDB, PostgreSQL (immich, honcho), infisical-db — Unraid User Scripts scheduling layer
**Duration:** Backups silently stopped at a host reboot on 2026-06-26; undetected for approximately two days

---

## Summary

A routine cron-inventory audit led to investigating whether the homelab's MariaDB and PostgreSQL
databases were actually receiving dump-level backups. The answer turned out to be yes — properly
engineered, transaction-safe dumps had been running nightly — but they had silently stopped two days
earlier, following a host reboot. Root-cause investigation found the backups had been triggered by a
cron entry that existed only in the cron daemon's runtime memory and was never written to any
boot-persistent configuration. Every standard scheduling surface was checked and confirmed empty before
the real mechanism was identified. The investigation also surfaced a related, unconnected gap: a
PostgreSQL instance backing a recently onboarded secrets-management service had never been added to
backup coverage at all. Both issues were resolved in the same session.

---

## Timeline

| Time | Event |
|------|-------|
| 2026-06-26, ~02:00–03:00 | Last successful MariaDB/PostgreSQL dumps before the gap began |
| 2026-06-26, ~11:56 | Host reboot; the in-memory cron entry triggering both backup scripts is lost |
| 2026-06-28 | Cron-inventory audit begins; backup coverage status flagged as unverified |
| 2026-06-28 | Investigation confirms MariaDB/PostgreSQL dump mechanisms are real and well-built, but two days stale |
| 2026-06-28 | Manual dump run executed immediately to close the live gap |
| 2026-06-28 | Exhaustive elimination across every persistence mechanism confirms no surviving trigger exists |
| 2026-06-28 | Persistent scheduling re-registered; a fourth database (infisical-db) added to the existing rotation |
| 2026-06-28 | Both fixes verified: real dump files produced, retention rotation confirmed, scheduling confirmed present in the actual boot-persisted configuration file |

---

## Root Cause

The backup scripts (`mariadb-dump --all-databases --single-transaction --routines --triggers --events`
for MariaDB; `pg_dumpall --globals-only` plus per-database `pg_dump -Fc` for PostgreSQL) were sound and
had been producing valid, growing, correctly retained dump sets. The problem was entirely in how they
were triggered.

Investigation eliminated every standard persistence mechanism in turn: root's crontab, `/etc/cron.d/`,
systemd timers, the `at` queue, container-internal crontabs, and the boot-time custom script were all
confirmed empty of any reference to either script. The Unraid User Scripts plugin had per-script
`schedule` files showing the expected cron expressions, but these turned out to be artifacts of an
older plugin version — the currently installed version reads only a central `schedule.json`, which
showed both jobs as disabled.

The most likely explanation: at the time the scripts were originally created, something registered an
entry directly into the running cron daemon's state without writing it to any persistent source. The
daemon kept honoring that entry across normal operation, but a subsequent regeneration of the
persistent cron configuration (triggered by opening the scheduling UI, which read the saved — disabled
— state) silently dropped it from the file used to rebuild the daemon's configuration on the next
restart. The reboot on 2026-06-26 was that restart: the in-memory entry was gone, and nothing was left
to replace it.

Separately: `infisical-db`, the PostgreSQL instance holding the secrets manager's machine identities and
project metadata, was never added to the existing PostgreSQL backup rotation when that service was
onboarded earlier in the week. The backup script hardcoded references to the two databases it was
written for and had no mechanism to pick up a new one automatically.

---

## Remediation

1. **Stopped the bleeding immediately** — ran both backup scripts manually to produce current dumps of
   all affected databases before doing anything else, independent of fixing the scheduling.
2. **Re-registered persistent scheduling** using the mechanism actually read by the current plugin
   version: updated the central schedule configuration to mark both jobs active with their correct
   cron expressions, appended matching entries to the custom-schedule file the plugin merges into the
   boot-persisted cron configuration, and ran the plugin's own regeneration step.
3. **Verified persistence directly** by reading the literal contents of both the live cron configuration
   and the boot-persisted custom-schedule file after regeneration — not just trusting that the
   regeneration command exited cleanly.
4. **Added infisical-db to the PostgreSQL backup script** as a fourth dump target, using its own
   dedicated database role rather than assuming it shared credentials with the existing targets, and
   extended the existing retention-rotation logic to cover the new dump prefix.
5. **Smoke-tested the updated script**, confirming a real, non-trivial dump file was produced for the
   new database and that the retention logic correctly applied the same keep-7 rule to it.

---

## Prevention

- Persistent scheduling for both backup jobs is now registered through the same mechanism already
  confirmed working for an existing, reliable weekly scan job — used explicitly as the reference
  pattern rather than re-deriving the registration format from scratch.
- The fix for `infisical-db` establishes a precedent: any future database added to the stack gets
  checked against existing backup coverage as part of onboarding, not discovered missing by accident.
- Open follow-up: the exact mechanism that originally created the in-memory-only cron entry was never
  definitively identified — only ruled out as coming from any standard persistent source. If a similar
  "looks scheduled, isn't persisted" pattern recurs elsewhere, the same elimination method applies.

---

## Lessons Learned

1. **A cron job that is currently running is not evidence that it is durably scheduled.** The failure
   here wasn't in the backup logic — it was in an execution path with no recorded origin, invisible right
   up until a reboot tested it. Verifying that a scheduled job survives a restart requires reading the
   boot-persistent configuration file, not observing the daemon's current behavior.

2. **Onboarding a new stateful component doesn't automatically extend existing coverage to it.** Backup
   and monitoring inventories need to be checked explicitly against new components, not assumed to scale
   with the stack. infisical-db had been running for days, accumulating state, with no backup running
   against it.

3. **"Persisted" has to mean readable from disk after a real restart**, not just present in a daemon's
   current behavior. The fix was verified by reading the actual boot-persisted file, not by trusting a
   regeneration command's exit code.

---

*Environment: KONYKS-SERVER (Unraid) · MariaDB · PostgreSQL · Infisical · Unraid User Scripts · homelab-incident-reports*
