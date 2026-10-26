# Hermes State Backup Fails After Upgrade — Tolerated Errors Become Fatal, Discarded Output Hides Why

**Date:** 2026-10-25  
**Severity:** P2 Medium  
**Status:** Resolved (follow-ups open)  
**Affected:** Hermes-Agent — weekly Hermes State Backup cron (`638f655cd387`) and its retention on the backup share  
**Duration:** One failed scheduled run (2026-10-25 06:10) to verified fix (21:39 the same day), about 15.5 hours. The underlying per-file errors had been present on every weekly run for weeks without being noticed, and the retention bug predates both.

---

## Summary

The Cron Failure Watchdog flagged two failed jobs on the morning of 2026-10-25. Triage split them. A Backup Verification alert that called the Ollama appdata archive "truncated" was a false positive: the local models had been removed on purpose on 10-15, and a decision note already said the smaller archive was expected. The other alert, a failed Hermes State Backup, was real. The 10-22 upgrade of Hermes-Agent from 0.20.5 to 0.22.4 had changed how `hermes backup` treats files it cannot read. A handful of per-file read errors that had been logged and tolerated every week now failed the whole command. The backup script threw away the command's output, so the alert said only that the command had failed, not why. Diagnosis took two rounds. First a stale root-owned snapshot was moved out of the Hermes home directory, and the script was changed to capture the command's output. The next run then showed the remaining causes: four root-only files, plus a cron output file that another job deleted mid-run. Along the way, the job's own keep-3 clean-up turned out to have been keeping only one Hermes backup. The fix delivers zips that have known gaps, stays silent when the only skipped files are on a commented allowlist, and warns, without failing, on anything else. A manual run then delivered a 721 MB zip silently, leaving three backups on the share. No data was lost: the zip from the failed run was intact and was saved to the share before any change was made.

---

## Timeline

| Time | Event |
|------|-------|
| 2026-10-15 | Local Ollama models removed on purpose; Ollama kept only as Frigate's cloud gateway. The 10-17 appdata set produces a ~1.5 KB `ollama.tar.gz`. |
| 2026-10-18 06:14 | Hermes State Backup reports success on 0.20.5. The backup log records `errors=8`, unnoticed. |
| 2026-10-22 ~21:23 | Hermes-Agent redeployed on 0.22.4 as part of a large manual update day. |
| 2026-10-24 07:00 | Backup Verification flags `ollama.tar.gz` as truncated (−99% against the median), the second week in a row. |
| 2026-10-25 06:16 | Hermes State Backup fails: `ALERT: hermes backup command failed`, with no reason given. |
| 2026-10-25 07:03 | Cron Failure Watchdog reports both failures. |
| 2026-10-25 evening | Read-only triage. The Ollama alert is matched to the 10-15 decision note (false positive). The Hermes logs show the same per-file errors on the 10-18 "success" and the 10-25 failure; only the version differs. |
| 21:20 | The intact 718 MB zip from the failed run is copied from staging to the backup share. The root-owned August pre-upgrade snapshot is moved out of `HERMES_HOME`. |
| 21:24 | Script v1 installed: captures the backup command's output, and fixes the keep-3 clean-up pattern. |
| 21:25–21:28 | Manual run fails again, but now the alert carries the reason: five files "could not be added". |
| 21:35 | Script v2 installed: delivers zips with gaps, allowlists the expected root-only files, and warns on anything else. |
| 21:36–21:39 | Manual run succeeds silently. The archive reports `errors=4` (the allowlisted files only), and a 721 MB zip lands on the share. Verified. |

---

## Root Cause

**Primary cause: an upgrade changed what counts as failure.** `hermes backup` walks the whole Hermes home directory (`/mnt/user/appdata/hermes`, mounted at `~/.hermes`) and zips it. Some files there cannot be read by the container's `hermes` user. On 0.20.5 those files were skipped, counted in the summary line and ignored, and the command exited 0:

```
2026-10-18 06:13:56 INFO hermes_cli.backup: backup phase=archive status=complete ... errors=8 bytes=824450365
```

