# Edge Auth Stack Remediation — Closing 17 Audit Findings and Three Independent Silent-Failure Discoveries

**Date:** 2026-06-26
**Severity:** P1 High
**Status:** Resolved
**Affected:** SWAG, Authelia, CrowdSec, Grafana, Vaultwarden, MariaDB, Nextcloud, Komodo-managed image versions across 8 stacks
**Duration:** Single extended session, continued from a prior critical-finding remediation

---

## Summary

Following resolution of a critical credential exposure (documented separately), this session worked through the remaining 17 findings from a proactive audit of the SWAG/Authelia/CrowdSec edge auth stack. Most findings were straightforward configuration fixes — externalizing inline secrets, pinning container image versions, correcting duplicate security headers, adding rate limiting. Three findings, however, surfaced something more significant on investigation: a documented "SSO is enabled" comment that didn't match actual behavior, and two independent cases of an intrusion-detection system silently failing to ingest logs from services it was supposedly monitoring — discovered only because each fix was verified against a real triggered event rather than a passing configuration check. All three were root-caused and closed with end-to-end verification.

---

## Root Cause (by category)

**Configuration drift between documentation and reality.** A reverse-proxy configuration comment asserted that single sign-on had replaced a service's native login. Investigation showed the reverse-proxy side of that integration had in fact been built correctly — the necessary identity headers were being forwarded on every request — but the receiving service had never been configured to trust or consume them. The comment described an intended future state, written at a point in time, and nothing had verified it described current reality since.

**Silent log-ingestion failure, case one (storage layer).** A monitored service's log file lived on a union/FUSE-backed storage path. The filesystem watch mechanism used by the log-shipping pipeline does not reliably fire change notifications across that storage layer. The pipeline reported the log file as present and being read, but zero lines were ever successfully parsed — and because the failure was silent (no error, just an empty result), it was indistinguishable from "nothing notable has happened yet" until a deliberately triggered event proved otherwise.

**Silent log-ingestion failure, case two (different mechanism, same symptom).** A second service exhibited an identical external symptom — zero parsed lines — investigated under the assumption it shared the first service's root cause. It did not. This service's logs were on the same problematic storage layer, but the immediate cause was that the small number of log lines present at investigation time were unrelated background noise, not authentication events; deeper inspection found the storage-layer issue was present but coincidental, and the actual fix required only addressing that single mechanism, not the additional log-fragmentation issue the first case had needed.

---

## Remediation

**Configuration externalization and hardening (straightforward findings):**
- Six secrets removed from a configuration file and externalized via file-reference environment variables.
- Eight container images pinned to their currently-running, verified-working digests rather than left on floating tags.
- A duplicated security header (set by both an upstream application and the reverse proxy) resolved by suppressing the upstream's version at the proxy layer — applied twice, for two different headers, after the first attempt's chosen mechanism turned out to have an undocumented execution-order interaction with the proxy's native header directive and had to be redone with a different, verified mechanism.
- Rate limiting added at the reverse-proxy layer specifically on authentication endpoints, scoped narrowly to avoid affecting unrelated traffic; verified by confirming a deliberate burst of requests was actually throttled with the expected status code.
- An extended session-persistence option enabled, with the explicit tradeoff (longer device-loss exposure window in exchange for fewer re-authentication prompts) made as a conscious choice rather than a default.

**SSO handoff fix:**
- The receiving service was configured to trust the identity header already being forwarded, with header-source restricted to the specific upstream proxy rather than any internal address. A direct administrative credential was retained and externalized as a recovery path independent of the SSO chain — verified by actually invoking that recovery path and confirming it worked, rather than trusting documentation describing how it should work.

**Log-ingestion fixes (both cases):**
- The affected services' logs were redirected to a storage path outside the problematic union/FUSE layer, with the corresponding log-shipping configuration updated to match.
- One case additionally required a log-format change at the source, since the original format fragmented multi-part log entries across the pipeline's processing stages in a way the receiving rule could never match regardless of file-path correctness.
- A new local parsing rule was added to correctly tag log lines from these directly-mounted file sources, since the existing parser chain's tagging stage only recognized syslog- and container-runtime-sourced lines by default — direct file sources were silently falling through every downstream filter with no error.
- Each fix was verified by generating a real failed-authentication attempt from a real external network address and confirming an actual alert was generated and an actual block applied — not by simulating the rule chain against a synthetic log line, which in one case had already reported success despite the live pipeline still being broken end-to-end.

---

## Prevention

- Documentation comments asserting a security control's status are no longer trusted at face value during audits; this session's experience is itself the argument for re-verifying documented state against actual behavior periodically, not just at original implementation time.
- The administrative recovery path for the newly-SSO-gated service is documented and was empirically confirmed working, not just asserted from training knowledge or vendor documentation (which was unreachable at the time and explicitly flagged as such rather than presented with false confidence).
- Open follow-up: the broader pattern underlying both log-ingestion failures — direct file-based log sources on union/FUSE storage being invisible to filesystem-watch mechanisms — is now a known class of failure. Any future service onboarded to the intrusion-detection pipeline via a direct file mount should be verified with a real triggered event before being considered "monitored," not just confirmed present in configuration.
- Open follow-up: at least one other secret-management gap (a service's admin credential, externalized in this session) joins a small accumulating list of services not yet onboarded to centralized secrets management, tracked as a consolidated future item rather than three separate ones.

---

## Lessons Learned

1. **A passing simulation of a detection rule is not evidence the live pipeline works.** Twice in this session, a tool's own dry-run/explain feature confirmed a parsing-and-alerting chain would fire correctly against a constructed test input — and in the live system, with real traffic, the pipeline was still completely non-functional, for a reason the simulation had no way to surface (a storage-layer notification failure upstream of where the simulation begins). The simulation tested whether the *rules* were correct; it could not test whether the *pipeline feeding the rules* was actually receiving anything. The only test that closes a detection-and-response finding is a real triggered event observed end to end, from real attempt to real alert to real automated response.

2. **Two failures with an identical external symptom do not imply an identical root cause — and assuming so costs a wasted fix.** Both log-ingestion failures presented the same way: a log file was being read, zero lines were ever successfully parsed, no error was logged anywhere. The natural assumption — apply the same two-part fix that resolved the first case to the second — would have been half right and half wasted effort. The second case shared one root cause with the first but not the other; diagnosing each independently, rather than pattern-matching from the first incident's resolution, was what kept the second fix minimal and correct rather than over-applied.

3. **A configuration comment describing intended behavior is a claim, not a fact, and ages the moment the system it describes changes underneath it.** The SSO-handoff comment was not malicious or careless — it accurately described what the author intended to build. It simply stopped being true at some point after being written, with nothing in the system designed to notice. The actionable takeaway isn't "write better comments" — it's that any control whose correctness depends on a comment being trusted rather than a behavior being tested should eventually get tested, on a cadence, rather than only at the moment it was first implemented.

---

*Environment: KONYKS-SERVER (Unraid) · SWAG · Authelia · CrowdSec · Grafana · Vaultwarden · MariaDB · Nextcloud · Komodo/Periphery GitOps · homelab-incident-reports*
