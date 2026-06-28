---

# Incident Report: CrowdSec Silent No-Op + Unauthenticated LAN Service Exposure

Date: 2026-06-11
**Severity:** P1 High
Status: Resolved
System: KONYKS-SERVER (Unraid 7.3.1)

## Summary

A full Docker stack audit revealed two compounding security failures: CrowdSec had been installed but was a complete no-op since deployment (bouncer never wired to SWAG, acquis.yaml broken), and multiple services had host ports exposed to the LAN with no authentication. Both issues had been silently present for weeks.

## Root Cause

CrowdSec: The acquis.yaml log acquisition config was a broken stub — CrowdSec was running but not reading any logs and not making any blocking decisions. The SWAG bouncer was never installed, so even if CrowdSec had been detecting threats, no blocking would have occurred. CrowdSec showed as "healthy" in Uptime Kuma throughout.

LAN exposure: A 30-container Docker audit identified 17 findings. Several containers had host port mappings that exposed management interfaces directly to the LAN — including database ports and internal tool UIs — with no authentication layer in front of them.

## Timeline

- Weeks prior — CrowdSec installed from Community Apps; assumed operational
- June 11 — Full Docker stack audit initiated via Claude Code over SSH
- Audit discovery — CrowdSec acquis.yaml identified as broken stub; bouncer-swag absent from SWAG config; 17 container findings identified including unauthenticated host port exposures
- Pass 1–5 — CrowdSec rebuilt: acquis.yaml rewritten, SWAG bouncer installed and confirmed pulling decisions, custom sshd parser added for newer OpenSSH format, all nginx confs updated, Telegram alerting configured
- Container hardening — MariaDB, Redis, and CrowdSec host ports removed; Krusader VNC ports removed and routed through SWAG + Authelia 2FA; Unraid-MCP restricted to Docker-internal only; adminer permanently removed; 11 containers set to restart: unless-stopped
- Resolved — 70 active CrowdSec scenarios confirmed, bouncer-swag validated, all priority container findings closed

## Resolution

CrowdSec required a full five-pass rebuild:
1. acquis.yaml rewritten with correct log source paths
2. SWAG bouncer key generated via cscli bouncers add bouncer-swag, DOCKER_MODS and CROWDSEC_LAPI_URL added to SWAG
3. Custom sshd parser deployed for newer OpenSSH rdomain log format
4. All SWAG nginx conf files updated to current spec; Immich port corrected to 2283
5. Telegram notifications wired and confirmed

Container hardening removed all unnecessary host port mappings and routed previously direct-access services through SWAG + Authelia SSO.

## Prevention

- Verify tool wiring, not just tool health. CrowdSec reported healthy in monitoring but was never actually blocking anything. A "container is running" check is not the same as "tool is doing its job."
- Bouncer installation is a separate step. CrowdSec without a bouncer is a detection system with no enforcement. Confirm cscli bouncers list shows a valid bouncer after any IDS/IPS install.
- Audit host port mappings as part of any new container deployment. Default Community Apps templates often include host port mappings that are unnecessary once services are behind a reverse proxy.
- Principle of least exposure. If a service is accessible through SWAG + Authelia, its host port mapping should be removed. Management interfaces that don't need external access should not be exposed to the LAN.

## Related

- Docker stack audit: 17 findings identified, all priority items resolved same session
- PostgreSQL_Immich isolated off frontend network into dedicated backend tier
- Gmail app password rotated; Authelia config updated same session

---