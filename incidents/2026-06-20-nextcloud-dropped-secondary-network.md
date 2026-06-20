# Nextcloud Database Connectivity Loss — Dropped Secondary Network + Stale PHP-FPM DNS Cache

**Date:** 2026-06-20
**Severity:** P2 Medium
**Status:** Resolved
**Affected:** Nextcloud, MariaDB (tier3_backend)
**Duration:** Unknown onset — discovered same-day, resolved within one session (~30 min from diagnosis to fix)

---

## Summary

Nextcloud lost the ability to reach its MariaDB backend after a standalone container recreate dropped its
`tier3_backend` secondary network attachment without a corresponding array restart, so the `network-reattach`
startup script never re-ran to restore it. Reconnecting the network alone did not resolve the issue — the
running PHP-FPM process had already cached a failed DNS lookup for the `mariadb` hostname from before the
network was reattached, so the application layer continued failing even after raw network connectivity was
restored. A container restart cleared the stale resolution and fully resolved the issue. This is a different
root cause from the earlier `2026-06-11-nextcloud-hardcoded-db-host.md` incident (hardcoded IP vs. dropped
network attachment) but the same downstream symptom — Nextcloud unable to reach its database — making this
the second distinct failure mode to produce that signature.

---

## Timeline

| Time | Event |
|------|-------|
| — | Nextcloud container recreated at some prior point without a following array restart |
| Same session | User reports inability to connect to Nextcloud |
| Same session | Diagnosis begins: `docker logs`, network inspection across `tier2_apps`/`tier3_backend` |
| Same session | Root cause identified: `nextcloud` missing `tier3_backend` secondary attachment (confirmed via full secondary-network audit against `network-reattach` script's expected state) |
| Same session | `docker network connect tier3_backend nextcloud` applied |
| Same session | `ping mariadb` succeeds from inside the container, but Nextcloud app logs continue showing `getaddrinfo for mariadb failed: Name does not resolve` |
| Same session | `docker restart nextcloud` applied |
| Same session | Logs clean on restart; `getent hosts mariadb` resolves correctly; web UI confirmed loading |

---

## Root Cause

A full secondary-network audit was run against the expected state encoded in the `network-reattach` User
Script, comparing every container's documented secondary network attachment to its live state. Out of 16
checked attachments, exactly one gap was found: `nextcloud` was missing its `tier3_backend` secondary
connection, while its `tier2_apps` primary remained intact.

The `network-reattach` script only executes "At Startup of Array" — it has no trigger for standalone
container-level recreates. When Nextcloud was rebuilt independently (not as part of a full array restart),
its secondary network connection was dropped and nothing re-applied it, since the only mechanism that does
so never ran. This is the same class of gap that previously caused SWAG to lose its `tier2_apps` attachment
during separate infrastructure work, and that affected seven containers after a Docker daemon restart on
2026-06-11 — a recreate or daemon-level restart event that isn't a full array startup leaves any
script-managed secondary network attachment vulnerable to silently dropping.

A second, distinct factor compounded the issue: `docker network connect` on a *running* container updates
that container's network namespace and embedded DNS immediately, but it does not force already-running
processes inside the container to redo failed DNS lookups. Nextcloud's PHP-FPM workers had attempted and
cached a failed resolution for `mariadb` prior to the network being reattached. As a result, raw network
connectivity (`ping`) succeeded immediately post-fix, while the application layer continued reporting
`SQLSTATE[HY000] [2002] php_network_getaddresses: getaddrinfo for mariadb failed: Name does not resolve`
until the container itself was restarted and PHP-FPM re-initialized its DNS resolution from a clean state.

---

## Remediation

1. Ran a full secondary-network audit comparing the `network-reattach` script's expected attachments
   against live `docker network inspect` output for every affected container.
2. Identified `nextcloud` as the sole container missing its `tier3_backend` secondary attachment.
3. Reattached the network without disrupting the running container:
```bash
docker network connect tier3_backend nextcloud
```
4. Verified raw connectivity — succeeded immediately:
```bash
docker exec nextcloud ping -c 2 mariadb
```
5. Application-layer errors persisted in `docker logs nextcloud` despite working ping, indicating a
   process-level stale DNS cache rather than a network-layer problem.
6. Restarted the container to force PHP-FPM to re-resolve cleanly:
```bash
docker restart nextcloud
```
7. Confirmed resolution and recovery:
```bash
docker exec nextcloud getent hosts mariadb
# <mariadb-tier3-ip>  mariadb  mariadb
```
8. Verified clean startup in logs (no database connection errors) and confirmed the Nextcloud web UI
   loaded successfully.

---

## Prevention

- **Documented pattern**: this is the third confirmed instance of a standalone container recreate or
  daemon-level restart dropping a secondary network attachment without the array-startup-only
  `network-reattach` script catching it (prior instances: SWAG losing `tier2_apps`; seven containers
  losing `frontend`-era attachments after a 2026-06-11 daemon restart).
- **Diagnostic pattern documented**: when a container can `ping` a dependency by name but the application
  still reports DNS resolution failure, check whether the network was attached to an *already-running*
  process rather than from container start — the fix in that case is a restart, not just a reconnect.
- **Open follow-up**: the `network-reattach` script's trigger (array startup only) is a known structural
  gap. A standing drift-detection check — comparing live `docker network inspect` state against the
  documented secondary-attachment table on a recurring schedule, independent of array restarts — has been
  proposed as a future improvement rather than continuing to catch this reactively.

---

## Lessons Learned

1. **A startup-triggered remediation script has a blind spot for any event that isn't a startup.**
   `network-reattach` is correct in design but incomplete in coverage — it assumes the only way a
   secondary network gets dropped is array restart, when standalone recreates and daemon restarts produce
   the identical failure with no startup event to trigger the fix.
2. **Network-layer connectivity and application-layer connectivity are not the same signal, and conflating
   them costs diagnostic time.** `ping` succeeding after a network reconnect can mask the fact that a
   long-running process inside the container cached an earlier failure. The correct test after any live
   network change to a running container is the application's own behavior, not just raw reachability.
3. **Recurring identical failure modes are a signal to invest in detection, not just repeated manual
   remediation.** Three instances of the same root cause (dropped secondary network outside array startup)
   is enough evidence to justify moving from reactive fixes to a standing drift check.

---

*Environment: KONYKS-SERVER (Unraid) · Nextcloud · MariaDB · Docker tiered networking (tier2_apps/tier3_backend) · homelab-incident-reports*
