# Pi-hole Beaconing Alert Storm — Alloy Usage Reporting to a Blocked Domain

**Date:** 2026-09-04 (onset visible) to 2026-09-06 (resolved)
**Severity:** P3 Low (availability impact: none; cost: ~3 days of noisy SOC alerts and a multi-hour misattribution hunt)
**Status:** Resolved
**Affected:** obs-alloy (grafana/alloy:v1.17.0), Pi-hole (192.168.4.250), Wazuh rule 100402
**Duration:** ~2 days of alerting; root-cause hunt ~4 hours on 2026-09-06

---

## Summary

Wazuh rule 100402 ("possible beaconing", level 10) fired every 5 minutes for three
days: Pi-hole blocked **exactly 50 queries** for `stats.grafana.org` per 5-minute
window, sourced from KONYKS-SERVER (192.168.4.70). The 50-count is what tripped the
rule's frequency threshold — and the round number was the first clue.

Root cause: **Grafana Alloy's built-in usage reporting**. Alloy posts usage stats to
`https://stats.grafana.org/alloy-usage-report` every 60 seconds. Pi-hole blocks the
domain, so every post failed, and Alloy's failure handler retries **5 times** with a
fresh DNS lookup before each retry. One report cycle = 1 attempt + 5 retries = 6
queries per minute ≈ 50 per 5-minute window. The "beacon" was a telemetry endpoint
failing into a retry loop.

---

## Timeline

| Time (UTC) | Event |
|------|-------|
| 2026-09-04 | Earliest stats.grafana.org blocks visible in Pi-hole logs (~7,200/day) |
| 2026-09-06 ~14:00 EDT | Investigation opens. Grafana initially suspected (domain shares its name) |
| 2026-09-06 18:30 | Grafana telemetry disabled (`GF_ANALYTICS_*` env vars), obs-grafana recreated — **alerts continue**; Grafana exonerated |
| 2026-09-06 ~18:31 | Loki config checked: `analytics: reporting_enabled: false` already set — exonerated |
| 2026-09-06 18:31 | `docker logs obs-alloy` shows live retry loop: `failed to report usage ... Post "https://stats.grafana.org/alloy-usage-report" (×5)` |
| 2026-09-06 18:29 | `--disable-reporting` flag deployed to obs-alloy, container recreated |
| 2026-09-06 18:29:22 | **Last steady-state stats.grafana.org query in Pi-hole log** |
| 2026-09-06 18:28:39 | Last Wazuh 100402 alert (pre-fix; queue drains) |
| 2026-09-06 ~18:48 | tcpdump (installed on Pi-hole) captures 7 min of port-53 traffic: zero stats.grafana.org packets — fix confirmed at packet level |
| 2026-09-06 ~21:30 | Fix committed (`68aa037`) and pushed; incident report written |

---

## Root Cause

Alloy's usage reporting is **on by default** and cannot be disabled via config file —
it is a runtime flag (`alloy run --disable-reporting`). The observability stack's
compose file ran Alloy with only `run /etc/alloy/config.alloy --storage.path=...`,
so reporting stayed active. Pi-hole's blocklist (correctly) classified
stats.grafana.org as telemetry, and the combination of a 60 s reporting interval,
blocked egress, and a 5× retry loop produced a self-sustaining query pattern that
matched Wazuh's beaconing signature.

### Why attribution was hard — the DNS-path gotcha

All host **and** container DNS on KONYKS-SERVER transits **tailscaled** (MagicDNS,
100.100.100.100), which forwards upstream to Pi-hole as the host IP 192.168.4.70.
Verified by control test: a single `dig @100.100.100.100 stats.grafana.org` produced
two new Pi-hole log lines instantly.

Consequence: **Pi-hole logs cannot attribute queries to containers** — every query
from the box appears as 192.168.4.70. Early in the hunt this made every host-level
exoneration test (stopping/reconfiguring one container) ambiguous, because the
query rate "didn't move" — until the broken measurement below was discovered.

### Measurement error that nearly misattributed the incident

An unquoted grep pattern (`grep query.A. stats.grafana.org`) silently treated the
domain as a **second file argument**, matching ALL DNS queries from 192.168.4.70
(~90/min) instead of only stats.grafana.org (~10/min). This inflated the apparent
query rate ~10×, made Grafana's daily-cycle reporting look plausible by comparison,
and made the working Alloy fix appear ineffective for ~45 minutes mid-session.
Quoted patterns (`grep "query.A. stats.grafana.org"`) reconciled every number with
Wazuh's count=50 threshold.

---

## Resolution

1. **Alloy** (the fix): added `--disable-reporting` to the compose `command`;
   recreated obs-alloy. Verified via container `.Args`, log silence, Pi-hole query
   stop, and tcpdump capture.
2. **Grafana** (hygiene): `GF_ANALYTICS_REPORTING_ENABLED=false` +
   `GF_ANALYTICS_CHECK_FOR_UPDATES=false` env vars; recreated earlier the same day.
3. **Loki**: no change — already disabled in config.yaml.
4. Committed to homelab-infra (`68aa037`, both the Komodo repos clone and the
   deployed stacks copy), pushed to origin/main.

### Telemetry toggle cheat-sheet (Grafana stack)

| Product | Toggle | Scope |
|---|---|---|
| Grafana | `GF_ANALYTICS_REPORTING_ENABLED=false`, `GF_ANALYTICS_CHECK_FOR_UPDATES=false` | env vars |
| Alloy | `--disable-reporting` | **runtime flag only** — not in config file |
| Loki | `analytics: reporting_enabled: false` | config.yaml |

---

## Prevention / Follow-up

- [ ] **Audit all default-on telemetry in the stack** — Alloy was found by accident
      during an alert hunt, not by design review. Loki/Grafana were already clean,
      but any future Grafana-stack addition (Tempo, Mimir, Pyroscope, Beyla) ships
      its own stats endpoint. Add a telemetry toggle check to stack-deployment review.
- [ ] **Investigation hygiene: quote grep patterns** — an unquoted multi-word
      pattern silently becomes a multi-file grep and over-matches. Cost ~45 min of
      wrong attribution. Checklist item: "reconcile raw counts against the alert's
      own count field before trusting rate math."
- [ ] **Wazuh-side silence for stats.grafana.org** (optional, not done): rule 100402
      still fires on any future telemetry domain Pi-hole blocks. Cheap insurance,
      moot while traffic is zero.
- [ ] **tcpdump permanently available on Pi-hole** (done) — the right vantage point
      for DNS attribution; Unraid itself cannot run tcpdump without an OpenSSL 1.1
      dependency fight not worth having.
- [ ] **Bind-mount checkout of homelab-infra cannot commit** (pre-existing, hit
      again): root-owned `.git/objects` blocks writes from the agent container;
      commits route through the Komodo repos clone via ssh. Sync of the fix to the
      bind-mount checkout still pending.
