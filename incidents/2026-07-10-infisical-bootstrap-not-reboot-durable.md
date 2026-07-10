# Infisical Client-Auth Not Reboot-Durable — Every Secrets Consumer Silently Fragile

**Date:** 2026-07-10  
**Severity:** P1 High  
**Status:** Resolved (schedule UI-confirmed; end-to-end reboot verification pending)  
**Affected:** All Infisical-authenticating automation on KONYKS-SERVER — git push-PAT rotation (`do_push_rotation.sh`), plus `fetch_authelia_secrets.sh`, `fetch_swag_secrets.sh`, `fetch_loki_secrets.sh`, and the `rotate-*` scripts  
**Duration:** Latent since the machine-identity was created (2026-06-27); surfaced 2026-07-10

## Summary
A git push to `mrkonyk/homelab-infra` failed because the push script could not authenticate to the on-box Infisical instance. Investigation showed the credential file the script sources, `/root/.infisical_env`, was absent. 

**Root cause:** `/` on Unraid is a RAM-backed rootfs rebuilt from the flash boot device every boot, so anything written to `/root` is wiped on reboot. A boot-time hook restored the Infisical CLI binary but nothing restored the client-auth it depends on. 

Crucially, the gap was not specific to the push flow — every Infisical-authenticating script on the host sourced the same ephemeral file, so all of them were one reboot away from silent failure; the push simply happened to be the first consumer to run and fail after a reboot. Remediation moved the machine-identity credential to a persistent appdata store and installed a First-Array-Start hook that re-materializes `/root/.infisical_env` from it. The fix un-broke the entire consumer class, not just the push, and the four stacked commits (guard hardening + three Trivy commits) were subsequently published.

## Timeline
| Time | Event |
| :--- | :--- |
| — | Machine identity `konyks-scripts` created; client-auth written to `/root/.infisical_env` (ephemeral, unknown at the time) |
| — | Boot hook added to restore the Infisical CLI binary only — auth restoration never added |
| **Session start** | `git push` fails; push script cannot authenticate |
| **+recon 1** | `/root/.infisical_env` confirmed absent; `/` confirmed RAM-backed rootfs; boot hook restores binary but not auth |
| **+recon 2** | Push script identified as consumer; but found all Infisical scripts source the same file — gap is class-wide, not push-specific |
| **+recon 3** | Client-secret confirmed to have no persistent home anywhere on the box; only viable fix is durable persistence |
| **+fix** | Durable store created in appdata (600, root:root); fresh client-secret rolled on existing identity; domain/project/env values reconstructed |
| **+verify** | First auth test: login OK, but secret-fetch failed — traced to two missing scoping vars (project-id, env), not a permission problem |
| **+verify** | Durable store completed to five variables; auth + secret-fetch both green reading from the file |
| **+hook** | First-Array-Start hook installed; proven via wipe-and-restore without a reboot |
| **+push** | PAT rotation script ran (rotated + propagated); separately, `git push` published the four commits — branch now in sync with origin |

## Root Cause
`/root/.infisical_env` is sourced by every Infisical-authenticating script on KONYKS-SERVER to obtain the Universal Auth machine-identity credential. On Unraid, `/` is a RAM-backed rootfs reconstructed from the flash boot device on every boot — so any file written under `/root` is non-persistent by design, not by accident.

A hook in the Unraid `go` file (added 2026-06-27) copied the Infisical CLI binary into place at boot, which created a false sense that Infisical was "set up to survive reboot." It was not: the binary persisted, the auth did not. Nothing — no `go` step, no `@reboot` cron, no User Scripts entry — regenerated `/root/.infisical_env` after a wipe.

Because the credential half of the setup was silently missing after any reboot, the failure was invisible until a consumer actually ran. The push flow was the first to do so and failed on authentication. Two secondary findings shaped the fix:
1. **The gap was class-wide.** `do_push_rotation.sh`, all three `fetch_*_secrets.sh`, and the `rotate-*` scripts source the same file. They had not failed only because they had not been invoked since the last reboot — every one of them was equally fragile.
2. **The reconstructed credential file was initially incomplete.** The login credential (client-id/secret/domain) was sufficient to authenticate but not to locate a secret. The consumer scripts fetch with explicit project-id, environment, and path scoping flags. The first rebuild omitted project-id and environment, producing a login-succeeds / fetch-fails signature that was initially misread as a permissions issue before being correctly traced to the two missing scoping variables.

## Remediation
1. **Durable credential store.** Created a dedicated directory outside any git checkout, on persistent appdata (XFS, real per-file permissions), holding the full five-variable machine-identity credential set: `/mnt/user/appdata/infisical-bootstrap/client-auth.env` (chmod 600, root:root). Appdata was chosen over the flash boot device deliberately: the flash is FAT32 (no real per-file permissions) and is subject to auto-zipped flash backups that could carry a plaintext credential off-box.
2. **Fresh client-secret.** A new client-secret was rolled on the existing identity (`konyks-scripts`) via the Infisical UI. 
3. **Completed the credential set.** After an initial login-OK / fetch-FAIL, the store was completed with the two missing scoping variables (`project-id` and `environment`) confirmed against the actual secret path.
4. **First-Array-Start hook.** Installed a User Scripts hook that re-materializes the ephemeral file from the durable store at array start:
   ```bash
   SRC=/mnt/user/appdata/infisical-bootstrap/client-auth.env
   DST=/root/.infisical_env
   if [ -f "$SRC" ]; then
     cp "$SRC" "$DST" && chmod 600 "$DST" && chown root:root "$DST"
     logger -t infisical-bootstrap "restored env from durable store"
   else
     logger -t infisical-bootstrap "ERROR: durable store missing at array start"
   fi
   ```
5. **Proved durability without a reboot.** The array-start condition was simulated: the live copy was deleted, the hook run by hand, and the restored file confirmed to contain all five variables at correct permissions.
6. **Published the stack.** The PAT rotation script ran and propagated the rotated token; the four pending commits were then published with a separate `git push`.

## Prevention
*   Durable, correctly-located credential store replaces the ephemeral file as the source of truth.
*   Boot hook now covers auth, not just the binary.
*   Proof-without-reboot pattern established for verifying array-start hooks.
*   Blast radius flagged for a future split: separating the write-scoped push identity from the read-only config identity.

## Lessons Learned
*   A boot hook that restores part of a dependency is more dangerous than no hook at all. Partial provisioning defeats observability.
*   "It authenticated" and "it can do the thing" are different assertions. Testing auth and authorization as separate checks is critical.
*   Verify what a script does by reading it, not by trusting its name. Conflating rotation with pushing commits in the plan caused unnecessary confusion.
*   Sequencing a safety-net removal matters. Removing the fallback secret before a real reboot confirmed the hook fires at array-start collapsed the margin the deferral was meant to protect.

**Environment:** KONYKS-SERVER (Unraid) · Infisical (self-hosted) · User Scripts · homelab-infra · homelab-incident-reports