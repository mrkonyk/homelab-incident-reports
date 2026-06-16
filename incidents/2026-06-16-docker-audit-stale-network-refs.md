# Proactive Docker Audit — Stale Network References and Orphaned Templates

Date: 2026-06-16
Severity: P3 Low
Status: Resolved
Affected: Hermes-Agent (SYSOPS.md, AGENTS.md), Honcho stack (orphaned Unraid UI templates)
Duration: Latent — stale references present since June 11 network migration; discovered and remediated same-session

## Summary

A proactive full-stack Docker audit run against KONYKS-SERVER surfaced two categories of post-migration documentation drift. First, two files in the Hermes agent config directory still referenced the frontend Docker network — which was deleted during the June 11 segmentation migration — including a hardcoded IP for the Unraid-MCP container that pointed at a network address that no longer existed. Second, Unraid UI XML templates for honcho-api and honcho-deriver were found in the Docker template directory despite both containers being designated as rebuild.sh-managed only. Neither finding caused live impact, but both represented latent failure modes: the stale IP would have caused Hermes to reference a dead endpoint, and the orphaned templates created an accidental recreation path that bypasses the sanctioned management process. Both were remediated in the same session with no service interruption.

## Timeline

Omitted — single audit session, no meaningful before/after timestamps.

## Root Cause

Stale network references: The June 11 network segmentation migration deleted the frontend network and moved all containers to named tier networks. SYSOPS.md and AGENTS.md were not updated to reflect this — both retained references to the frontend network as if it still existed. Specifically, SYSOPS.md documented the Unraid-MCP endpoint as a hardcoded Docker bridge IP on the now-deleted frontend network. The container was running correctly on tier2_apps, but the documented endpoint pointed at an address that no longer existed on any active network.

Orphaned Unraid UI templates: When honcho-api and honcho-deriver were designated as rebuild.sh-managed containers, their Unraid UI XML templates (my-honcho-api.xml, my-honcho-deriver.xml) were not removed from /boot/config/plugins/dockerMan/templates-user/. These templates remained functional — anyone with Unraid UI access could have used them to recreate the containers outside of rebuild.sh, causing config drift or overwriting the custom startup parameters that rebuild.sh manages.

## Remediation

The audit was run via the docker-auditor skill against all 30 running containers across five config scan paths. Four fixes were applied.

Fix 1 — SYSOPS.md stale Unraid-MCP endpoint: hardcoded bridge IP replaced with http://Unraid-MCP:6970/mcp. Container DNS name used instead of IP to prevent recurrence on container recreation.

Fix 2 — SYSOPS.md stale network reference: Hermes-Agent network reference updated from frontend network to tier2_apps.

Fix 3 — AGENTS.md stale network reference: updated from "frontend — EMPTY, retained but unused" to note the network was deleted during the June 11 migration.

Fix 4 — Honcho orphaned templates removed:
cp my-honcho-api.xml /tmp/my-honcho-api.xml.bak
cp my-honcho-deriver.xml /tmp/my-honcho-deriver.xml.bak
rm /boot/config/plugins/dockerMan/templates-user/my-honcho-api.xml
rm /boot/config/plugins/dockerMan/templates-user/my-honcho-deriver.xml

Backups retained at /tmp/ for the session. Full backups also at SYSOPS.md.bak.2026-06-16 and AGENTS.md.bak.2026-06-16 in the Hermes appdata directory.

## Prevention
Docker auditor skill deployed — docker-auditor is now installed in Claude Code and runs a structured five-phase audit: container inventory, tier placement, hardcoded IP scan, security constraints, and findings report. Results are saved to /mnt/user/appdata/audit/docker-audit-[date].md for diff against future runs.

Container DNS over IPs — SYSOPS.md endpoint references now use container DNS names rather than bridge IPs that change on container recreation.

Open items from this audit (deferred):
- binhex-delugevpn running privileged=true — likely reducible to NET_ADMIN + /dev/net/tun
- music-assistant running privileged=true — test targeted caps + host network
- mcp-tunnel.mjs on Edith hardcodes Unraid-MCP container IP — should resolve via DNS at tunnel-start time

## Lessons Learned

1. Migration checklists need a docs sweep step. The June 11 network migration was validated by confirming services came back online — but the agent config files that document the network topology weren't audited. Operational docs drifting from reality is a silent failure mode that only surfaces when something depends on them being accurate.

2. Management path constraints need to be enforced structurally, not just documented. Documenting "rebuild.sh only" in SYSOPS.md doesn't prevent someone from using the Unraid UI if the XML templates still exist. The fix — removing the templates — makes the constraint real rather than advisory.

3. Scheduled audits surface drift that monitoring can't. Uptime Kuma and Beszel confirmed all 30 containers healthy throughout; none of these findings would have triggered a monitor. Periodic structural audits catch a class of problems that service-level monitoring misses entirely.

*Environment: KONYKS-SERVER (Unraid) · Hermes-Agent · docker-auditor skill · homelab-incident-reports*
