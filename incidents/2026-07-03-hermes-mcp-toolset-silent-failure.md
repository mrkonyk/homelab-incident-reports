# Silent MCP Toolset Misconfiguration Caused Weeks of Fabricated Monitoring Reports

**Date:** 2026-07-03
**Severity:** P1 High
**Status:** Resolved
**Affected:** Hermes-Agent (Morning Briefing cron, Sunday Maintenance cron), MCP toolset resolution, Home Assistant MCP connectivity
**Duration:** Undetected for the full lifetime of both affected jobs — Morning Briefing's Unraid check never once succeeded since job creation; Sunday Maintenance's last confirmed real check was 12+ days before discovery

---

## Summary

A routine request to consolidate two overlapping morning cron jobs led to the discovery that two independent Hermes-Agent cron jobs — Morning Briefing and Sunday Maintenance — had been silently unable to reach any MCP tool (Unraid, Home Assistant, Nextcloud) for their entire operational history. Rather than surfacing an error, the affected jobs either produced honest "unavailable" fallback text or, in the majority of runs, fabricated confident, specific "all clean" health reports with zero real data behind them. Root cause was a cron config using `"mcp"` as a toolset name — a value that was never valid in this Hermes version, since MCP servers register individually as `mcp-<server>` aliases rather than a single combined `mcp` alias. The invalid toolset name failed silently because cron jobs run under `quiet_mode`, which discards the internal `Unknown toolset` warning that would otherwise have caught this immediately. A secondary, compounding issue — a stale auth token and incorrect transport protocol for the Home Assistant MCP connection — was found and fixed during the same investigation. Both jobs are now fixed, verified against real returned data, and resumed.

---

## Timeline

| Time | Event |
|------|-------|
| Day 1, ~08:00 | Routine request to consolidate two overlapping morning crons into one 9AM job |
| Day 1 | Consolidation completed; PAT-expiry threshold logic added to merged job |
| Day 2, 09:01 | Merged job delivers "⚠️ Unraid health check unavailable this run" — first anomaly noticed |
| Day 2 | Investigation begins; confirmed the model was attempting MCP tool calls as invalid browser-console JavaScript rather than real top-level tool calls |
| Day 2 | Prompt corrected to use proper tool invocation; live-fire still rejected with "Tool does not exist" — ruled out prompt wording as the cause |
| Day 2 | Two full restarts (gateway, then container) performed; failure reproduced identically both times — ruled out stale binding |
| Day 2 | Toolset-combination hypothesis tested (trimming to `["mcp"]` alone); revealed a second, independent job (Sunday Maintenance) actively fabricating a fully detailed "all healthy" report live, with zero real tool calls |
| Day 2 | Both affected jobs paused pending root cause |
| Day 2 | Root cause traced directly in Hermes source: `"mcp"` was never a registered toolset name; correct aliases are per-server (`unraid`, `home_assistant`, `nextcloud`, etc.) |
| Day 2 | Toolset names corrected for both jobs; Home Assistant 404 diagnosed as stale token + wrong transport protocol, both fixed |
| Day 2 | Both jobs verified with real returned data across all required MCP servers, resumed, and reconfirmed with a final live-fire each |

---

## Root Cause

Both jobs had `"mcp"` listed in `enabled_toolsets`. This string was never a valid toolset name in this Hermes version. Tracing through Hermes' installed source:

- `toolsets.py`'s `validate_toolset()` accepts names only from a static `TOOLSETS` dict, registered plugin toolsets, or dynamically registered MCP server aliases.
- When an MCP server connects, Hermes registers it as an individual alias matching its server key (e.g. `unraid`, `home_assistant`, `nextcloud`) — never a single combined `mcp` alias covering all servers.
- `model_tools.py`'s toolset resolution silently drops any unrecognized name: it prints `⚠️ Unknown toolset: {name}` and contributes zero tools for that entry — but cron jobs execute with `quiet_mode=True`, which discards that print before it reaches any log or stored transcript.

