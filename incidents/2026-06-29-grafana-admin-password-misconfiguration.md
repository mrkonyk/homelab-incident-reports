# Grafana Admin Password Misconfiguration — Secret Mount Was a Directory, Password Reset on Every Restart

**Date:** 2026-06-28 to 2026-06-29
**Severity:** P2 Medium
**Status:** Resolved
**Affected:** obs-grafana (Grafana 13.0.2), GF_SECURITY_ADMIN_PASSWORD_FILE secret mount, fetch_grafana_secrets.sh
**Duration:** Condition present since Grafana was migrated to secrets-managed configuration; exact onset not independently verified

---

## Summary

During work to build monitoring coverage for a set of infrastructure gaps, investigation of the
Grafana deployment found that the admin password secret mount was misconfigured: the path pointed to
by `GF_SECURITY_ADMIN_PASSWORD_FILE` was a directory, not a file. Grafana 13.x reads this
environment variable on every startup and resets the admin password from the file it references —
not only on first initialization. Because the referenced path was a directory, the password was not
being set from the secret on any restart. A second bug compounded the issue: the script responsible
for populating the secret file was writing to the wrong destination path, so the correct file had
never been created. Fixing both bugs required rewriting the destination path, removing the stale
directory, running the population script, and fully recreating the container rather than just
restarting it, because Docker's overlay had recorded the directory and rejected a plain restart once
the mount type changed.

---

## Timeline

| Time | Event |
|------|-------|
| 2026-06-28 | Monitoring gap audit identifies Grafana admin password secret path as unverified |
| 2026-06-28 | Inspection of the mount path reveals a directory at the location expected to be a file |
| 2026-06-28 | `GF_SECURITY_ADMIN_PASSWORD_FILE` confirmed to point at the directory path |
| 2026-06-28 | Grafana 13.x behavior confirmed: reads `GF_SECURITY_ADMIN_PASSWORD_FILE` on every startup, not only on first run — persistent misconfiguration, not a one-time initialization miss |
| 2026-06-28 | `fetch_grafana_secrets.sh` reviewed; DEST path found to write to `repos/homelab-infra/stacks/observability/secrets/` — the read-only Komodo repository clone, not the Komodo deploy path |
| 2026-06-28 | Correct destination identified: `stacks/observability/stacks/observability/secrets/` (the Komodo-managed deploy checkout) |
| 2026-06-28 | DEST path corrected in `fetch_grafana_secrets.sh`; committed as `732edb3` |
| 2026-06-28 | Stale directory removed; `fetch_grafana_secrets.sh` executed; secret file created with content from secrets manager |
| 2026-06-28 | `docker restart obs-grafana` fails: `unable to start container process: error mounting ... not a directory` — container overlay retains the directory from the prior mount |
| 2026-06-28 | Container fully recreated with `docker stop` + `docker rm` + `docker compose up -d grafana`; overlay cleared |
| 2026-06-28 | Verified: `curl -s -u admin:<secret-value> http://<host>:<port>/api/user` returns HTTP 200 with valid user data |

---

## Root Cause

Two bugs combined to produce the misconfiguration.

**Bug 1 — Wrong destination path in `fetch_grafana_secrets.sh`:** The script that populates the
Grafana admin password secret file was writing to a path inside the read-only Komodo repository
clone (`repos/homelab-infra/stacks/observability/secrets/`). The correct destination is inside the
Komodo deploy checkout, which has the nested path
`stacks/observability/stacks/observability/secrets/`. The repository clone is read-only from the
host's perspective and is periodically overwritten by Komodo; files written there by other scripts
do not persist and would not have been seen by the Grafana container. Because the script ran without
error (the path was writable), no indication of the wrong destination was ever surfaced.

**Bug 2 — Directory at the mount path:** At the location where the secret file was expected, a
directory existed instead — a likely artifact of an earlier Docker secrets mount pattern that created
directories as placeholder mounts. `GF_SECURITY_ADMIN_PASSWORD_FILE` in Grafana 13.x is read on
every container startup, not only on first initialization. With a directory at the referenced path,
Grafana cannot read a password from it; the exact fallback behavior (whether it silently ignores the
env var or uses a hardcoded default) was not tested exhaustively, but the effect was that the admin
password was not being managed by the secret on any restart.

A third issue compounded the remediation: `docker restart` does not clear the container's overlay
filesystem. Once Docker had recorded the mount path as a directory in the container's layer, changing
the host-side path to a file and restarting was not sufficient — Docker rejected the restart with a
`not a directory` error because its stored overlay state conflicted with the new mount type. A full
`stop` + `rm` + `compose up` was required to clear the overlay and let Docker re-mount from the
corrected host path.

---

## Remediation

1. Identified the wrong destination path in `fetch_grafana_secrets.sh` by comparing the path in the
   script against the actual Komodo deploy directory structure. The nested path
   (`stacks/<stack>/stacks/<stack>/secrets/`) is an artifact of how Komodo checks out the repo inside
   the deploy directory.
2. Corrected the DEST variable in the script and committed the fix.
3. Removed the stale directory at the mount path and ran the corrected script to populate the secret
   file with the actual admin password from the secrets manager.
4. Fully recreated the container (stop, rm, compose up) rather than restarting, because Docker's
   overlay retained the directory mount from the prior container lifecycle.
5. Verified the fix by authenticating against the Grafana API directly with the value read from the
   secret file — not by checking Grafana's UI or trusting that the container came up cleanly.

---

## Prevention

- `GF_SECURITY_ADMIN_PASSWORD_FILE` is now populated from the secrets manager before the container
  starts; the file exists at the correct path in the correct deploy directory. The script path is
  committed and tested.
- The behavior of `GF_SECURITY_ADMIN_PASSWORD_FILE` in Grafana 13.x (reads on every restart, not
  only initial provisioning) is now documented as a constraint in the secrets-management notes for
  this deployment. This differs from the behavior documented in some earlier Grafana versions and
  from how similar env vars behave in other tools.
- Any future change to a container's bind-mount configuration (type, path, or file vs. directory)
  should be accompanied by a full `stop` + `rm` + `compose up`, not just a restart, when the mount
  path has previously been used with a different type.

---

## Lessons Learned

1. **A script that writes to the wrong path and exits 0 has produced no evidence of success — only evidence that it ran without an error.** `fetch_grafana_secrets.sh` completed normally on every execution. Its output path was writable. Nothing in its own exit code or output indicated that the file it created would not be seen by the container it was meant to configure. The discrepancy between `repos/` (read-only clone) and `stacks/` (deploy checkout) was only visible by reading both paths and comparing them.

2. **Docker's container lifecycle distinguishes between restart and recreation in ways that are not always visible until they matter.** A `docker restart` preserves the container's overlay filesystem; a `stop` + `rm` + `compose up` does not. When a bind-mount path changes type (directory to file), restart will fail or behave unexpectedly because the overlay records the prior mount type. This is a difference between "restart the process" and "recreate the container from its definition" that is easy to miss until it causes a concrete failure.

3. **Verifying a configuration change means checking the behavior it was supposed to produce, not just the absence of error during the change.** The fix was not considered complete until a real authenticated API call confirmed the password was being read from the secret file — not when the container came up without errors, not when the secret file had the right content on disk, and not when the Grafana UI showed a login page. Each of those intermediate signals would have been consistent with the fix being correct or with it still being broken in a new way.

---

*Environment: KONYKS-SERVER (Unraid) · Grafana 13.0.2 · Komodo GitOps · Infisical secrets manager · homelab-incident-reports*
