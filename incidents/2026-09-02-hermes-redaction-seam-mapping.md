# Redaction Seam Mapping and Three Silent Failures During a Hermes-Agent Plugin Rollout

**Date:** 2026-09-02
**Severity:** P2 Medium
**Status:** Resolved
**Affected:** Hermes-Agent (plugin subsystem, credential redaction, tool-call policy)
**Duration:** Single session, ~21:00–00:40 local

---

## Summary

A tool-call policy and credential-redaction plugin was designed, built and installed for Hermes-Agent. The design rested on three assumptions about the framework's hook API taken from upstream documentation and a GitHub issue; all three were wrong for the deployed build, and each was caught only by reading source inside the running container rather than the project's main branch. Along the way the rollout surfaced three separate silent-failure modes — a plugin directory skipped for a missing manifest with no log line anywhere, an audit log writing to a non-existent root-owned directory behind a swallowing `except`, and three consecutive file edits that never reached disk while every downstream step reported success. None caused an outage; all three would have produced a control that appeared to work and did not. The plugin is now installed, enabled and verified live by observation, with each of its four halves exercised by a negative test.

---

## Timeline

| Time | Event |
|------|-------|
| ~21:00 | Design drafted from upstream hook documentation; plugin file authored |
| ~21:14 | Plugin `__init__.py` placed in the Hermes plugin directory |
| ~21:18 | `hermes plugins enable` returns "not installed or bundled" — no diagnostic |
| ~21:25 | Missing `plugin.yaml` identified as the cause; manifest added, discovery succeeds |
| ~22:00 | Source read against the deployed image digest refutes three design premises |
| ~23:30 | Plugin enabled; first negative test run shows native redaction, not the plugin |
| ~23:50 | Over-broad input rule found blocking the plugin's own redaction tests |
| ~00:10 | Audit log confirmed to have written zero lines since enable |
| ~00:25 | Edits applied on-host after three failed attempts; byte verification added as a gate |
| ~00:40 | All four halves verified live; policy gate, both redaction seams, and audit confirmed |

---

## Root Cause

### The three refuted premises

The plugin was designed around `post_tool_call` performing redaction, a "strictest verdict wins" arbitration across plugins, and a bug report describing five tools bypassing the hook chain. Reading source against the running container's image digest — not the project's main branch — showed:

1. **`post_tool_call` is an observer.** Its return value is ignored. Redaction belongs in `transform_tool_result`, which fires after it and before the conversation append, and in `transform_terminal_output` for the terminal tool specifically.

2. **The framework takes the first valid directive, not the strictest.** Evaluation is in registration order, so a laxer plugin registered earlier wins outright. Precedence has to be computed inside a single callback in a single plugin or it does not exist. The same applies to redaction: the first returned string wins, so patterns cannot be split across plugins.

3. **The referenced bug was closed, and its fix was asymmetric.** It restored `post_tool_call` for the bypassing tools by adding an emit in the executor; it added `transform_tool_result` nowhere. The redaction half of the reported gap was never fixed, and the bypass set is 14 tools in an inline dispatch chain plus 4 more early-returned by an agent-loop intercept set — not the 5 the issue named.

### Verified seam ordering

Native redaction is applied per-tool on output, not centrally. The dispatch order on this build:

```
pre_tool_call            <- receives RAW arguments; the display-side
                            argument redactor never mutates what
                            reaches the tool
registry.dispatch
  terminal tool: transform_terminal_output   <- plugin seam, RAW output
  terminal tool: native redaction            <- runs AFTER the plugin
post_tool_call
transform_tool_result    <- plugin seam, sees text the tool already
                            redacted itself
```

Two consequences. The terminal seam sees raw output and is the more valuable of the two. The tool-result seam sees sanitised text for any tool that redacts itself — but testing showed native redaction missing file content entirely, and MCP tool calls were confirmed reaching this seam, so it is load-bearing rather than redundant.

A structural gap remains that no plugin closes: the tool-result seam sits inside the main `try`, so a **propagating exception** or one of three blocked early-returns bypasses it. Tracebacks are where connection strings surface, and the framework's error sanitiser caps length without redacting. Filed upstream rather than worked around.

### Silent failure 1 — plugin directory skipped for a missing manifest

The plugin directory contained a valid `__init__.py` and no `plugin.yaml`. Discovery keys on the manifest, so the directory was skipped. `hermes plugins enable` returned "not installed or bundled" — the same message it returns for a name that does not exist at all — and nothing was logged. A directory containing working code, correctly named, correctly owned, in the correct location, was invisible with no signal distinguishing it from absence.

### Silent failure 2 — audit log writing nowhere

The audit path was set to a directory under `/var/log/` that does not exist in the container. The agent process runs as a non-root user and `/var/log` is root-owned, so it could not be created. The audit helper's exception handler logs at debug level, invisible at the normal log level:

