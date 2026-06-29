# Weekly IP Scan — Two Independent MCP Failure Classes Found in Same Script

**Date:** 2026-06-28 to 2026-06-29
**Severity:** P2 Medium
**Status:** Resolved
**Affected:** ip-scan.sh Unraid-MCP integration, infisical / infisical-db / infisical-redis container inspection coverage
**Duration:** Both bugs pre-existed for an indeterminate period; the combined effect was that the MCP container scan had never successfully completed a full run

---

## Summary

Investigation of the silently-failing MCP section of ip-scan.sh found two independent bugs. The
first was a protocol mismatch: the script initialized its MCP session using a GET request against an
endpoint that now returns 400, relying incidentally on the fact that the server includes a session ID
in the error response's headers. This worked well enough for the session handshake, but was undefined
behavior. The second bug was more consequential: when inspecting container details, the script passed
container names with the leading `/` stripped, causing the MCP server to perform prefix matching
rather than exact lookup. For the `infisical` container, this matched three Docker names
(`/infisical`, `/infisical-db`, `/infisical-redis`) and the server returned a plain-text ambiguity
error rather than JSON. The script's only guard for empty results did not catch a non-empty,
non-JSON string, so `json.loads()` threw a JSONDecodeError that crashed the entire Python block,
leaving containers at index 17 through 19 unscanned on every run.

---

## Timeline

| Time | Event |
|------|-------|
| 2026-06-28 | MCP section of ip-scan.sh confirmed silently failing on every run (see hermes-cron-fleet-audit) |
| 2026-06-28 | Phase 1: GET /mcp behavior characterized — returns 400, but server includes `mcp-session-id` header in the 400 response; script harvests this header and uses it for subsequent calls |
| 2026-06-28 | Phase 1: Correct MCP protocol identified — POST initialize (no prior session ID) returns 200 with session ID in response header; this is the MCP streamable-HTTP transport spec (2025-03-26) |
| 2026-06-28 | Phase 1: POST-based approach tested directly; session ID from POST response accepted for all subsequent calls |
| 2026-06-28 | Exact script block extracted and run as a standalone file; same JSONDecodeError observed — rules out heredoc encoding as a factor |
| 2026-06-28 | Per-container instrumentation added; all 17 containers before `infisical` succeed; `infisical` call returns 285 bytes with `text` field containing a plain-text error message, not JSON |
| 2026-06-28 | MCP server error identified: `"Internal error: Container identifier 'infisical' is ambiguous — matches: /infisical, /infisical-db, /infisical-redis"` |
| 2026-06-28 | Root cause confirmed: `lstrip("/")` on container names causes prefix matching; `infisical` matches all three Infisical-stack containers |
| 2026-06-29 | Fix applied: `get_session()` rewritten to use POST initialize; details calls use `raw_name` (full `/name`); `try/except json.JSONDecodeError` added as defensive guard |
| 2026-06-29 | Instrumented verification: all 20 containers return OK, `infisical` (index 17) confirmed scanned, `JSONDecodeError containers: []` |
| 2026-06-29 | Full script run: clean exit, no stderr, correct log entry written |

---

## Root Cause

**Bug 1 — Wrong MCP transport protocol:**

The MCP server had been updated to implement the streamable-HTTP transport (spec 2025-03-26). The
correct initialization sequence is a POST to `/mcp` with no prior session ID, carrying a standard
`initialize` JSON-RPC request; the server returns the assigned `mcp-session-id` in the response
header. The script instead sent a GET to `/mcp` (the old SSE transport pattern), which the server
rejects with 400 — but also includes a fresh `mcp-session-id` in the error response headers. The
script extracted this header and used the ID for subsequent calls. The server accepted these calls
because the ID itself was valid; the 400 rejection was on the GET method, not on the session. This
was behaviorally equivalent to the correct protocol in the cases tested, but was undefined behavior
dependent on a server implementation detail that could change. The separate `# Initialize MCP
session` block that followed `get_session()` was redundant: it re-sent the initialize request (now
with a session ID), but initialization had already implicitly occurred when the server assigned the ID.

**Bug 2 — Container name prefix ambiguity causing non-JSON error text passed to json.loads:**

