---
title: "External Heartbeat Monitoring: Dead-Man's-Switch Coverage for Three Single Points of Failure"
date: "2026-06-26"
severity: "P2 Medium"
status: "Complete"
affected:
  - "KONYKS-SERVER (heartbeat + Alertmanager path test)"
  - "Pi Zero 2 W (Pi-hole / Unbound / PiVPN)"
  - "HA Pi 4 (HAOS)"
duration: "Structural blind spot present since initial homelab deployment; closed this session"
summary: >
  All three primary monitoring systems (Beszel, Uptime Kuma, Alertmanager) run on the same hosts
  they monitor, meaning a total host failure or network death produces no alert — the alerting path
  shares fate with whatever it is monitoring. This is a class of failure that cannot be closed by
  adding more in-host monitoring sophistication. Four external dead-man's-switch checks were added
  via Healthchecks.io (a service with no dependency on the homelab network): a 5-minute server
  heartbeat on KONYKS-SERVER, a 5-minute heartbeat on the Pi Zero 2 W (single point of failure for
  DNS and remote access), a 5-minute HA automation-engine heartbeat on the HA Pi 4 (proving trigger
  execution rather than just OS liveness), and a weekly semi-manual Alertmanager path test that
  requires a human to confirm Telegram delivery before the check is marked green. All four confirmed
  live at session close. Doctrine documented in both SYSOPS.md copies (repo + Hermes, v0.2).
---

**Severity:** P2 Medium

## Timeline

| Time | Event |
| :--- | :--- |
| Session start | Gap identified: Beszel, Uptime Kuma, and Alertmanager all co-located with the hosts they monitor; Pi Zero 2 W had zero monitoring coverage |
| Early session | KONYKS-SERVER heartbeat implemented: User Scripts cron `*/5 * * * *`, curl to Healthchecks.io check `konyks-server-heartbeat`; grace window 10 min |
| | Pi Zero 2 W heartbeat implemented: plain crontab `*/5 * * * *`, same curl pattern, check `pihole-heartbeat`; grace window 10 min |
| | HA Pi 4 heartbeat implemented: HA-native `rest_command` + `time_pattern` automation (`/5`), check `ha-pi-heartbeat`; grace window 10 min |
| | Alertmanager path test implemented: User Scripts weekly cron `5 9 * * 1`, synthetic `SyntheticPathTest` alert via Alertmanager API, check `path-test` (confirm exact name on Healthchecks.io dashboard) |
| Session end | All four checks confirmed live: green "last ping" on Healthchecks.io for three 5-min heartbeats; path-test confirmed via manual Telegram delivery check |
| Session end | Doctrine added to repo-scoped SYSOPS.md and authoritative Hermes SYSOPS.md; both bumped to v0.2 |

## Root Cause

**Fate-sharing monitoring architecture.** All three monitoring systems deployed in this homelab run on the same hosts they monitor:

- **Beszel** and **Uptime Kuma** run on KONYKS-SERVER. A power loss, kernel panic, or network isolation on KONYKS-SERVER silences both with no outbound alert.
- **Alertmanager** runs on KONYKS-SERVER inside the observability stack. Same fate.
- **The Alertmanager → Telegram delivery path** runs through the HA Pi 4 (HA notification receivers). If the HA Pi 4 fails independently, notification delivery fails even if Prometheus and Alertmanager are still running on KONYKS-SERVER.
- **The Pi Zero 2 W** (Pi-hole, Unbound, PiVPN) had no monitoring of any kind — it is the single point of failure for all LAN DNS resolution and all remote access, and no check existed to detect its failure.

This gap is structurally distinct from a threshold-based omission (e.g., a missing disk-usage alert). Adding more in-host monitoring — more Prometheus rules, more Uptime Kuma checks, more Beszel panels — cannot close a fate-sharing gap, because the new monitoring would have the same architectural dependency on the host it is watching.

## Remediation

### KONYKS-SERVER heartbeat

