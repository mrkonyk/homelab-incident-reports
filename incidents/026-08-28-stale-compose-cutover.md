# Stack Cutover Deployed From Stale On-Disk Compose After Unexecuted ResourceSync

**Date:** 2026-08-28
**Severity:** P1 High
**Status:** Resolved
**Affected:** music-assistant
**Duration:** ~5 minutes hard down; ~4.5 hours running with a security regression

---

## Summary

A stack was migrated from Unraid dockerMan management to Komodo GitOps management. The
repository compose file was correct, committed, and pushed. The deploy nonetheless read a
two-month-old compose file from the host, because pushing a `files_on_host = false` change to
`resources.toml` places it on origin but does not apply it — the ResourceSync must be executed
for Komodo's Stack object to change. The Stack object still said `files_on_host = true`, so
Komodo deployed the June file it had always used.

That stale file was unpinned, carried `privileged: true` and `security_opt: [label=disable]`,
had no read-only media bind, and specified the wrong timezone. The deploy therefore pulled a
newer image than intended and performed a one-way database schema migration. A rollback to the
prior version failed on an incompatible schema, and the older version's startup routine
overwrote the application's own database backup with the already-migrated copy, eliminating the
rollback path. Service was restored by redeploying the newer version. Full remediation required
executing the sync and redeploying from the repository.

---

## Timeline

| Time | Event |
|------|-------|
| ~14:30 | Cutover attempt: running container renamed aside, stack deployed via Komodo |
| ~14:31 | Deploy reads host compose, pulls newer image, migrates database schema forward |
| ~14:35 | Rollback attempted by starting the prior version; crashes on missing column |
| ~14:35 | Prior version's startup overwrites its own database backup with the migrated copy |
| ~14:40 | Service restored by redeploying the newer version (~40s) |
| 18:50 | ResourceSync executed — the step that had been missing |
| 18:55 | Corrected deploy; container recreated from the repository clone |
| ~19:00 | Verified: nine of nine checks pass, security regression cleared |

Times are approximate where the log record is incomplete.

---

## Root Cause

Two distinct failures, one nested inside the other.

**Primary — a push is not an apply.** Komodo's ResourceSync reads `resources.toml` from the
repository and writes the result into its own Stack objects. Committing and pushing a change to
that file makes it *available* to the sync; it does not execute the sync. The Stack object
retained `files_on_host = true` and its original run directory, so `DeployStack` resolved to the
on-disk compose at `/mnt/cache_ssd/appdata/komodo/stacks/music-assistant/compose.yaml`, last
modified in June.

The state was visible the entire time and was not checked. Every container carries a
`com.docker.compose.project.config_files` label recording the compose file actually used at
deploy time. A container-classification audit run the previous day had already printed that
label showing the `files_on_host` path. Verification effort went into confirming the repository
was correct — which it was, and which was irrelevant.

**Secondary — rollback destroyed its own precondition.** The application maintains
`library.db.backup` alongside its live SQLite database. Starting the older version against the
migrated schema caused it to attempt a downgrade migration, fail on a column introduced by the
newer schema, and — before crashing — refresh its backup file from the current database. The
backup and the live database became byte-identical at the post-migration schema. The only
remaining compatible copy was six days old, inside a scheduled appdata archive.

---

## Remediation

Executed the sync, then redeployed:

1. Re-pinned the repository compose to the digest of the image now running, since the previous
   pin referenced a version the migrated database could no longer open.
2. Took a consistent online snapshot of the live database before any further deploy. A
   filesystem copy was attempted first and rejected — the write-ahead log did not match, because
   the database was live:

   ```
   sqlite3 library.db ".backup /path/library.db.consistent"
   PRAGMA quick_check   →   ok      (193 schema objects)
   ```

   Service stayed available throughout the 81-second snapshot.
3. Ran the ResourceSync in the Komodo UI.
4. Redeployed the stack and verified the `config_files` label **first**, before any other check.

Verification after remediation:

| Check | Result |
|-------|--------|
| `config_files` resolves into the repository clone | pass |
| `privileged` | false |
| `security_opt` | empty |
| Read-only media bind present | pass |
| Timezone | correct |
| Image digest matches repository pin | pass |
| Service responding, instance identity preserved | pass |
| Database schema unchanged post-deploy | pass |

Rolling back to the prior version was never available after the migration and was explicitly
ruled out rather than attempted a second time.

---

## Prevention

- The repository compose is pinned by digest to the version the database schema now requires.
- A consistent database snapshot is retained until the new version has run clean.
- A container-to-repository drift check was designed: for every running container, compare
  `com.docker.compose.project.config_files` against what `resources.toml` declares. It requires
  no API access. Its first run flagged exactly three stacks and cleared 28. Two blind spots are
  documented rather than hidden — it reads the artifact of the last recreate rather than the
  current Stack object, so an applied-but-not-yet-deployed config change reads as drift
  indefinitely; and a declared stack with no running container cannot be checked. It must also
  handle both valid declaration shapes or it false-positives on stacks that omit a run directory.
- Operational documentation now states the distinction: a push publishes a change, the
  ResourceSync applies it, and the deploy realises it. Three steps, not one.

Open follow-up: the drift check is a proposal, not yet built.

---

## Lessons Learned

**Verify against the artifact, not the source of truth you edited.** The repository was correct
and the verification confirmed it, which produced high confidence in a deploy that never read
it. The deployed container's own labels record what was actually used, and that is the only
evidence that answers the question. A correct repository is not evidence about what is running.

**A rollback that starts an older version can consume the thing that made it a rollback.**
Downgrade paths in stateful applications frequently run migration logic before failing, and
that logic may touch backup files. The safe order is snapshot out-of-band first, then attempt
the downgrade — not the reverse. This turned a recoverable version mismatch into a one-way door.

**This was the second instance of one error class.** An earlier incident on the same platform
established that pushing an image-pin change is not a deploy. The same conflation — publish
mistaken for apply — recurred one level up, at the sync. Recording the specific lesson was not
enough; the generalisation had to be stated before it stopped recurring.

---

*Environment: KONYKS-SERVER (Unraid) · Komodo GitOps · SQLite-backed application stack · homelab-incident-reports*
