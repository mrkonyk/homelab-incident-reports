# Frigate's no-login API was reachable from 13 containers

**Date:** 2026-09-15
**Severity:** P1 High
**Status:** Resolved
**Affected:** frigate (ollama and swag changed as part of the fix)
**Duration:** Unknown. The exposure existed for as long as Frigate sat on `tier2_apps`; it was found and closed on 2026-09-15.

---

## Summary

During a security review of the camera setup, Frigate's internal API turned out to be reachable from every container on the shared `tier2_apps` Docker network. That API has no authentication and grants full control of the NVR, including live video and deleting recordings. The port was not published to the host, so a host-level port audit showed nothing wrong. Thirteen other containers shared the network, five of them internet-facing (SWAG, Nextcloud, Immich, Emby, Vaultwarden), which put the cameras one hop from any compromise of those services. Frigate was moved to a new dedicated network, `tier_nvr`, shared only with the two containers that need it, and the change was verified at runtime.

---

## Timeline

| Step | Action |
|---|---|
| Discovery | Security review of the camera setup found Frigate's internal API (port 5000, no authentication) reachable from every container on `tier2_apps`. |
| Scoping | `docker network inspect tier2_apps` listed the network's actual members; checked against which of them had a real need to reach Frigate — only `ollama` and `swag` did. |
| Network created | `tier_nvr` created by hand as a dedicated bridge network with its own /24, alongside the existing tier ranges. |
| Compose updated | `frigate`, `ollama`, and `swag` stacks edited in the GitOps repo to declare `tier_nvr` as `external: true`. |
| Deployed | Rolled out through Komodo in dependency order — ollama → swag → frigate — so Frigate never moved before SWAG could reach it on the new network. |
| Verified | Network membership, AI descriptions, and Frigate reachability confirmed on the host after the deploys. |

---

## Root Cause

Frigate serves two APIs: an authenticated one on port 8971 and an unauthenticated internal one on port 5000. Docker user-defined bridge networks do not filter traffic between members. Any container on the network can reach any port on any other member by container name, whether or not that port is published. `tier2_apps` had grown into the general application tier with 14 members, so Frigate's internal API was effectively shared with all of them.

The exposure was scoped by listing the network's actual members (`docker network inspect tier2_apps`) and then asking which of them had a real need to reach Frigate. Only two did: ollama, which generates Frigate's AI event descriptions, and SWAG, which proxies remote access to Frigate's authenticated UI behind Authelia. Home Assistant reaches Frigate through its published authenticated port, not through Docker networking.

---

## Remediation

1. Created `tier_nvr` by hand as a bridge network with its own /24 alongside the existing tier ranges. It is not marked internal: it is Frigate's only network, and Frigate must reach the camera VLAN and the MQTT broker outside Docker.
2. Edited three compose stacks in the GitOps repo, declaring `tier_nvr` as `external: true` in each. Without that flag, Compose silently creates a separate per-project network and the services would not share one.
   - frigate: `tier2_apps` replaced with `tier_nvr`, so Frigate is on `tier_nvr` only.
   - ollama: `tier_nvr` added; it stays on `tier2_apps` for its other clients.
   - swag: `tier_nvr` added, keeping remote Frigate access behind Authelia.
3. Deployed through Komodo in the order ollama → swag → frigate, so Frigate never moved before SWAG could reach it on the new network.
4. Updated the network lists in the repo documentation and in the Hermes agent's operations notes.

**Verification** (on the host, after the deploys):
- `docker network inspect tier_nvr` lists exactly frigate, ollama and swag.
- From frigate, ollama's version endpoint answers, so AI descriptions still work.
- frigate no longer appears among `tier2_apps` members (count 0).
- From swag, Frigate's authenticated port answers 401 (reachable, login required).
- Both cameras still stream in Home Assistant.

---

## Prevention

- The recreate commands for all hand-made tier networks are now recorded in the ops documentation. No script creates these networks, so the documentation is the recovery path if Docker's image file is ever lost.
- Naming convention made explicit: the numbered tiers (`tier1_proxy`, `tier2_apps`, `tier3_backend`) form the edge → apps → data ladder, and the un-numbered tiers (`tier_monitoring`, `tier_nvr`) are purpose-built segments outside it.
- Open: review `tier2_apps` for other services that rely on network membership as their only access control.
- Open: the agent-facing network list was stale beyond this change (missing members, an outdated container name); a correction is pending.

---

## Lessons Learned

1. **"Not published" is not "not reachable."** Host port mappings only describe exposure to the LAN. A port audit has to include container-to-container reachability on every shared network.
2. **A shared network is a trust zone, so size it by need.** Listing the real members and the real consumers turned a vague concern into a concrete blast radius (13 containers, 5 internet-facing) and a concrete requirement (2).
3. **Verify at the runtime layer, and order deploys by dependency.** The repo diff showed intent; network membership on the host was the proof. Moving the dependent service last avoided a remote-access outage during the change.

---

*Environment: KONYKS-SERVER (Unraid) · Frigate · Docker networking · Komodo GitOps · SWAG/Authelia · homelab-incident-reports*