The script extracted container names with `lstrip("/")`, removing the leading `/` that Docker
includes in all container name strings. For most containers, the stripped name is unambiguous: there
is no other running container whose name begins with `obs-grafana`, `crowdsec`, `Authelia`, etc.
For `infisical` (index 17 of 20), the stripped name is a prefix of `/infisical-db` and
`/infisical-redis`. The MCP server's container lookup performs prefix matching on the stripped
identifier and, finding three matches, returns an error as plain text in the `text` field of an
otherwise-successful MCP result: `"Internal error: Container identifier 'infisical' is ambiguous —
matches: /infisical, /infisical-db, /infisical-redis. Use..."`.

The script's only guard before parsing was `if not det_text: continue` — which only catches empty
strings. A 168-character plain-text error message is truthy. `json.loads(det_text)` throws
`JSONDecodeError: Expecting value: line 1 column 1 (char 0)`, the outer `except Exception` catches
it, prints to stderr, and exits the Python block. Containers at indices 17, 18, and 19 were never
inspected.

These two bugs existed independently. Either would have been sufficient to suppress MCP output on
most runs; the second was the active crash cause for the specific container list present at the time
of investigation.

---

## Remediation

Three targeted changes were applied to the MCP Python block within ip-scan.sh:

1. **`get_session()` rewritten** to POST initialize with no prior session ID, read `mcp-session-id`
   from the response header, and drain the response body. The separate post-`get_session()` initialize
   block was removed as redundant — initialization now happens inside `get_session()`.

2. **Details calls use `raw_name`** (the full `/name` string as returned by the list endpoint) rather
   than the `lstrip("/")`-stripped version. The stripped `name` is retained for display purposes only.
   Full names match exactly; there is no prefix ambiguity.

3. **`try/except json.JSONDecodeError` added** around `json.loads(det_text)` as a per-container
   defensive guard. This is belt-and-suspenders: with `raw_name` in use, the ambiguity error cannot
   occur for any container currently in the fleet. The guard ensures that if a future container
   configuration triggers a different non-JSON MCP error, the crash is contained to that one container
   rather than killing the entire scan.

Scope of the prefix-matching bug was confirmed before applying the fix: all 20 running containers were
checked; `infisical` was the only name that was a prefix of another running container's name.

---

## Prevention

- **Exact-match identifiers by default:** when querying an API for a specific resource by name, always
  use the canonical identifier the API returns, not a normalized form. The MCP server returned full
  `/name` strings in the list response; using those exact strings for subsequent calls eliminates an
  entire class of ambiguity errors.
- **Narrow exception guards:** a `try/except Exception` that catches everything and continues silently
  should either be narrowed to the specific exceptions that represent recoverable conditions, or should
  write the exception to the same log path as normal output so it is discoverable. The fix applied both
  principles: `json.JSONDecodeError` is the specific recoverable case; any other exception still
  reaches the outer handler.
- **MCP protocol version awareness:** the server's transport changed from SSE (GET-based) to
  streamable HTTP (POST-based initialize). Any client code targeting this server should be updated
  when the transport changes rather than relying on incidental behavior of error responses.

---

## Lessons Learned

1. **Relying on a behavior that appears in an error response is not the same as relying on a behavior that appears in a success response.** The GET /mcp returning a session ID in its 400 error headers was not a documented contract — it was an incidental server behavior. The script was functioning, but its correctness depended on an implementation detail of the error path. The correct protocol, documented in the MCP streamable-HTTP spec, did not require any such dependency.

2. **A guard that catches empty strings is not a guard against non-JSON strings.** The `if not det_text: continue` check was written with the assumption that the only failure mode for `mcp_post()` was returning nothing. The MCP server can return non-JSON content as a valid, non-empty `text` field inside an otherwise-successful response. Validating that a result is parseable before parsing it — either with a length check, a format check, or a `try/except` — is a necessary second layer distinct from checking that the result exists at all.

3. **Testing on a representative dataset is not the same as testing on the full dataset.** The GET-based session initialization happened to work for the first 17 containers. The 18th container (infisical) had a name that triggered a different code path on the server. The bug was latent in the script from the moment the Infisical stack (with its three containers) was deployed. Testing against a fleet with no such naming pattern would not have caught it.

---

*Environment: KONYKS-SERVER (Unraid) · Hermes-Agent · Unraid-MCP · ip-scan.sh · MCP streamable-HTTP (2025-03-26) · homelab-incident-reports*
