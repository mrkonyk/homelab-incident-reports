# Hermes-Agent Telegram Interface Degraded With No Detection Surface

**Date:** 2026-07-23
**Severity:** P1 High
**Status:** Monitoring — cause unconfirmed, recovered without intervention
**Affected:** Hermes-Agent (Telegram platform adapter)
**Duration:** Unknown. Functional impact observed 2026-07-25, service confirmed working 2026-07-26

---

## Summary

Following the upgrade of Hermes-Agent from v0.14.0 to v0.19.0, the agent became unreachable over Telegram.
The container remained healthy by every available signal — clean exit code, zero restarts, stable uptime,
all five MCP servers up, LLM egress working — while its primary user-facing interface did not function. The
failure was discovered when a human noticed the agent had stopped replying. Differential testing ruled out
network egress, DNS, TLS/SNI filtering, the CA bundle, IPv6, HTTP proxying and HTTP/2, localising the fault
to the adapter's own connection-management code, where two timeout inversions were identified as the leading
candidate. The service recovered before any change was applied. Separately and more consequentially, triage
established that this system has **no reliable surface for the state it needed to report**: successful
connections are never logged, and the one status field that records them latches on first connect and never
clears. The outage duration is therefore unknown and unknowable from the available telemetry.

---

## Timeline

| Time (UTC) | Event |
|------|-------|
| 07-23 23:49 | Hermes-Agent container created at v0.19.0 |
| 07-23 → 07-25 | Telegram reconnect failures logged intermittently; container reports healthy throughout |
| 07-25 04:43 | honcho-api restarts; hermes-agent follows 41s later. Trigger never identified; host load elevated |
| 07-25 ~05:00 | Discovery — agent noticed to be unresponsive over Telegram |
| 07-25 05:00–06:00 | Triage: container state, network path, DNS, TLS, proxy, transport config |
| 07-25 06:00 | Fault localised to Hermes' own adapter stack; timeout inversions identified |
| 07-26 | Telegram confirmed working. No change had been applied |
| 07-26 | Telemetry audit finds no success-side logging and a latching status field |

---

## Root Cause

**Unconfirmed.** What is established, and what is not, is set out below.

### Failure signature

Exception census across the retained log window:

| Class | Count |
|---|---|
| `httpx.ConnectTimeout` | 24 |
| `httpx.ReadTimeout` | 14 |
| `httpx.ConnectError` | 8 |
| `httpx.ReadError` | 8 |

Plus 116 transport-level primary-connect failures logged with an **empty** exception message, 41 adapter
reconnect lines with empty tails, and 142 bare `Timed out` strings. Notably absent: `ssl.SSLError`,
`ssl.SSLCertVerificationError`, `ConnectionResetError`, `httpx.PoolTimeout`, `httpx.RemoteProtocolError` —
zero occurrences of any of them.

### What was ruled out

Each hypothesis was tested against the same path the application takes, not an adjacent one:

- **Container restart loop** — `RestartCount` 0, `ExitCode` 0, no healthcheck defined, stable uptime. No
  autoheal-class container on the host; no orchestrator action in the window. The apparent "restarting" in
  the logs was the adapter's own reconnect churn inside a stable container.
- **Network egress / DNS** — resolution succeeded via the Docker embedded resolver; `curl` to the Telegram
  API returned 302 in 0.39s from inside the container, and an unrelated HTTPS target returned 200 in 0.18s.
- **IPv6** — the container has no global IPv6 address and no default route, and the resolver returns AAAA
  first, which initially looked promising. Refuted: `/proc/net/tcp6` was empty (zero IPv6 sockets), an
  unroutable address fails in ~1ms rather than timing out, and a plain hostname-based request over the same
  system DNS succeeded.
- **TLS / SNI-based filtering** — refuted by two successful full TLS handshakes with `SNI =
  api.telegram.org` from inside the same container during the incident window.
- **HTTP proxy** — no proxy variables set; every established connection observed going direct to :443.
- **HTTP/2** — no `http2` kwarg anywhere; transports default to HTTP/1.1.
- **CA bundle** — both the venv `certifi` bundle and the stdlib bundle present and valid.
- **Python TLS stack** — minimal test in the same interpreter and venv, with no Hermes imports:
  `httpx.get()` against the API returned 302, and a raw `ssl`+`socket` handshake to the same address with
  the same SNI negotiated TLSv1.3 successfully.

TCP connections to the Telegram API were observed in `ESTABLISHED` state throughout. The environment could
reach Telegram; the application could not use it reliably.

### Leading candidate

Two timeout inversions exist in the adapter's connection stack:

1. The fallback transport attempts up to three sequential connects at 10s each (~30s worst case), and that
   entire chain is wrapped by a 30s outer bootstrap deadline. The outer deadline is therefore **less than or
   equal to** the worst-case inner chain — under any latency, the outer `wait_for` fires first, producing an
   empty-message `asyncio.TimeoutError`, abandoning a half-built initialisation and retrying.
