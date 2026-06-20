# Appdata Backup Plugin Cannot Resolve Bind-Backed Named Docker Volumes

**Date:** 2026-06-20
**Severity:** P2 Medium
**Status:** Resolved
**Affected:** Appdata Backup plugin (Unraid), Komodo stack (2 containers), observability stack (4 containers)
**Duration:** Ongoing since initial deploy (2026-06-19); discovered via alert email same morning

---

## Summary

The Appdata Backup (AB) plugin generated six identical error emails overnight, each reporting that a named Docker volume "does NOT exist" for a Komodo- or observability-stack container. Investigation traced this to a genuine bug in AB's volume-resolution logic: it treats Docker named-volume identifiers as literal filesystem paths and calls `file_exists()` on the bare volume name, which is guaranteed to fail regardless of whether the volume's actual backing data exists. The underlying data was never at risk — all six volumes are bind-backed (`driver_opts: type=none,o=bind`) to real paths under a cache pool — but AB had no functioning code path to back any of them up, with or without the error. The six affected containers were added to AB's skip list, and a dedicated cron-based backup was created for the one volume holding genuine source-of-truth state.

---

## Root Cause

The Komodo and observability stacks are deployed via Docker Compose with named volumes that use bind-mount `driver_opts`, e.g.:

```yaml
volumes:
  komodo_mongo_data:
    driver_opts:
      type: none
      o: bind
      device: /mnt/cache_ssd/appdata/komodo/mongo
```

This is a real, on-disk bind mount — Docker just exposes it under a named-volume identifier rather than a literal host path in the compose file. `docker volume inspect` confirms the correct `Mountpoint` for each.

AB, however, retrieves volume information from Unraid's internal `DockerClient` rather than resolving it itself. For a named volume, this returns a bare string such as `komodo_mongo_data:/data`. AB's `ABHelper.php` then does:

```php
$hostPath = rtrim(explode(":", $volume)[0], '/');
if (!file_exists($hostPath)) {
    self::backupLog("'$hostPath' does NOT exist! ...", LOGLEVEL_ERR);
    continue;
}
```

`file_exists("komodo_mongo_data")` evaluates a bare string with no leading slash — it is never going to resolve to anything, regardless of where the volume's actual data lives. The plugin author left this unimplemented intentionally, marked with an inline comment acknowledging the gap (paraphrased): if there's no slash in the path, it's a named volume, not a bind mount, and isn't handled.

Even with that resolved, a second, independent issue would still block backup: AB's `allowedSources` setting is scoped to the default Unraid appdata path, while this data lives on a separate cache pool mount. The named-volume bug and the allowed-source scoping are two unrelated reasons the same six containers can never back up cleanly through AB as currently configured.

---

## Remediation

1. **Confirmed the data was never at risk.** All six affected named volumes resolved to real, populated bind-mount targets via `docker volume inspect`; the backup job itself completed successfully overall (full job log ended with a normal success line), with the six volumes simply skipped after the resolution error.

2. **Read the plugin's actual logic** rather than guessing, by inspecting `ABHelper.php` directly via SSH to confirm the failure mode was a code-level limitation, not a configuration mistake on this end.

3. **Marked all six containers as Skip** in AB's per-container settings, since no configuration change to this stack can make AB's current logic succeed.

4. **Built a dedicated backup path** for the one volume holding genuine, hard-to-reproduce state (the GitOps engine's own database): a User Script that stops the two affected containers cleanly, archives the appdata directory, restarts them, retains the last four runs, and exits non-zero (visibly, in the script's own log output) if the archive step fails — rather than silently producing a corrupt backup that looks successful.

```bash
docker stop komodo-core
docker stop komodo-mongo
tar -czf "$ARCHIVE" /mnt/cache_ssd/appdata/komodo/
TAR_STATUS=$?
docker start komodo-mongo
docker start komodo-core
if [ $TAR_STATUS -ne 0 ]; then
    echo "ERROR: tar failed with exit code $TAR_STATUS — backup is incomplete!"
    exit 1
fi
chmod 600 "$ARCHIVE"
```

5. **Verified end-to-end before trusting the schedule.** Ran the script manually first: confirmed the archive was created with correct size and `600` permissions, confirmed both containers returned to a genuinely healthy state (not just "running") by reading their post-restart logs, then scheduled it for a slot between the existing backup jobs.

---

## Prevention

- The four observability containers (Prometheus, Grafana, Loki, Alertmanager) were left unbacked-up by deliberate choice, not oversight — their data is operational/derived (metrics, dashboards, log indices) rather than source-of-truth, and is acceptable to lose and rebuild.
- The GitOps engine's own database — config, credentials, deployment history — was identified as the one component in this failure mode actually worth protecting, and now has its own verified backup path independent of the plugin.
- **Open follow-up:** no equivalent protection exists yet for the observability stack's *configuration* (alerting rules, dashboard definitions) beyond what's already tracked in git. Since the dashboards and rules are git-managed, this is likely already covered — worth a one-time confirmation rather than assuming.

---

## Lessons Learned

1. **A backup tool succeeding overall can still mean zero coverage for specific components.** The nightly job reported a clean success exit, which would have looked fully healthy in a dashboard or summary check — the real signal was buried in per-container log lines, not the job's final status. Health checks that only look at the top-level exit code would have missed this entirely.

2. **Tooling built for one deployment model (Unraid templates with bind-mount paths) doesn't automatically generalize to another (Compose-managed named volumes), even when both are "just Docker."** The fix wasn't a misconfiguration to correct — it was recognizing the tool's resolution logic had no code path for this case at all, and routing around it rather than fighting it.

3. **Not every component needs backup, and saying so explicitly is itself a decision worth documenting.** Treating "observability data is reproducible" as a deliberate, stated tradeoff — rather than silently accepting whatever a plugin happens to cover — keeps the actual risk surface honest.

---

*Environment: KONYKS-SERVER (Unraid) · Appdata Backup plugin · Komodo (GitOps engine) · Prometheus/Loki/Grafana/Alertmanager observability stack · homelab-incident-reports*
