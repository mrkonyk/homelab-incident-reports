---
title: "Backup Strategy Hardening: Skip List Audit, Beszel WAL Fix, and First Restore Test"
date: "2026-06-17"
severity: "P2 Medium"
status: "Resolved"
affected: ["Appdata Backup plugin", "beszel", "unmanic", "all backed-up containers"]
duration: "Ongoing misconfiguration since backup plugin deployment; resolved this session"
summary: >
  A review of the Appdata Backup plugin configuration revealed two skip-list errors and an unresolved WAL file risk for the Beszel monitoring container. The beszel hub was being stopped and backed up directly while running, producing potentially dirty SQLite WAL snapshots. The unmanic container was being included despite having no appdata to back up, generating log noise that could mask real failures. Both issues were resolved: a User Script was created to perform a clean stop/copy/start backup of Beszel's database files before the Saturday AB job runs, and unmanic was added to the skip list. The session also completed the first-ever end-to-end restore test against the June 13 backup set, successfully restoring the Wallos container from archive. A 1.1GB bloat in the boot flash backup was identified and resolved by removing the /boot/previous rollback directory left over from the Unraid 7.3.1 upgrade.
---

**Severity:** P2 Medium

## Timeline

| Time | Event |
| :--- | :--- |
| Session start | Backup plugin skip list audited against all containers |
| ~13:05 | Identified beszel and unmanic as misconfigured in skip list |
| ~13:10 | Beszel PocketBase backup attempted via UI — failed due to live WAL files |
| ~13:12 | Clean manual backup performed: container stopped, db files copied, container restarted |
| ~13:20 | User Script created at `/boot/config/plugins/user.scripts/scripts/beszel-backup/script`, scheduled 4 AM Saturday |
| ~13:25 | beszel and unmanic flipped to Yes in AB.Main |
| ~13:30 | Restore test initiated against `ab_20260613_050001/Wallos.tar.gz` |
| ~13:40 | Wallos restored successfully; service verified operational; safety copy removed |
| ~13:50 | Boot flash bloat investigated; `/boot/previous` (1.1GB) identified and deleted |

## Root Cause

- **Beszel WAL issue**: PocketBase (the database backend for Beszel) writes active WAL files (`data.db-wal`, `auxiliary.db-wal`) while running. The Appdata Backup plugin's stop/backup/start method would stop the container mid-write, but SQLite WAL checkpointing on clean shutdown isn't guaranteed to complete before the tar snapshot begins. This left beszel outside the skip list with no native backup configured — the worst of both worlds.
- **unmanic log noise**: The container had no appdata bind mount returning content, causing AB to log "no volume to back up" on every run. With unmanic included but producing no backup, real failures could be obscured in log output.
- **Boot flash bloat**: The `/boot/previous` directory is created automatically by Unraid during OS upgrades to enable Tools → Downgrade OS. After upgrading to 7.3.1 on May 12, the previous version's full kernel and firmware files (`bzmodules` 734MB, `bzfirmware` 305MB, `bzroot` 79MB) remained on the flash drive, inflating the weekly boot backup from ~300MB to ~1.4GB.

## Remediation

### Beszel backup
Manual clean backup (stop → copy → start):
```bash
docker stop beszel
cp /mnt/user/appdata/beszel/data.db /mnt/user/appdata/beszel/backups/data_20260617.db
cp /mnt/user/appdata/beszel/auxiliary.db /mnt/user/appdata/beszel/backups/auxiliary_20260617.db
docker start beszel
```

User Script created at `/boot/config/plugins/user.scripts/scripts/beszel-backup/script`:
```bash
#!/bin/bash
DATE=$(date +%Y%m%d)
BACKUP_DIR="/mnt/user/appdata/beszel/backups"

docker stop beszel
cp /mnt/user/appdata/beszel/data.db "$BACKUP_DIR/data_$DATE.db"
cp /mnt/user/appdata/beszel/auxiliary.db "$BACKUP_DIR/auxiliary_$DATE.db"
docker start beszel

# Keep only last 4 backups of each file
ls -t "$BACKUP_DIR"/data_*.db | tail -n +5 | xargs -r rm
ls -t "$BACKUP_DIR"/auxiliary_*.db | tail -n +5 | xargs -r rm

echo "Beszel backup complete: $DATE"
```
Scheduled at `0 4 * * 6` (4 AM Saturday), one hour before the AB job. `beszel` flipped to Yes (skip) in AB.Main.

### unmanic
Flipped to Yes in AB.Main skip list to eliminate log noise.

### Restore test
```bash
# Stop container, preserve current appdata as safety copy
docker stop Wallos
mv /mnt/user/appdata/wallos /mnt/user/appdata/wallos.bak

# Extract backup
tar -xzf /mnt/user/backup/unraid/ab_20260613_050001/Wallos.tar.gz -C /

# Restart and verify
docker start Wallos
# Verified: service loaded correctly with all subscription data intact

# Clean up safety copy
rm -rf /mnt/user/appdata/wallos.bak
```

### Boot flash bloat
```bash
rm -rf /boot/previous
```
Confirmed: `/boot/previous` is only used by Tools → Downgrade OS. System boots from `bz*` files in `/boot` root.

## Prevention

- **beszel** now has a pre-AB User Script ensuring clean WAL-free database snapshots every Saturday.
- **unmanic** removed from AB scope — eliminates false-negative log noise.
- Skip list fully audited against all containers with rationale documented for each.
- Restore procedure validated end-to-end against a real backup set for the first time.

### Remaining open
- **Offsite backup (Pi at Miriah's)**: Still deferred, single point of failure on-array.
- Saturday AB job will serve as the first post-hardening report card.

## Lessons Learned

1. **"Backups exist" and "backups restore" are different claims.** The backup configuration had been treated as complete for months, but no restore had ever been attempted. The Wallos test revealed the archive structure was correct, paths resolved cleanly, and the stop/extract/start pattern is repeatable — but this was only confirmed by actually doing it.
2. **WAL-active databases require special handling.** Appdata Backup's stop/start cycle isn't sufficient for PocketBase because the WAL checkpoint isn't atomic with the tar operation. The pre-script pattern (stop → copy → start before AB runs) is the correct architectural response.
3. **Log noise degrades signal quality.** unmanic's entries normalized non-zero log output. Skip lists should be audited for accuracy to ensure logs only signal real issues.
