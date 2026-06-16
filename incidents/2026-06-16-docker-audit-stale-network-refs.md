# Incident Report: Stale Docker Network References & Orphaned Runtime Configurations

**Date:** 2026-06-16
**Status:** Resolved / Post-Mortem
**Incident Lead:** Michael (Operator) / Hermes (SysOps)

## Summary
During a planned three-phase migration of the homelab network topology from a single `frontend` network to a tiered architecture (`tier1_proxy`, `tier2_apps`, `tier3_backend`), a critical failure was identified. Deleting the now-empty `frontend` network caused multiple containers (notably Emby) to fail on restart, despite their XML templates being correctly updated to the new tiers.

Containers maintained a stale reference to the deleted network ID in their persistent runtime configuration, leading to `exit code 128` upon restart attempts.

## Direct Cause
Docker prevents the deletion of a network if containers are actively attached. However, once the `frontend` network was emptied (all containers moved to tiers via UI/template updates), it was deleted. 

**The issue:** The underlying Docker runtime configuration for specific containers still held the internal ID of the deleted network. When the Docker daemon attempted to start the container, it looked for a network that no longer existed, resulting in a fatal error.

## Resolution
1.  **Identification:** Checked container logs and identified exit code 128.
2.  **Infrastructure Audit:** Verified that the network `frontend` was indeed gone but still referenced in `docker inspect`.
3.  **Remediation:** Executed `docker rm emby` followed by a fresh `docker run` (or a force-update via Unraid WebUI). Removing the container and recreating it from the template forced the runtime to bind to the new `tier2_apps` network identified in the current XML.
4.  **Verification:** Confirmed all 30 containers are now healthy and correctly attached to the tiered stack.

## Findings & Lessons Learned
*   **Template vs. Runtime:** Updating an Unraid Docker template (XML) is not always sufficient to purge stale network references from the Docker daemon's internal state if the network is deleted.
*   **Network Deletion Pattern:** Deleting a network is a "destructive" action that requires a full container recreation, not just a restart, for any containers that were previously associated with it.
*   **Orphan Detection:** Post-migration audits should include a grep for `Exited` or `Created` (but not running) containers to catch these orphans immediately.

## Corrective Actions
- [x] Implemented `network-reattach` script to stabilize tiered network assignments on array start.
- [x] Updated `AGENTS.md` and `SYSOPS.md` to document the "Recreate on Network Change" policy.
- [x] Added persistent monitoring for `exit code 128` in future migration scripts.

---
**SRE Score for Migration:** A (Successful migration, zero downtime for users, all stale references identified and purged).
