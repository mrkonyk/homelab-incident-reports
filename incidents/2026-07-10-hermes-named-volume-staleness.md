# Hermes Update Silently No-Ops: Runtime Code in a Populated Named Volume

**Date:** 2026-07-10  
**Severity:** P2 Medium  
**Status:** Diagnosed — Remediation Planned  
**Affected:** Hermes-Agent (deployment / update path)  
**Duration:** Latent ~6 weeks (runtime code frozen 2026-05-28, discovered 2026-07-10)

---

## Summary

During an audit of the Hermes-Agent stack, the agent's runtime Python code was
found to be served from a *populated named Docker volume* rather than from the
container image. Docker seeds a named volume's contents only at first creation
and never re-seeds an already-populated volume when a container is recreated —
so a routine image pull and redeploy on 2026-07-05 silently continued running
the code baked into the volume when it was created on 2026-05-28, roughly 149
commits (eight upstream releases) stale. The standard GitOps update pattern used
elsewhere in this homelab (bump the image tag, redeploy via Komodo) is therefore
a silent no-op for this container. The vendor-supported update path — an
in-container git-pull-and-reinstall — was identified; a live update remains
pending resolution of an open question about which of two on-disk checkouts that
command targets. No configuration or data was at risk: Hermes state lives on a
separate host bind mount, untouched by any of these paths.

---

## Root Cause

The Hermes container serves its runtime code from `/opt/hermes`, which is mounted
from a named Docker volume created on 2026-05-28. The container image was
subsequently re-pulled and the container recreated on 2026-07-05.

Docker's named-volume semantics are the crux: a named volume is populated from
the image's contents **only when the volume is first created and is empty**. On
every subsequent container recreation, an already-populated named volume is
mounted as-is — Docker does not reconcile it against, or re-seed it from, a newer
image. The result is that the July 5 "fresh pull" swapped the image but left the
five-week-old code in the volume running unchanged. This lines up exactly with
the `--version` self-report of ~149 commits behind and with the first missed
release dating to just after the volume's creation.

The consequence that makes this a deployment-integrity issue rather than a
cosmetic one: the container *looks* freshly deployed (recent image, recent
recreation timestamp) while running arbitrarily stale code, with nothing
surfacing the divergence.

---

## Remediation

Diagnosis and the correct update path were established; the live update is
scoped as a separate maintenance action.

1. Confirmed via container inspection that `/opt/hermes` is volume-backed and
   that the code in the volume predates the current image by five-plus weeks.
2. Confirmed the standard image-pull + redeploy flow does **not** advance the
   code, and must not be relied on for this container.
3. Identified the supported update mechanism: an in-container
   git-pull-and-reinstall (`hermes update`), which exposes a read-only
   `--check` dry-run and a `--backup` flag.
4. Confirmed that Hermes state (config, memories, sessions, cron, skills) lives
   on a separate host bind mount, independent of the code volume, and survives
   any of these update paths.

---

## Prevention

- **Documented the failure mode:** code served from a populated named volume is
  not advanced by an image pull; the update runbook for this container must use
  the in-container update path, not the GitOps image-tag flow used elsewhere.
- **Open follow-up (blocks live update):** two on-disk checkouts exist — the
  runtime path (no `.git`) and a second checkout with `.git` that is ahead of
  the running code. Resolve which one the in-container updater targets, via the
  read-only `--check` dry-run, before running a live update — otherwise an
  "update" could report success while still booting stale code.
- **Recommended monitoring:** a periodic running-version-vs-upstream comparison
  as an explicit signal, so version drift surfaces on its own rather than only
  during a manual audit.
- **Update procedure:** run the update with `--backup`, then verify scheduled
  jobs and all MCP server connections post-upgrade, since those subsystems
  changed most across the missed releases.

---

## Lessons Learned

- **"Freshly pulled image" does not mean "fresh code" when code lives in a
  populated named volume.** Docker seeds named volumes only on first creation,
  so a redeploy that is safe and correct for a stateless container can silently
  run arbitrarily old code for a volume-backed one. Deployment-integrity
  assumptions must be re-derived per container based on where its code actually
  lives, not assumed uniform across a stack.
- **Silent staleness is an observability gap, not just a maintenance lapse.**
  Nothing in the stack surfaced a six-week code divergence; it took a manual
  audit to find. A version-drift check turns an invisible latent condition into
  a monitorable signal.
- **Patch currency on an internet-facing agent is a security property.** An
  update path that silently no-ops means security fixes accumulate as missed
  work with no operator-visible indication — the absence of a signal is itself
  the risk.

---

*Environment: KONYKS-SERVER (Unraid) · Hermes-Agent · Docker named volumes · homelab-incident-reports*
