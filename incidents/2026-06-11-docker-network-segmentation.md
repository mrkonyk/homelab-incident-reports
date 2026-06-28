# Docker Network Segmentation — Three-Phase Zero-Downtime Migration

**Date:** 2026-06-11
**Severity:** P2 Medium
**Status:** Resolved
**Affected:** All containers across the stack (proxy, application, database, and monitoring tiers)
**Duration:** Same session as the credential-rotation audit filed separately; executed as a deliberate follow-on hardening project

---

## Summary

Following the credential-rotation and audit work documented in a separate report from this same session, the entire container stack was migrated off a single flat network onto a tiered network architecture — separating the reverse-proxy layer, application layer, database layer, and monitoring layer from one another, so that a compromise or misconfiguration in one tier no longer has unrestricted network reach into every other tier. The migration was executed in three phases specifically designed to avoid any service downtime, rather than as a single cutover.

---

## Timeline

| Phase | Action |
|---|---|
| Phase 1 — Expand | Every container is connected to its new, tier-appropriate network while remaining simultaneously connected to the original flat network — both paths work during this phase, so nothing breaks |
| Phase 2 — Recreate | A small number of services that couldn't simply be reconnected, because of how they were originally created, are recreated directly on their target monitoring-tier network, off the old flat network entirely |
| Phase 3 — Disconnect | Every container has its connection to the original flat network removed, leaving only the new tiered networks in place; the old network is retained as an empty, unused artifact rather than deleted outright |

---

## Root Cause

This wasn't a response to a failure — it's filed as a finding from the same audit session because the flat, single-network topology it replaced was itself a standing risk identified during that audit: every container on the stack could reach every other container's exposed ports by default, regardless of whether that container had any legitimate reason to talk to it. A compromised low-value service had exactly the same network reach to a database or an SSO service as a component that was actually supposed to be talking to them.

---

## Remediation

- Four purpose-specific networks were established: one for the reverse-proxy and edge-auth components, one for general application services, one for backend datastores, and one for monitoring tools — plus an existing VPN-routed network for a media-automation stack that was left untouched since it already had its own isolation.
- All containers were migrated onto the network(s) appropriate to their actual function, with a small number of multi-role services (the reverse proxy, the SSO service) correctly placed on more than one tier where their function genuinely required reaching both.
- A supporting rebuild script for one memory/context service was updated so its two component processes launch directly onto their correct respective tiers going forward, rather than needing manual network reattachment after every rebuild.

---

## Prevention

- New services are now placed on their appropriate tier network at creation time as standing practice, rather than defaulting to a flat network and segmenting later.
- The three-phase expand/recreate/disconnect pattern used here is now the template for any future network-topology change of this kind — it's what made zero-downtime achievable, and repeating it rather than improvising a new approach each time reduces the risk of a future migration causing an outage this one avoided.

---

## Lessons Learned

1. **A flat network topology doesn't fail loudly — it just sits there as unused attack surface until something exploits it, which makes it easy to deprioritize relative to problems that are actively breaking something.** This migration only happened because an unrelated outage prompted a broad audit that surfaced it as a finding; nothing about the flat network was urgent in the way a service outage is, which is exactly why it's the kind of risk that can sit unaddressed indefinitely without a deliberate audit forcing the question.

2. **A network migration doesn't have to mean a cutover, and treating "zero downtime" as a hard requirement up front is what produces a genuinely safe migration plan rather than a risky one with a maintenance window attached.** The expand-then-recreate-then-disconnect sequence meant every container had a working network path at every single point in the migration — there was never a moment where something could only reach its dependencies over the network that was about to be removed.

---

*Environment: KONYKS-SERVER (Unraid) · Docker network segmentation · four tiered networks*