2. `pool_timeout` (8s) is shorter than `connect_timeout` (10s) — currently masked by a large pool, but wrong
   independent of this incident.

This is consistent with the observed exception mix and with the empty-message failures. It is **not proven**.
The theory requires the connection legs to be slow, and every direct measurement showed them fast. It fits
spontaneous recovery better than a deterministic code regression does — a deterministic bug does not heal —
but "fits better" is not a demonstrated causal chain, and this report is filed accordingly.

---

## Remediation

None applied. The service recovered on its own.

Four hypotheses were tested and abandoned during triage. Three were refuted by evidence already collected
rather than by new tests:

- A v0.19 dashboard auth-gate change was suspected of driving healthcheck-triggered restarts — refuted on
  discovering no healthcheck exists at all.
- IPv6-first resolution with no clean fallback — refuted by the empty IPv6 socket table and by an
  unroutable address failing instantly rather than timing out.
- SNI-based filtering of the Telegram API — refuted by two prior successful TLS handshakes with that exact
  SNI from the same container.

The fourth was not a hypothesis about the system but about the evidence, and it invalidated a headline
finding — see below.

---

## The Measurement Error

Triage initially concluded that the adapter had **never** connected since the upgrade, based on grepping the
retained container log for success markers and finding zero across the full window. That conclusion was
false, and the method that produced it could not have produced any other answer.

**Hermes does not log successful connections.** The transition to `connected` is silent; only failures are
written. A search for success markers in this system returns zero unconditionally — for a healthy service and
a dead one alike. The zero was a property of the instrument, not of the system.

The state is instead exposed at an unauthenticated status endpoint on the dashboard port, which returns a
per-platform map of `state`, `error_code`, `error_message` and `updated_at`. That surface carries its own
defect: **the field latches.** It records the first successful connect and does not revert on subsequent
failures. During the audit it read `connected` with a timestamp frozen two seconds after container start,
while reconnect failures were being logged forty minutes later.

The consequences are direct:

- The outage duration in this report is unknown, and cannot be recovered from existing telemetry.
- A healthcheck built on that status endpoint would report healthy for the entire lifetime of a broken
  container — reproducing this incident's exact failure mode inside the control meant to detect it.

---

## Prevention

**Detection — revised approach.** The originally planned healthcheck against the status endpoint was
abandoned once the latching behaviour was found. Because this system logs failures loudly and successes not
at all, detection must be built on the failure side: an alert on Telegram reconnect-failure rate over a
rolling window, sourced from the existing log pipeline, with no container changes required. A process
liveness probe on the dashboard port is explicitly insufficient and would have reported healthy throughout.

**Open follow-ups:**

- Correct both timeout inversions: give the outer bootstrap deadline headroom over the worst-case fallback
  chain, and raise `pool_timeout` above `connect_timeout`.
- The 07-25 04:43 paired restart of honcho-api and hermes-agent remains unattributed. No orchestrator
  action, no autoheal-class container, no retained docker event.
- Container logs are per-container and do not survive recreation. The diagnostic record for this incident
  was captured before any redeploy; durable log shipping would remove that constraint.
- A redactor blind spot was found during the audit: the dashboard's HTML responses embed a session token in
  a shape the output redactor does not recognise. Pattern to be added; token treated as exposed.
- Confirm stability over a longer window before reclassifying this report as Resolved.

---

## Lessons Learned

**A search that can only return one answer is not evidence.** The claim that the service had never connected
came from grepping for success markers in a system that does not emit them. The result was structurally
predetermined, and it survived several rounds of otherwise careful reasoning because a zero looks like a
measurement. Every negative finding needs the same question asked of it: what would a positive have looked
like, and is this instrument capable of producing one? This is the third occurrence of the same class in one
month — a prior credential sweep enumerated by known values and was therefore blind to everything it did not
already know about, and a prior report closed as Resolved on claims never verified at the layer where they
could fail, costing eleven days of invisible breakage. In each case the tool answered the question it was
capable of answering rather than the question that was asked.

**Container health is not service health, and the gap here was total.** Zero restarts, clean exit code,
stable uptime, all MCP servers up, LLM egress working — every signal reported healthy while the agent's
primary interface did not work. Detection was a human noticing silence, which bounds the outage only by how
long it takes someone to expect a reply.

**A status field that latches is worse than no status field.** It answers "did this ever work" while
appearing to answer "does this work now", and a monitoring control built on it inherits the deception at
exactly the moment it is needed. The intended healthcheck for this incident would have been green for the
duration of the incident it was designed to catch. Where success is unobservable, monitor the failure side
instead — and treat any always-green signal as unverified until something has been seen to turn it red.

---

*Environment: KONYKS-SERVER (Unraid) · Hermes-Agent · Komodo GitOps · homelab-incident-reports*
