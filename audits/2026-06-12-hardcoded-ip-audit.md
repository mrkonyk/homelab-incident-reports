# Audit: Hardcoded Docker IP Scan — 2026-06-12

## Overview

| Field | Value |
|-------|-------|
| Date | 2026-06-12 |
| Type | Preventive audit |
| Status | Resolved — fixes applied same session |
| Auditor | Claude Code (read-only scan, then targeted fixes) |

## Trigger

Three prior incidents shared the same root cause: a hardcoded Docker bridge IP
broke when the target container restarted and received a new address. This audit
was initiated to find all remaining instances before a fourth incident
occurred.

## Scope

- All 29 running containers: env vars and healthcheck commands
- All nginx proxy-conf files
- All Unraid container templates
- All user scripts
- Config files for all major services
- Unraid Docker templates (live and backup)

## Result

Zero active hardcoded bridge IPs found that would break on restart.

The June 11 network restructure (flat → tiered segmentation) was confirmed
complete and clean. All containers are on their correct networks. No live
template references the decommissioned flat network.

## Findings

### MEDIUM — Stale subnet in SWAG reverse-proxy real-IP config

- File: SWAG nginx real-IP trust config
- Issue: A /16 subnet from the decommissioned flat network remained in the
  Cloudflare real-IP trust list. No container currently has an IP in this range,
  so there was no active failure. Risk: if Docker reused the subnet for a future
  network, those containers' X-Forwarded-For headers would be unconditionally
  trusted — a potential IP-spoofing vector.
- Fix applied: Removed the stale entry. Added two explicit entries covering
  the current tiered network subnets. Nginx config tested (nginx -t) and
  reloaded (nginx -s reload). Verified on disk.

### LOW — Orphaned dev config for a security service

- File: A development-time alternate database config for CrowdSec
- Issue: File pointed at an IP on the Docker default bridge (which has no
  containers attached). The service's live config uses SQLite and never
  references this file. Completely inert but confusing for future audits.
- Fix applied: File deleted. No service restart required.

### INFORMATIONAL — Historical log artifacts

- Session/request dump logs from before the June 11 migration contain old
  bridge IPs. These are timestamped diagnostic records, not active config.
  No action required; may be purged as part of routine log rotation.

### INFORMATIONAL — Decommissioned project appdata

- Source code and dev-time config for a Nextcloud integration project that
  was evaluated but never deployed. No containers running. No operational risk.
  Low-priority disk-space cleanup candidate.

## Timeline
| Time | Action |
|------|--------|
| 2026-06-12 | Full read-only audit completed |
| 2026-06-12 | MEDIUM finding fixed: stale subnet removed, SWAG reloaded |
| 2026-06-12 | LOW finding fixed: orphaned dev config deleted |
| 2026-06-12 | Audit report saved to server appdata for future diff comparison |

## Lessons Learned

- Subnets from decommissioned Docker networks should be removed from all
  referencing configs (not just the network itself) as part of network teardown.
- Broad set_real_ip_from entries should be reviewed whenever networks are
  restructured.
- Dev/alternate config files for production services should either be clearly
  named .example or not committed to the appdata directory.