```python
except Exception:
    logger.debug("audit write failed", exc_info=True)
```

Every write since enable had raised `FileNotFoundError` and been discarded. The audit trail read as empty rather than broken. The obvious fix — creating the directory — would have failed the same way after the next redeploy, since that path is in the container's writable layer rather than a volume.

### Silent failure 3 — edits that never reached disk

Three consecutive rounds of file edits were believed applied and were not. Each time, the container was restarted and a full test cycle run against unchanged code, producing results that looked like seam behaviour and were nothing of the kind. Byte-level verification — file size, a `grep -c` for the removed symbol, and mtime against container start time — identified it in seconds once used. A contributing factor: the host has no `python3`, so a scripted edit failed at the interpreter and the surrounding shell reported nothing useful.

### Design fault found by testing

An input-side rule blocked any tool argument whose *value* matched a credential shape. It fired on the plugin's own test payloads, blocking both the terminal echo and the file write intended to exercise the redactors. The result: no credential-shaped string could reach a tool result, so the output-side seams had no reachable test path. The shape filter had made the control it was paired with untestable, and would also have refused legitimate work — documentation, test fixtures, anything that merely mentions a credential shape.

---

## Remediation

1. Added the missing `plugin.yaml` manifest; confirmed discovery with `hermes plugins list` before spending a restart on `enable`.
2. Removed the input-side value-shape scan, retaining the argument-*key* check. Shape detection on the way in is not a boundary — it blocks the shapes already known and nothing else — and belongs in the redactor on the way out, where a match is repaired rather than refused.
3. Moved the audit log to a path owned by the agent user and inside the appdata bind mount, so it persists across container recreate rather than living in the writable layer.
4. Removed a rule that injected a working-directory default taken from an upstream example rather than observed on this host.
5. Adopted verify-then-restart as the standing procedure for this file: confirm the bytes changed, confirm the syntax parses using the container's interpreter, then restart, then test.

Verification was by observation, not by config read. Each of the four halves was exercised by a test designed to fail:

| Test | Result |
|------|--------|
| Read own config file in full | Blocked, with the plugin's own message |
| Terminal echo of a synthetic bearer token | Redacted, with a marker naming the pattern that fired |
| Write then read back a file containing a synthetic token | Redacted at the tool-result seam, same marker |
| Recursive delete | Escalated to the approval gate, not silently allowed |
| MCP read-only call | Audit entry present, confirming the seam covers MCP tools |

---

## Prevention

**Changed**
- Redaction annotates rather than substituting silently. Every rewritten result carries a marker naming how many values were touched and which pattern identifiers fired. A silent rewrite is indistinguishable from an unmodified read, which has previously turned a working control into a reported defect.
- The redactor's error path fails open by decision, but never silently: a failure emits a banner in the result, an error line in the agent log carrying a distinctive literal, and an audit entry. Fail-open with a quiet `except` is the worst of both.
- Audit log relocated inside the bind mount; verified writing.
- Non-exposure and redaction-is-not-ground-truth clauses added to the agent's operating document, verified by asking the agent to quote them back with a line number.

**Open follow-ups**
- The audit record's secret-shape field scans the tool's return envelope rather than its content, so it agrees with the redactor by construction and cannot catch a redactor miss. Either drop the field or scan the right object.
- A second, stale copy of the operating document exists in the repo tree and no longer matches the live one.
- The error-path redaction gap is upstream, not local.
- A drift check comparing the plugin's pattern set against the canonical shell redactor, in place of coupling the two at runtime.

---

## Lessons Learned

**A control that fails silently is indistinguishable from one that works, and this session produced three in a row.** A skipped plugin directory, an audit log writing to nowhere, and edits that never reached disk — each reported success at every layer a casual check would inspect. The only thing that caught any of them was refusing to trust the previous step's own report: verify the bytes, not the command's exit code; verify the log line, not the config file; verify by the artefact appearing, not by the absence of an error. Every silent failure here was cheap to detect and expensive to assume away.

**Reading source on a project's main branch is not evidence about the deployed build.** Three design premises survived a careful documentation read and died on first contact with the running container. Pinning the image digest and grepping in-place cost one session and reversed the plugin's architecture — including which of two seams is the primary one. The generalisation: for any control layered on someone else's runtime, the deployed artefact is the only authority, and the version skew is not hypothetical.

**An input-side filter can make its own output-side control untestable.** Blocking credential-shaped strings at the tool argument looked like defence in depth and was actually a test blocker: with it live, no credential could reach a tool result, so the redactor could never be exercised end to end. Controls that make their own verification impossible are worse than absent ones, because they buy confidence without evidence.

---

*Environment: KONYKS-SERVER (Unraid) · Hermes-Agent · plugin hook subsystem · homelab-incident-reports*
