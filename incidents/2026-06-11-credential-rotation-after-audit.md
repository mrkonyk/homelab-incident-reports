# Full-Stack Credential Rotation Triggered by a Hardcoded-IP Outage

**Date:** 2026-06-11
**Severity:** P1 High
**Status:** Resolved
**Affected:** Nextcloud, MariaDB, Redis, Authelia, GitHub PAT, Hermes-Agent, Honcho (OpenRouter keys), CrowdSec
**Duration:** Roughly two days, beginning as a late-night outage and expanding into a full-stack audit

---

## Summary

A mobile client outage traced back to a hardcoded database host address exposed a deeper pattern: infrastructure that depends on a container's IP address staying fixed will eventually break, because nothing in a container orchestration platform guarantees that. The outage prompted a full health audit across the stack, which in turn surfaced several issues unrelated to the original failure — among them a security tool that had silently never been functional, and a reverse-proxy component that had been exposing an unintended network port for months without anyone noticing.

---

## Timeline

| Time | Event |
|---|---|
| Late evening | A mobile client begins failing logins with a malformed-response error |
| Shortly after | Root cause traced to a hardcoded database host address in application config |
| Same session | An unrelated credential rotation earlier that day had caused the database container to be recreated, and the container's address changed when it came back up — the hardcoded value no longer pointed anywhere valid |
| Following hours | Fix applied: switch the config to resolve the database host by container name instead of a fixed address |
| Same session, expanded scope | A full health pass across the entire container stack is run opportunistically, given the outage already had attention on it |
| Audit findings | Several unrelated issues surface: one background service had been silently crash-looping on a stale API key; a security/intrusion-detection tool turns out to have never been reachable due to a missing port mapping, meaning it had effectively never functioned since deployment; a separate auth service is found to have been binding to a random host port on every restart for several months, an unintended network exposure that had gone unnoticed; two datastore services are flagged for pending version upgrades |
| Remediation | A full credential rotation is performed across every affected service as a precaution, given the scope of what the audit uncovered |

---

## Root Cause

The proximate cause was a classic container-networking assumption failure: a piece of application configuration stored a database host as a fixed network address rather than resolving it dynamically by container name. Container platforms reassign addresses on recreation as a matter of course; the configuration had simply never been tested against that reality, because the database container hadn't been recreated since the address was first set. An unrelated credential rotation earlier in the day triggered exactly that recreation, and the stale address broke the dependency immediately.

The deeper, more consequential finding came from the audit the outage prompted, not from the outage itself: a security tool everyone believed was active had no actual network path to do its job, and had been in that state since it was first deployed. Its presence in the running-container list looked identical to it functioning correctly — nothing distinguished "running and ineffective" from "running and working" without an actual test.

---

## Remediation

- The database host configuration was switched from a hardcoded address to container-DNS resolution, eliminating the entire class of failure going forward.
- A full credential rotation was performed across the database, cache layer, the SSO service's session and signing secrets, the GitHub access token, and the API keys used by two AI-assistant components — treated as a precaution given how much the audit had already turned up that wasn't previously known.
- The security tool's missing port mapping was corrected, making it actually reachable for the first time since deployment.
- The auth service's unintended port exposure was fixed by binding it to an internal-only address rather than a host-exposed one.
- The two flagged datastore services were scheduled for their pending version upgrades.

---

## Prevention

- Configuration that depends on a specific container's network address is now treated as a known anti-pattern; container-name-based resolution is the standing default for any new service wiring.
- The fact that a security tool can sit silently non-functional while its container reports healthy is now an explicit audit item — "is it running" and "is it actually doing its job" are treated as two separate questions that both need a real answer, not just the first one.
- A public incident-log practice was established following this session specifically because of how much got found and fixed in one sitting — better to have a durable record than rely on memory for what changed and why.

---

## Lessons Learned

1. **A container's network address is not a stable identifier, and any configuration that treats it as one is a latent failure waiting for the container's next recreation.** This bug was dormant for as long as the database container happened not to be recreated — which made it look stable right up until an unrelated, completely reasonable maintenance action (a credential rotation) triggered the exact condition that broke it. The fix isn't "be more careful recreating containers" — it's that any config referencing another service by address rather than by name carries this risk permanently, regardless of how long it's gone unnoticed.

2. **A security tool's container running successfully says nothing about whether the tool is actually doing anything.** The intrusion-detection service had been deployed, was "up," and had been silently useless since day one because of a missing network path — and nothing about its outward state differed from a correctly working instance. The only way this got caught was an unrelated outage prompting a broader audit; without that audit, there's no natural point at which "it's running" would have been questioned.

3. **An outage is a legitimate trigger for a wider audit, not just a prompt to fix the one thing that broke.** The credential rotation, the dead security tool, and the months-long port exposure were all unrelated to the original failure — none of them would have been found by fixing only the reported symptom. Treating an incident as license to look broadly, rather than narrowly, is what actually found the higher-value issues in this session.

---

*Environment: KONYKS-SERVER (Unraid) · Nextcloud · MariaDB · Redis · Authelia · CrowdSec · Hermes-Agent · Honcho*