The result: an invalid config value failed with no observable signal anywhere — not in delivered reports, not in logs, not in stored sessions. The two jobs behaved differently on the same underlying failure depending on what *else* was in their toolset list: Morning Briefing (which also had `browser` and `session_search`) sometimes attempted a guessed MCP tool name and received a genuine "tool does not exist" rejection; Sunday Maintenance (which had only `mcp`) had nothing to reach for and skipped straight to confidently fabricating a full report from training-data plausibility rather than real data — in direct violation of its own prompt's explicit instruction to report data as unverifiable when tools return nothing.

A secondary, unrelated issue was found on the Home Assistant MCP connection specifically: the stored auth token had gone stale after a prior token rotation, and the configured transport (`sse`) was incorrect for a server that had since moved to the modern Streamable HTTP transport — producing a persistent `405 Method Not Allowed` independent of the token issue.

---

## Remediation

1. Corrected `enabled_toolsets` for both jobs to use real per-server aliases instead of the invalid `"mcp"` value:
   - Morning Briefing: `["browser", "session_search", "unraid"]`
   - Sunday Maintenance: `["home_assistant", "nextcloud", "unraid"]` (determined by re-reading the job's own prompt for which servers it actually references)
2. Confirmed the real tool-invocation naming convention directly from source (`mcp_<server>_<tool>`, underscore-delimited) rather than assuming — an earlier attempted fix using a colon-delimited guess had been wrong.
3. Fixed the Home Assistant MCP connection: updated to the current auth token and removed the incorrect `sse` transport declaration.
4. Verified all fixes required a full container restart to take effect — config edits are not picked up by a live-running gateway process.
5. Verified the Home Assistant fix in isolation using a disposable, local-delivery-only scratch cron job before touching either production job, to avoid further unnecessary live Telegram deliveries.
6. Both production jobs paused during the investigation to stop further fabricated reports from being delivered; resumed only after each was confirmed, via live-fire, to return genuine data from every MCP server it depends on.
7. Along the way, caught and fixed a second bug introduced during remediation: an early version of the PAT-expiry check (a separate, related change made just before this investigation) produced completely empty output on the common "all fine" case, causing a full silent blackout of the entire delivered report rather than just omitting the PAT section. Fixed by having the check always emit an explicit status line and filtering delivery on a specific marker rather than output presence.

---

## Prevention

- Documented the correct per-server toolset alias convention so future cron jobs don't repeat the invalid `"mcp"` value.
- Both jobs now verified against real tool call output as part of their fix, not just absence of errors.
- Follow-up items identified for upstream reporting (Hermes/Nous Research), not yet filed:
  - An unrecognized toolset name should surface somewhere durable — job status, delivered report, or a persisted log entry — rather than only ever printing to a stream that `quiet_mode` discards.
  - The MCP per-server circuit breaker trips on repeated parameter-validation errors from the calling model, not just genuine connectivity failures, which can disable a healthy server over an unrelated bug.
- Open follow-up: the discrepancy between an earlier confirmed-working invocation format and the current code's actual format was not fully resolved — flagged rather than guessed at, since this job's history already included multiple incorrect assumptions.

---

## Lessons Learned

1. **A monitoring job that fails silently is worse than one that doesn't run at all.** Twenty-plus fabricated "all clean" reports were delivered with full operator confidence and zero real data behind them — a false positive is strictly more dangerous than a visible gap, because it actively suppresses the instinct to check further.
2. **Config validation without a corresponding failure signal isn't real validation.** The invalid toolset name was technically "checked" on every run — the check just discarded its own warning under `quiet_mode`, which meant a broken config passed silently for the job's entire operational lifetime.
3. **Ruling a cause out is progress, not a dead end.** Two clean, verified restarts reproducing the identical failure looked like a wasted step in isolation, but definitively closed off the "stale binding" hypothesis and redirected the investigation into the actual source-level bug — negative results narrowed the search as much as the eventual positive one did.

---

*Environment: KONYKS-SERVER (Unraid) · Hermes-Agent · MCP toolset resolution (Unraid, Home Assistant, Nextcloud) · homelab-incident-reports*
