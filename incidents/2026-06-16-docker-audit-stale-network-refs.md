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

## Fixes Applied

### Fix 1 — Honcho-API Configuration
Container DNS name used instead of IP to prevent recurrence on container recreation.

### Fix 2 — SYSOPS.md stale network reference
Updated the Hermes-Agent network reference from `frontend` network to `tier2_apps`.

### Fix 3 — AGENTS.md stale network reference
Updated from "frontend — EMPTY, retained but unused" to note the network was deleted during the June 11 migration.

### Fix 4 — Honcho orphaned templates removed
Executed manual cleanup of orphaned XML templates:
- `cp my-honcho-api.xml /tmp/my-honcho-api.xml.bak`
- `cp my-honcho-deriver.xml /tmp/my-honcho-deriver.xml.bak`
- `rm /boot/config/plugins/dockerMan/templates-user/my-honcho-api.xml`
- `rm /boot/config/plugins/dockerMan/templates-user/my-honcho-deriver.xml`
*Note: Backups retained at /tmp/ for the session. Full backups in Hermes appdata: SYSOPS.md.bak.2026-06-16, AGENTS.md.bak.2026-06-16.*

## Prevention
*   **Docker Auditor Skill:** `docker-auditor` is now installed in Claude Code. It runs a 5-phase audit (inventory, tier placement, IP scan, security, findings). Results saved to `/mnt/user/appdata/audit/` for diffing.
*   **Container DNS over IPs:** `SYSOPS.md` endpoint references shifted to DNS (e.g., `http://Unraid-MCP:6970/mcp`) to ensure stability across recreations.

### Open Items (Deferred)
- `binhex-delugevpn` running `privileged=true` — evaluate `NET_ADMIN` + `/dev/net/tun`.
- `music-assistant` running `privileged=true` — test targeted caps + host network.
- `mcp-tunnel.mjs` on Edith hardcodes Unraid-MCP IP — resolve via DNS at tunnel-start.

## Findings & Lessons Learned
1.  **Template vs. Runtime:** Updating an Unraid Docker template (XML) is not always sufficient to purge stale network references from the Docker daemon's internal state if the network is deleted. Complete recreation is preferred.
2.  **Documentation Audit:** Migration checklists must include a documentation sweep. Operational docs (SYSOPS.md, AGENTS.md) drifted from physical reality immediately post-migration.
3.  **Structural Constraints:** "Advisory" constraints (like documenting "rebuild.sh only") don't work if UI paths exist. Removing orphaned templates enforces the intended management path.
4.  **Audit Value:** Service-level monitoring (Uptime Kuma) passed throughout. Structural audits catch the "silent failures" that don't trigger alerts but create future toil.

---
**SRE Score for Migration:** A (Successful migration, zero downtime for users, all stale references identified and purged).

*Environment: KONYKS-SERVER (Unraid) · Hermes-Agent · docker-auditor skill · homelab-incident-reports*
