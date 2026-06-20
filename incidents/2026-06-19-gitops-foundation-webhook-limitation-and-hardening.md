# GitOps Foundation: Webhook Limitation, Silent Polling Failure, and Observability Stack Hardening

Date: 2026-06-19
Severity: P2 Medium
Status: Resolved
Affected: Komodo Core/Periphery, homelab-infra reconciliation loop, observability stack (Prometheus, cAdvisor, Uptime Kuma scrape target)
Duration: Single session

---

## Summary

Standing up Komodo as the GitOps deployment engine for the homelab surfaced three separate issues, each masked behind a more obvious one. The intended webhook-triggered reconciliation loop turned out to be unreachable on the installed Komodo version, which led to switching to scheduled polling instead — but the polling mechanism was also silently broken, failing on every cycle due to a missing git credential in the Core container. Separately, the new observability stack shipped with a cAdvisor container that would not survive a host reboot, and a Prometheus scrape target that authenticated incorrectly against Uptime Kuma's metrics endpoint. All three were diagnosed with evidence (not assumed) and resolved in the same session.

---

## Timeline

| Time | Event |
|------|-------|
| T+0 | Komodo Core and Periphery deployed; Periphery registers successfully against Core |
| T+0 | Webhook reconciliation configured per plan; first webhook delivery returns HTTP 405 |
| T+0 | poll_for_updates configured as the interim fallback |
| T+0 | Observability stack deployed; cAdvisor and Prometheus come up healthy |
| Later | Investigation requested into whether the 405 was a real limitation or a misconfiguration |
| Later | OPTIONS probe against /webhook/* and /listener/* shows zero registered route handlers in this Komodo version — confirmed as a genuine platform limitation, not a config error |
| Later | While confirming polling was functioning as the fallback, discovered the Stack resource had been failing to sync on every 5-minute cycle |
| Later | Root cause traced to a missing .netrc in the Core container, preventing the private repo clone |
| Later | GitHub Git Provider Account created in Komodo and linked to the Stack/Repo resources; polling cycle begins succeeding |
| Later | mount --make-rshared / added to /boot/config/go; confirmed active without requiring a reboot |
| Later | Uptime Kuma target found returning 401 in Prometheus; root cause traced to an unauthenticated scrape against an authenticated metrics endpoint |
| Later | Scoped API key created in Uptime Kuma; Prometheus scrape config updated to basic auth via a gitignored, mode-600 secrets file |
| Later | All 11 Prometheus targets confirmed UP; all four original verify gates green |

---

## Root Cause

1. Webhook routes do not exist in Komodo v1.16.12. The initial assumption was that the webhook listener was misconfigured — wrong base URL, wrong path. An OPTIONS probe against the webhook endpoints showed no registered POST handler at all; the only thing responding was the SPA's catch-all GET route, which returns 405 for any other method. KOMODO_WEBHOOK_BASE_URL only controls what URL the UI displays to the user for configuring a git provider's webhook — it has no effect on whether the listener itself exists. This is a version limitation, not a configuration mistake.

2. The Core container had no git credentials. With webhooks ruled out, poll_for_updates + auto_update was configured as the reconciliation mechanism. This looked correct in the Komodo UI (the Stack resource existed, polling was enabled), but every poll cycle was silently failing to clone the private homelab-infra repository, because the Core container had no .netrc or equivalent git authentication configured. The Stack sat in a non-running state with missing_files populated, but nothing surfaced this as an active failure — it just looked like "waiting for the next poll."
3. cAdvisor's bind mounts require shared mount propagation that Unraid doesn't set by default. Without mount --make-rshared / in the Unraid boot script, cAdvisor's host filesystem mounts (/rootfs, /sys, /var/lib/docker) don't propagate correctly after a reboot, and the container fails to start cleanly on next boot.

4. Prometheus was scraping Uptime Kuma's metrics endpoint without authentication. Uptime Kuma's /metrics endpoint was enabled and required an API key; the scrape config had no basic_auth block, so every scrape returned 401 and Prometheus marked the target DOWN. No alerting rule existed on Prometheus target health itself, so this had no visible symptom beyond a red row in the targets list.

---

## Remediation

Webhook → polling, with the polling path actually fixed:
- Confirmed via OPTIONS probe that no webhook route exists in this Komodo version; documented rather than continuing to chase a fix that doesn't exist server-side.
- Created a GitHub Git Provider Account in Komodo (persisted in its MongoDB datastore) and linked it to both the Stack and Repo resources, giving Core the credentials it needed to clone on each poll.
- Confirmed the Stack now reports "state":"running" with missing_files: [], and that the 5-minute poll picks up new commits correctly.

cAdvisor persistence:
Added to /boot/config/go:
# cAdvisor requires shared mount propagation on the bind mounts (/rootfs, /sys,
# /var/lib/docker) to survive a reboot — without this, the container fails to
# start cleanly after Unraid boots.
mount --make-rshared /

Confirmed active post-change via the mount propagation flag, without requiring an actual reboot to verify.

Prometheus / Uptime Kuma auth:
- Generated a scoped API key in Uptime Kuma (prometheus-metrics, ID 2).
- Stored the token at stacks/observability/secrets/uptime_kuma_metrics_token, permissions 600, owned by the Prometheus container's non-root uid, and covered by the existing secrets/ entry in .gitignore.
- Updated prometheus.yml to use basic_auth with password_file pointing at the mounted secret, replacing the unauthenticated scrape.
- Mounted the secret read-only into the Prometheus container via compose.yaml.
- Verified all 11 Prometheus targets report UP with no scrape errors after reload.

---

## Prevention

- Documented the webhook limitation directly in the project brief and repo, rather than leaving a future reader to rediscover via another failed webhook attempt — including the OPTIONS-probe evidence, so the conclusion is reproducible, not asserted.
- Polling is now monitored implicitly through Stack state rather than assumed to be working — the missing-credential failure mode (silently stuck, no error surfaced) is a known gap worth a follow-up: an alert on Stack sync staleness would have caught this faster than a manual check.
- Secrets handling pattern established for future scrape targets: gitignored secrets directory, mode-600 files, non-root container ownership, mounted read-only. Any future authenticated scrape target should follow this same pattern rather than reinventing it.
- /boot/config/go now documents the cAdvisor mount-propagation requirement inline, so it survives the next person (including future-self) auditing that script without context on why the line is there.
- Open follow-up: the Alertmanager → Home Assistant route uses a hardcoded LAN IP for the Pi running Home Assistant, because the internal domain doesn't resolve from the Docker bridge network. This is documented inline in alertmanager.yml, but it's a known fragility pattern (see prior incident on hardcoded bridge IPs) worth revisiting if/when DNS resolution from Docker networks is addressed.

---

## Lessons Learned
1. A fallback mechanism needs the same verification rigor as the primary one. Switching from webhooks to polling was the right call once the webhook limitation was confirmed — but polling was accepted as "the working mechanism" before anyone checked that it was actually working. The Stack looked configured, not broken; only a credentials check revealed it had been failing every cycle. Anytime a fallback replaces a primary mechanism, it earns the same scrutiny the primary one would have gotten, not less, precisely because a clean-looking config is no guarantee of a working one.

2. Cosmetic-looking failures are still failures until proven otherwise. The Uptime Kuma target showing DOWN was initially framed as cosmetic because no alert fired on it — but the absence of an alert says more about a gap in the alerting rules than about the severity of the underlying problem. A red target with no corresponding alert is a blind spot, not a non-issue; closing the auth gap rather than tolerating the red status keeps the observability layer trustworthy as a source of ground truth.

3. Platform limitations should be confirmed with evidence, not assumed from a status code. A single 405 response could plausibly mean a wrong path, a wrong method, or a missing route entirely — three very different fixes. The OPTIONS probe that revealed zero registered handlers turned a guess into a documented fact, which is the difference between "we couldn't get webhooks working" and "this version of Komodo does not implement webhook routes," only one of which is useful to the next person who hits the same wall.

---

Environment: KONYKS-SERVER (Unraid) · Komodo Core/Periphery · Prometheus / Loki / Grafana Alloy / Alertmanager · Uptime Kuma · homelab-infra (private) · homelab-incident-reports