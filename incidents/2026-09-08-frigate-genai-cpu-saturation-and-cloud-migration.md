# NVR GenAI Descriptions Saturate a CPU-Only Host — Then a Cloud Migration Surfaces Four Silent-Failure Modes

**Date:** 2026-09-08
**Severity:** P1 High (availability impact: full host unresponsiveness ~25 min; no data loss)
**Status:** Resolved
**Affected:** NVR (Frigate 0.17.2), local inference server (Ollama 0.32.9), the Unraid host and all 46 containers on it
**Duration:** Outage ~04:51–05:29 EDT (~38 min degraded, ~25 min hard-unresponsive); migration work continued to ~07:25 EDT

---

## Summary

Enabling the NVR's GenAI object descriptions against a **local** vision model on a
CPU-only host drove the load average to **302** and available RAM to **1.3 GiB** on a
box with **no swap**. The host still answered ICMP and completed TCP handshakes, but
SSH could not finish an authentication handshake — sshd could not get scheduled.

Root cause was a latency/timeout mismatch that no model choice can fix: image encode
alone took **~42 s** on this hardware, and the NVR's inference client has a **120 s
timeout hardcoded with no configuration field exposing it**. Requests never completed,
timed out, were resubmitted, and newly tracked objects queued more — while the
inference runtime spawns a thread per core *per request*. The operator compounded it
by pulling and benchmarking a second model on the already-saturated box.

Recovery was clean: a graceful reboot completed despite the load, the array restarted
with **no parity check and zero sync errors**, and no data was lost.

Local inference was then ruled out permanently for this hardware class and the feature
was migrated to **cloud inference through the same local endpoint**, which leaves the
host idle. Getting there surfaced four independent silent-failure modes, documented
below — each of which produced a plausible-looking success state while being broken.

---

## Timeline

| Time (EDT) | Event |
|------|-------|
| 04:44 | GenAI enabled globally for the `person` object; NVR restarted. Config verified via its API |
| 04:47 | Single description manually triggered to test. Multimodal decode begins |
| 04:51 | First failure: image decode took 42.4 s; request cancelled at exactly **120 s**. Client logs "timed out" |
| ~04:55–05:10 | Second model pulled and benchmarked **on the same box** while the first workload was still saturating it |
| ~05:10 | Host stops completing SSH handshakes. ICMP and TCP connects still succeed |
| 05:12 | Access regained after a long-patience handshake: **load 302.64, 1.3 GiB available, 0 B swap** |
| 05:13 | Container stop/kill blocked by policy controls. Fell back to removing the trigger |
| 05:15 | NVR config restored from the pre-change backup; `genai` block absent, verified on disk |
| 05:21:59 | Graceful reboot issued. Load already falling (302 → 90) as services stopped |
| 05:25:41 | Host back. **Uptime 1 min, load 1.15** |
| 05:27 | Array `STARTED`, `mdResync=0`, `sbSyncErrs=0` — **no parity check, clean shutdown confirmed** |
| 05:29 | 46 containers up. NVR healthy, GenAI confirmed disabled **via its API**, not just the file |
| 06:41 | Migrated to a cloud vision model tag through the local endpoint. NVR restarted |
| 06:43 | End-to-end success on a real tracked object. Host load flat throughout |
| 07:09 | Prompt rewritten after a refusal-storage failure mode was found (see Finding 4) |

---

## Root Cause

**Operator sequencing, not bad luck.** The feature was enabled fleet-wide on a live
object type *before* anyone measured a single inference on the actual hardware. One
measurement — 42 s for one image — would have ended the plan immediately, because it
sits against a fixed 120 s ceiling with no headroom for a second image, a queue, or
the NVR's own detection workload.

Three mechanics turned a slow feature into a host outage:

1. **The timeout is not tunable.** The client is constructed with a default 120 s
   timeout and the config schema exposes no field for it, so it cannot be raised. A
   smaller model shortens the encode but does not change the ceiling, and the host
   is CPU-only.
2. **Failure amplifies load.** A timed-out request is resubmitted while newly tracked
   objects enqueue their own, and the inference runtime allocates threads per core
   per request. The run queue grows superlinearly.
3. **No swap.** Memory pressure on this host is a hard wall, not a slowdown.

The compounding error was investigating a saturated box *by adding work to it* —
pulling and benchmarking a second model mid-incident.

---

## Findings

### 1. The inference client's timeout is hardcoded with no config surface

The client is instantiated with a 120 s default and the GenAI config schema has no
timeout field, so nothing in user-facing configuration can raise it. **This is the
finding that makes local inference impossible on this class of hardware regardless of
model size** — it is a ceiling, not a tuning parameter.

### 2. The orchestrator's Environment panel feeds interpolation, not injection

A variable set in the stack's Environment panel is written to an env file used for
Compose **interpolation**. It does **not** reach the container unless `compose.yaml`
also references it under `environment:`. Without that reference the variable silently
resolves to unset — the panel shows a correct-looking value, the deploy succeeds, and
the container has nothing. Diagnosed only by checking the variable's presence inside
the running container.