Unraid User Script `konyks-server-heartbeat`, cron `*/5 * * * *`. Pings Healthchecks.io check `konyks-server-heartbeat`. Grace window: 10 minutes. Confirms the server is alive and the User Scripts executor is functional.

### Pi Zero 2 W heartbeat

Plain crontab entry on the Pi Zero 2 W, schedule `*/5 * * * *`. Pings Healthchecks.io check `pihole-heartbeat`. Grace window: 10 minutes. Covers the single point of failure for all LAN DNS (Pi-hole + Unbound) and remote access (PiVPN). First monitoring of any kind on this host.

### HA Pi 4 heartbeat

Home Assistant `rest_command` integration combined with a `time_pattern` automation (every 5 minutes). Pings Healthchecks.io check `ha-pi-heartbeat`. Grace window: 10 minutes.

This check is deliberately HA-native rather than a plain cron on the Pi 4 OS, because the objective is to verify that **the HA automation engine is executing triggers** — not merely that the Pi 4's Linux network stack is responsive. A plain OS-level ping would pass even if HA were stuck, restarting, or failing to process automations.

### Alertmanager path test

Unraid User Script `alertmanager-path-test`, cron `5 9 * * 1` (Monday 09:05). Posts a synthetic `SyntheticPathTest` alert (severity `informational`) directly to Alertmanager's API to exercise the full Prometheus → Alertmanager → receiver → Telegram delivery chain.

Healthchecks.io check: `path-test` (confirm exact name against dashboard). **This check is intentionally semi-manual.** The Healthchecks.io ping is only sent after a human confirms that the Telegram message actually arrived and was readable. This forces weekly end-to-end verification of the full human-perceived delivery chain. Automating the Healthchecks.io ping would reduce it to confirming that a curl command executed without error, which misses the failure mode this test exists to catch: a silently broken notification receiver that accepts the alert but never delivers to Telegram.

## Prevention

- Three 5-minute external heartbeats now provide absence-of-signal coverage for all three primary hosts. A dead host generates a Healthchecks.io alert via Telegram within the 10-minute grace window — independent of whether the host itself is still running.
- The Pi Zero 2 W is no longer dark. Its failure now produces a signal.
- The full Alertmanager → Telegram delivery path is verified weekly by a human, not just confirmed as "script ran."
- Operational doctrine documented in both the homelab-infra repo-scoped SYSOPS.md and the authoritative Hermes SYSOPS.md (v0.2 in both). The `path-test` semi-manual requirement is explicitly flagged in both with a "do not automate the Healthchecks ping" callout, so future sessions cannot silently remove the human-confirmation step in the name of automation.

### Remaining open

- Healthchecks.io is the sole notification channel for these checks. If Telegram itself is unreachable when a check fires, the dead-man's-switch alert is undeliverable. Acceptable at current homelab scale; email escalation would close this if needed.
- The weekly path-test relies on the human reviewer being present Monday mornings. No formal fallback if the check window is silently missed.

## Lessons Learned

1. **Fate-sharing is a class of failure, not a misconfiguration.** In-host monitoring is useful but has a hard architectural ceiling: it cannot alert when the host itself is gone, because it shares the host's fate. The only structural answer is external monitoring that runs entirely outside the monitored environment and treats silence as the alarm signal, not as the absence of a problem.

2. **"The script ran" and "the human received it" are different claims.** The Alertmanager path test was deliberately built to require the second, harder proof. Automating the Healthchecks.io ping would have been simpler but would have confirmed only that curl exited 0 — not that the notification was delivered and readable. The extra friction in the weekly check is load-bearing, not an oversight.

3. **Cross-cutting operational doctrine needs two landing points.** The heartbeat monitoring is relevant to both the GitOps-focused homelab-infra SYSOPS.md and the broader Hermes context file. Writing to only one creates a divergence that future sessions will silently act on as if the undocumented version is still authoritative. Both were updated in the same session as the implementation.
