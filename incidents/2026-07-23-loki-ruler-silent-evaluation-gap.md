# Log-Based Alerting Silently Stops Evaluating Under Array I/O Load

**Date:** 2026-07-23
**Severity:** P2 Medium
**Status:** Open — cause confirmed, hardening identified but not yet applied
**Affected:** obs-loki (ruler), all Loki-backed alert rules
**Duration:** Intermittent across the retained window (2026-07-13 → 2026-07-26); 7 discrete evaluation failures

---

## Summary

The Loki ruler — the evaluation engine behind every log-based alert rule on this host — stops evaluating
during periods of sustained array I/O load. Under that load the single-binary Loki instance fails its own
internal ingester health check, the hash ring transiently empties, and any rule evaluating at that moment
errors out with `too many unhealthy instances in the ring`. A failed evaluation produces no alert and no
operator-visible signal; the rule simply does not run for that cycle. Seven such failures occurred in the
retained window, all against the same pre-existing rule. The condition is self-recovering and leaves the
ruler reporting healthy afterwards, which is why it had gone unnoticed. It was found only because a newly
deployed rule set prompted a deliberate check of whether the evaluation substrate was actually reliable —
not because anything alerted.

---

## Timeline

| Date | Event |
|------|-------|
| 07-14 | First observed evaluation failure (isolated) |
| 07-22 | One evaluation failure, coinciding with a short load excursion |
| 07-23 | Five evaluation failures across roughly one hour of sustained load |
| 07-26 | New alert rules deployed; substrate reliability questioned as part of validation |
| 07-26 | Ruler logs reviewed; failures correlated against retained load metrics; cause confirmed |

---

## Root Cause

Loki runs here as a single binary — distributor, ingester, querier, ruler and compactor in one process,
with the ingester registered in the ring against its own loopback gRPC endpoint. That self-registration is
the fragile part: under heavy I/O wait the process cannot answer its own health check within the deadline,
so it evicts itself from the ring.

Observed sequence in the ruler logs:

```
removing ingester failing healthcheck  reason="rpc error: code = DeadlineExceeded"
auto-forgetting instance from the ring because it is unhealthy for a long time
error getting addresses from ring  err="at least 1 healthy replica required, could only find 0"
rule evaluation failed  err="too many unhealthy instances in the ring"
```

The load source is a parity check across the array. The bottleneck is I/O wait rather than CPU contention —
`node_load1` reached 23 during the 07-22 excursion and 30 → 86 → 37 → 27 across the 07-23 window on an
eight-thread host, while Loki itself sat at roughly 146 MiB and 1% CPU throughout.

Resource exhaustion was ruled out directly:

| Check | Result |
|---|---|
| Memory / CPU limits | None set |
| Actual usage | ~146 MiB, ~1% CPU |
| OOM killed | No |
| Restart count | 0 |

Loki was not starved of memory or throttled by a limit. It lost the scheduling race, and the ring health
check has no tolerance for that.

Ingestion is affected by the same condition: `POST /loki/api/v1/push` returned 500 with
`at least 1 live replicas required` during the load windows, meaning log lines can be rejected at the
moment the ring is empty. Historical per-day push-failure counts could not be reconstructed (see
Observability Gaps below). Since the most recent restart, push success has been 100%.

### Why it stayed invisible

Three properties combine:

1. **A failed evaluation is not an alert.** The rule logs an error and moves on. There is no
   "evaluation failed" notification path.
2. **Ruler health reflects only the most recent evaluation.** Querying the ruler API after recovery shows
   `health: ok` and an empty `lastError` — a rule can report healthy having missed evaluations earlier.
3. **The condition self-recovers.** By the time anyone looks, the ring is intact and nothing is wrong.

---

## Observability Gaps Found During Investigation

The component that all log-based alerting depends on is the component with the least telemetry:

- **Loki is not scraped by Prometheus.** There is no scrape job for it, so no retained series for rule
  evaluation failures, push status codes, or ring health. Every figure in this report came from parsing
  container logs.
- **The Loki proxy ships no access logs**, so push outcomes cannot be reconstructed from that side either.
- **No alert exists on alerting-layer failure.** Nothing watches whether the thing that watches is working.

---

## Remediation

None applied. The condition is self-recovering and no fix was made during investigation.

---

## Prevention

**Identified, not yet applied:**

- **Widen short rule lookback windows.** A rule with a 5-minute lookback and no `for:` duration loses an
  event outright if evaluation stalls longer than its window — by the time evaluation resumes, the event
  has aged out of the query range. Widening the window to comfortably exceed a plausible stall costs a
  later and longer-lived alert and removes the hole. Rules with longer windows and a `for:` duration are
  substantially less exposed.
- **Scrape Loki with Prometheus.** Rule evaluation failures and push status are exported as metrics; none
  of them are currently collected. This would also make an alert on alerting-layer failure possible.
- **Do not add resource limits to Loki.** Usage is negligible and the failure is scheduling-related.
  Constraining it would make the condition more likely, not less.
- **Longer term**, moving the array off its current single-bridge storage path would reduce the I/O wait
  that triggers this. That work is already tracked separately for reliability reasons; this is an
  additional argument for it.

---

## Lessons Learned

**The detection layer has its own availability, and here it was the only layer not measured.** Every
service on this host is watched by something. The thing doing the watching is not scraped, exports no
retained metrics, and has no alert on its own failure. That inversion is easy to arrive at honestly —
monitoring gets built to watch workloads, and then quietly becomes a workload nobody watches — but it means
the failure mode with the widest blast radius is the one with the least evidence available after the fact.
Reconstructing seven evaluation failures required parsing container logs, and the ingestion-loss question
could not be answered historically at all.

**Correlated failure defeats independent-looking controls.** The alerting degradation is not random: it
happens under sustained array I/O load, which is also when storage-related faults are most likely to
surface. A control that is reliable except during the exact conditions it exists to catch is weaker than
its uptime suggests. Worth asking of any monitoring component: what makes it fail, and does that overlap
with what makes the monitored thing fail?

**A control reporting healthy is not evidence it has been running.** The ruler shows `health: ok` after
recovery with no indication that earlier cycles errored. Anything that reports only its most recent state
will describe a gap as normal operation once the gap closes. Verifying a control means checking its
history, not its current status — and where no history is retained, that verification is not available at
all, which is itself the finding.

---

*Environment: KONYKS-SERVER (Unraid) · obs-loki · Grafana Alloy · Alertmanager · homelab-incident-reports*