Corollary: a **restart can never introduce a new environment variable**. A container's
environment is fixed at creation; only a recreate applies a change. Three restarts
were performed before this was understood.

### 3. Resource sync treats an absent TOML field as ambiguous, not as "reset"

Removing a configuration block from the declarative TOML produced **no pending change**
in the sync — the stored value simply persisted, and the sync reported success. An
absent field is not interpreted as "reset to default". Clearing the stored value
required editing it directly rather than relying on the sync to reconcile the deletion.

Separately, the sync's own repository checkout was **two commits stale** while the
per-stack checkouts were current, so a sync executed against config that no longer
matched the repository — and reported no changes, which looked like success.

**Relevant to any GitOps workflow using this orchestrator's resource syncs: state that
is not explicitly declared is invisible to the sync and cannot be reinstated from the
repo.**

### 4. The vision model stores safety refusals as if they were descriptions

The documented example prompt asks for a subject's "actions, behavior, and potential
intent". The cloud vision model's safety training **declines intent inference on images
of people** — and the NVR stores whatever text comes back. The result: a refusal
("I cannot fulfill this request… my safety guidelines prohibit…") was written into an
event description as though it were a real observation.

This is a **silent corruption** failure mode, not an error: no exception, no log
entry, no alert. It was also **intermittent** — two events succeeded and a third
refused under an identical prompt — so a single successful test proves nothing.

Resolved by rewriting the prompt to request only observable facts (actions, direction
of travel, carried objects, appearance, position in frame) and to explicitly forbid
speculation about thoughts or purpose. Verified across three distinct events: zero
refusals, including on the event that had previously refused. The rewritten output is
also materially better for a searchable log than speculation was.

### 5. Free-tier cloud usage is not observable programmatically

No rate-limit or quota headers are returned on responses, and no usage endpoint is
documented. **The quota wall will therefore hit silently** — descriptions will simply
stop being generated, with no error and nothing distinguishable from "no objects were
detected". The provider's web console is the only place consumption is visible.

---

## Resolution

1. **Contained** by restoring the NVR config from its pre-change backup and rebooting
   gracefully. Array came back clean: no parity check, zero sync errors, no data loss.
2. **Ruled out local inference permanently** on this hardware. Given Finding 1, this is
   a hardware/architecture conclusion, not a tuning problem.
3. **Migrated to cloud inference** using a cloud model tag served *through the same
   local endpoint*, so the NVR configuration change was one model string. The host
   performs no inference and stays idle.
4. **Corrected the authentication mechanism.** An API key had been provisioned on the
   assumption it authenticated the local server's cloud requests. It does not — that
   variable is for calling the provider's API directly as a client, and the local
   server never reads it. Cloud access is authenticated by a keypair registered to the
   account. Confirmed by the server's own startup log, which enumerates every
   environment variable it recognises; the API key was not among them.
5. **Reverted the dead configuration** once proven functionless, and verified cloud
   auth still worked afterward — which is what confirmed the keypair was the real
   mechanism all along.
6. **Rewrote the prompt** to eliminate the refusal failure mode (Finding 4).
7. **Treated the API key as exposed and revoked it.** It had been created without the
   masked/secret flag, making it readable in the console and API and liable to appear
   in deploy logs. Revoked at the provider; the local env file holding it was deleted.

---

## Prevention / Follow-up

- [ ] **Measure one inference on the target hardware before enabling any inference
      feature on a live object type.** This is the single control that would have
      prevented the outage. Ordering was the root mistake — the measurement was
      eventually taken, but only after the feature was already running fleet-wide.
- [ ] **Check whether a timeout is configurable before designing around it.** One
      log line enumerating recognised config variables would have invalidated both
      the local-inference plan *and* the API-key plan on day one. Two separate
      wrong turns in one session shared this root: assuming a knob exists.
- [ ] **Never add load to a box you are actively investigating.** Pulling and
      benchmarking a second model during saturation converted a slow feature into an
      unreachable host.
- [ ] **Treat plausible-looking output as a failure class.** Findings 2, 3, 4 and 5
      all produce a convincing success state while broken: an env panel showing a
      correct value that never reaches the container; a sync reporting no changes
      because deletion is invisible to it; a refusal stored as a description; a quota
      wall with no signal. **A failure that stores wrong-but-plausible output is worse
      than one that errors, because nothing alerts.** Verify via runtime state — the
      API, the container's actual environment — not the config file that was written.
- [ ] **Declare configuration state explicitly in the repo.** Undeclared state is
      invisible to a GitOps sync and cannot be reinstated from it.
- [ ] **Record model and dependency changes when they are made.** A model on this host
      was replaced weeks earlier with no written record anywhere; establishing when and
      why required filesystem timestamp forensics across manifests and cache files, and
      the "why" was never recovered. Cheap to write down at the time; unrecoverable
      later.
- [ ] **Cloud usage has no programmatic signal (open).** Consumption must be checked
      manually in the provider console until the feature's steady-state cost is known.
      No alerting exists and none was built this pass.
- [ ] **Refusal-rate spot check (open).** Three consecutive clean events is not proof
      given the intermittency in Finding 4. If refusals reappear, that is a model-fit
      problem rather than a wording problem.
