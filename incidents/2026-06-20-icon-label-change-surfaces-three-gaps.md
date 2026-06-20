# Icon Label Change Surfaces Three Latent Infrastructure Gaps

Date: 2026-06-19 to 2026-06-20
Severity: P1 High
Status: Resolved
Affected: Komodo (Core, Periphery, Mongo), SWAG, Authelia, all SWAG-proxied services
Duration: ~14 hours across two sessions (intermittent, not continuous outage)

---

## Summary

A purely cosmetic change — adding Docker `net.unraid.docker.icon` labels to 11
Komodo-managed observability/IaC containers — forced those containers to recreate.
The recreation did not introduce new bugs; instead, it tripped three separate,
pre-existing latent issues that had been sitting undetected in the infrastructure:
a non-persistent secrets bootstrap path, a reverse proxy missing a network
attachment to an entire application tier, and a reverse proxy auth rule that had
never correctly exempted the login endpoint itself from forward-auth. Each was
diagnosed and resolved independently; all three are now fixed and the underlying
fixes are verified to persist across restarts.

---

## Timeline

| Time | Event |
|------|-------|
| Session 1, ~T+0 | Icon labels added to 11 Komodo-managed compose services, pushed, redeploy triggered |
| Session 1, ~T+5min | Container recreation crashes the Komodo database container — missing persistent `.env` |
| Session 1, ~T+30min | Database credentials manually recovered; new `.env` written to a persistent path |
| Session 1, ~T+45min | Web login portal returns 403 — diagnosed as stale upstream proxy config, resolved |
| Session 1, ~T+1hr | Login form fails to render fully; root cause not yet found, session ends |
| Session 2, +several hours | Login portal returns 502 Bad Gateway — new, distinct issue |
| Session 2, +10min | Root cause: reverse proxy has no network path to an entire application tier |
| Session 2, +20min | Fix applied live; persistence gap identified and closed via existing-but-unscheduled automation |
| Session 2, +40min | Login form renders, but authentication fails — traced to a CORS/manifest asset issue |
| Session 2, +1hr | Manifest CORS issue fixed; authentication still inconsistent (401, then 302) |
| Session 2, +1.5hr | Final root cause found: login endpoint itself was never exempted from forward-auth |
| Session 2, +1.5hr | Fix applied, login succeeds, session closed |

---

## Root Cause

Three independent, unrelated root causes, each pre-existing and each only surfaced
because the icon-label change forced container recreation and a fresh proxy/auth
flow to be exercised end-to-end for the first time in a while:

**1. Non-persistent secrets bootstrap.** The IaC database's initial `.env` file
had been created under a temporary path during original setup rather than the
project's persistent appdata directory. It worked fine as long as the container
never needed to be recreated — the moment it did, the credentials it depended on
were gone, because the temp path had since been cleaned by routine system
maintenance.

**2. Reverse proxy missing a network attachment.** Following an earlier network
segmentation migration that moved application containers onto a dedicated tier
network, the reverse proxy container was never actually attached to that tier
network — only to the proxy-facing tier. This had been silently true for every
application behind the proxy (not just the one being worked on) since the
migration. It went unnoticed because long-lived authenticated sessions in
browsers don't necessarily re-resolve DNS, masking the gap until something forced
a clean request — in this case, a cleared session forcing a fresh login attempt.

**3. Auth endpoint not exempted from forward-auth.** The reverse proxy's
forward-auth integration wrapped every request path under a single broad rule,
with narrow exemptions already in place for a couple of specific paths (a
webhook receiver, and — added during this same incident — a static asset).
The application's own login API endpoint was never added to that exemption list.
Since the login endpoint is, by definition, called before any valid session
exists, routing it through forward-auth produced inconsistent results depending
on session/cookie state — sometimes a clean rejection, sometimes a redirect that
the frontend couldn't parse as a valid API response. This made the symptom look
like a credentials problem for an extended period, when the actual password was
correct the entire time.

---

## Remediation

**Issue 1 (secrets persistence):**
- Stopped the crash-restart loop on the database container
- Brought up a temporary unauthenticated instance to reset the application's
  database credential
- Wrote a new `.env` to the project's actual persistent appdata path
- Restarted the stack against the corrected, persistent secrets file

**Issue 2 (missing network attachment):**
- Confirmed via direct request testing (not assumption) that the reverse proxy
  could not resolve DNS for application containers on the missing tier — and
  confirmed this was true for multiple unrelated services, not just the one
  being debugged, before treating it as the actual root cause
- Applied a live `docker network connect` to restore connectivity immediately,
  with no service restart required
- Discovered a host automation script already existed for exactly this class of
  problem (used elsewhere on the same host for other secondary network
  attachments) but had never been scheduled to run
- Scheduled it to run automatically at every array startup, closing the
  persistence gap with no container or template modification needed

**Issue 3 (auth endpoint not exempted):**
- Diagnosed via direct comparison of browser network requests, application
  logs, and proxy logs together — confirmed the literal forward-auth rejection
  reason rather than assuming it was a credentials issue
- Added an exact-path exemption for the login endpoint, mirroring the existing
  exemption pattern already used for the webhook and static-asset paths in the
  same config file
- Reloaded the proxy configuration (no service restart required) and confirmed
  successful authentication end to end

---

## Prevention

- The persistent-secrets fix directly closes the gap that caused issue 1 —
  the credential now lives at the correct, durable path
- The network-reattach automation is now scheduled at array startup, closing
  the most common recurrence path for issue 2 (full host/array restart);
  a residual gap remains for a mid-session container recreate triggering the
  same issue without a manual re-run of that script — noted as an open item
  below, not yet automated
- The auth-exemption fix for issue 3 follows the same precedent now established
  twice in the same config file, making future exemptions for new internal API
  endpoints a known, low-risk pattern rather than a one-off
- **Open follow-up:** audit all containers' network attachments against the
  documented tier map, since issue 2 was discovered to potentially affect more
  than just the one service being worked on at the time. Not yet completed —
  scoped as a separate future session to avoid expanding blast radius during
  active remediation.
- **Open follow-up:** the network-reattach automation does not yet trigger on
  a mid-session container recreate (image update, manual edit, etc.) — only on
  full array startup. A manual re-run is currently required after any such
  recreate. Worth automating as a post-recreate hook in the future.

---

## Lessons Learned

1. **A cosmetic, "safe" change is only as safe as everything underneath it.**
   Adding a display label seems like the lowest-risk possible infrastructure
   change, but any change that forces a container recreation re-exercises every
   assumption that container's startup depends on — including ones that have
   silently held for a long time. The actual risk of a change isn't its stated
   purpose; it's the full state transition it triggers.

2. **Symptoms that look identical can have entirely different causes.** A blank
   or failing login screen showed up across this incident for at least four
   distinct underlying reasons (a 403, a 502, a CORS failure, and an
   inconsistent auth-endpoint redirect) — none of which were actually about the
   credentials, even though "wrong password" was the most intuitive read each
   time. Treating each new symptom as requiring fresh evidence, rather than
   reusing the previous explanation, was what eventually separated these out
   correctly.

3. **Cosmetic-looking log noise is sometimes the root cause wearing a disguise.**
   An auth-header parsing error had been appearing in logs from early in this
   incident and was twice dismissed as harmless noise before it turned out to
   be the literal mechanism behind the final root cause. Recurring log lines
   deserve a second look once a "fully resolved" state still isn't reached —
   persistence of an error is itself a signal.

---

*Environment: KONYKS-SERVER (Unraid) · Komodo · SWAG · Authelia · homelab-incident-reports*