On 0.22.4 the same situation produces a usable zip but a non-zero exit:

```
Archive kept, but 5 file(s) could not be added:
  <root-only file>: [Errno 13] Permission denied: ...
  ...
  cron/output/a763628ed6c7/2026-10-25_17-15-47.md: [Errno 2] No such file or directory: ...
```

The weekly runs had been "green" while quietly skipping files all along. The upgrade didn't create the gaps; it made them fatal.

**What the unreadable files were.** Two groups contributed:

1. **A root-owned pre-upgrade snapshot from August** sitting inside `HERMES_HOME` (`preupgrade-v020-20260808/`). Its directory was owned by root, so the backup's SQLite safe-copy step couldn't open the four databases in it: `attempt to write a readonly database`.
2. **Four root-only files** (mode 0600/0700, owned by root): two old credential backups and two root-only rotation scripts. These are unreadable to Hermes on purpose.

A fifth, intermittent cause appeared on the diagnostic run: a cron output file from the 5-minute Webhook HMAC Proxy Watchdog (`a763628ed6c7`) was deleted between the scan and archive phases. With a job firing every five minutes, this race can happen on any run.

**Why it was hard to see: discarded output.** The script ran the backup as:

```bash
if ! "$H" backup -o "$STAGING/$HERMES_ARCH" >/dev/null 2>&1; then
    echo "ALERT: hermes backup command failed at $(date -Is)"
```

Everything the command said about the failure was thrown away. The cause had to be pieced together from the agent log by correlating timestamps, and the second cause only surfaced once output capture was in place.

**Found along the way: the retention bug.** The keep-3 clean-up ran once per archive family:

```bash
for PREFIX in hermes_ hermes_root_scripts_; do
    ssh -T "$HOST" "cd '$REMOTE' && ls -1t ${PREFIX}*.zip ${PREFIX}*.tar.gz ... | tail -n +4 | xargs -r rm --"
```

`hermes_*` also matches `hermes_root_scripts_*`. So the "Hermes" family was really both families mixed together, and "keep 3" quietly became as few as one Hermes zip, with the small script tarballs taking the other slots. The share held a single Hermes state backup, from 10-18. The 10-11 zip had been deleted by the job meant to protect it.

**Ruled out:**
- **Free space:** 253 GB free where the zip is staged.
- **CLI flags:** `-o` is still valid in 0.22.4.
- **The Ollama alert:** a separate, already-documented expectation, not a shared cause.

---

## Remediation

1. **Saved the stranded backup first.** The failed run had left a complete 718 MB zip in staging, which passed a `zipfile -t` integrity check. It was copied to the share under the script's own naming, with matching ownership, and verified byte for byte:
   ```bash
   cp -n --preserve=timestamps .../staging/hermes_20261025_061046.zip /mnt/user/backup/hermes_backups/
   chown --reference=<10-18 zip> ...; chmod --reference=<10-18 zip> ...
   cmp <staging copy> <share copy> && echo IDENTICAL
   ```
2. **Moved the stale snapshot** out of `HERMES_HOME` to `/mnt/user/backup/hermes-preupgrade-v020-20260808`, deliberately not into the folder that CA Appdata Backup and the verifier treat as theirs. That removed the four SQLite errors.
3. **Script v1: made failures explain themselves, and fixed retention.**
   ```bash
   if ! "$H" backup -o "$STAGING/$HERMES_ARCH" >"$ERRLOG" 2>&1; then
       echo "ALERT: hermes backup command failed at $(date -Is)"
       echo "Last lines of $ERRLOG:"
       tail -n 5 "$ERRLOG"
       exit 1
   fi
   ```
   ```bash
   ls -1t ${PREFIX}[0-9]*.zip ${PREFIX}[0-9]*.tar.gz   # [0-9] keeps the families apart
   ```
   The next manual run still failed, but this time the alert named the five skipped files, which is how the second cause was found.
4. **Script v2: deliver zips with gaps, and warn only on surprises.** If `hermes backup` exits non-zero and its log says "could not be added", and the zip exists, the script delivers it; the existing size check still applies. It then compares each skipped path with a commented allowlist near the top of the script:
   ```bash
   # Files `hermes backup` is expected to skip: root-only, unreadable by the hermes user. ...
   EXPECTED_SKIPS=(
       <old credential backup 1>
       <old credential backup 2>
       bin/<rotation script 1>
       bin/<rotation script 2>
   )
   ```
   If every skipped file is on the list, the run stays silent. Otherwise it prints a `WARNING: ... could not be added: <files>` line and exits 0. A no-agent Hermes cron delivers stdout verbatim and records only non-zero exits as failures, so the warning reaches the phone without tripping the Cron Failure Watchdog. If the log format ever changes and no file names can be parsed, it warns rather than staying silent. Any other non-zero exit is still a hard failure.

   Each install followed the same routine: a dated backup of the current script, a checksum check of the new one, owner and mode copied with `--reference`, `bash -n`, and a `diff` reviewed against the intended change only.
5. **Verified** with a manual run (`hermes cron run 638f655cd387`, as the `hermes` user):
   - status `completed`, recorded as "silent (empty output)"
   - archive summary `errors=4`, exactly the allowlisted files
   - new 721 MB zip on the share, giving three Hermes zips (10-18, 10-25 06:16, 10-25 21:36) and three script tarballs
   - staging cleared of zips

---

## Prevention

- **Output capture.** The backup command's output is now kept on failure and quoted in the alert. A future failure arrives with its own reason.
- **Retention.** Retention now works: each archive family keeps its own newest three.
- **Allowlist instead of blanket tolerance.** Known, intentional gaps live in one commented list in the script. Anything outside it surfaces as a non-fatal warning.
- **Format-drift safety net.** If a future CLI changes the log format, the script warns rather than going quiet.
- **Open follow-ups:**
  - Confirm the scheduled run on 2026-11-01 06:10 is silent, and that the share still holds three zips (10-18 rotated out).
  - Expect the Ollama truncation alert once more on 10-31; it should clear on 11-07. If it still fires on 11-07, treat it as real.

---

## Lessons Learned

1. **A script that throws away a command's output turns every failure into a mystery.** `>/dev/null 2>&1` on the one command whose failure you alert on means the alert can never say why. This is at least the fourth time this pattern has cost diagnosis time on this host. The earlier three: a log-rotation hook in the MariaDB container that discarded its errors, the Honcho update script discarding its `git fetch` errors, and the MariaDB dump script discarding `mariadb-dump`'s errors. The rule now: a watchdog script may be silent on success, but it must never be silent about the reason for a failure. Capture the output and quote its tail in the alert.
2. **"Green" isn't the same as "complete", and upgrades can redefine it.** The 10-18 run reported success while skipping eight files. The upgrade didn't introduce the gaps; it changed the exit-code contract and exposed them. A success signal is only as good as the definition behind it. Version bumps that touch backup or verification tools deserve a check of the exit-code semantics and the summary counters, not just "did it run".
3. **The backup job's own clean-up was deleting backups.** A glob that matched two families meant "keep 3" quietly became as few as one, and nothing checked the result. Retention is part of what a backup does, and it needs the same scrutiny as the backup itself: a count of what's actually left on the share is cheap and would have caught this weeks ago.
4. **Design warnings around what's unexpected, not around every deviation.** Every run will skip the four root-only files, so a warning that fired whenever anything was skipped would fire every week and soon be ignored. An explicit, commented allowlist keeps the run silent for known, intentional gaps and loud for new ones, and a parse-failure warning covers the case where the check itself stops working. That keeps the signal worth reading.

---

## Related

- [Silent Monitoring Failure: Backup Verification Never Actually Checked Anything](2026-07-02-backup-verification-silent-monitoring-gap.md)
- [Silent-Success Failure Modes in Agent Configuration Management](2026-08-09-silent-config-failures.md)

---

*Environment: KONYKS-SERVER (Unraid) · Hermes-Agent 0.22.4 (no-agent cron scripts) · Unraid backup share · CA Appdata Backup · homelab-incident-reports*
