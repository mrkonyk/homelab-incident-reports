# Multi-System Failures During Routine Maintenance

Date: 2026-06-21
Severity: P1 High
Status: Resolved
Affected: Unraid-MCP, Hermes-Agent (Telegram gateway), GitHub integration (PAT revocation), Komodo Periphery alerting
Duration: ~4 hours (single session)

---

## Summary

During a maintenance session to add Unraid icon labels to the final two Komodo-managed
stacks, three pre-existing failures were discovered and remediated: Unraid-MCP had been
crash-looping for 98 restarts due to a breaking security gate in an image update; GitHub
PATs had been silently auto-revoked via a two-step credential leak through context
compression and Anthropic API traffic; and the Hermes-Agent gateway had a double-start
race condition on every container restart. A fourth issue emerged mid-fix: the race fix
introduced a post-redeploy regression where the gateway failed to start after every
Komodo auto-deploy. All four were root-caused and fixed within the same session. Telegram
round-trip confirmed as final verification.

---

## Timeline

| Time | Event |
|------|-------|
| Before this session (exact date unverified) | GitHub PATs silently auto-revoked via credential leak through context compression; revocation predated its discovery in this session |
| 2026-06-21 ~13:00 UTC | Session begins; target: icon labels for hermes-agent and unraid-mcp |
| 2026-06-21 ~13:00 UTC | Unraid-MCP discovered crash-looping (98 restarts) |
| 2026-06-21 ~13:30 UTC | Unraid-MCP fix deployed; RestartCount reset to 0 |
| 2026-06-21 ~14:00 UTC | PAT exposure sites found and scrubbed |
| 2026-06-21 ~14:30 UTC | Loki alert rules extended: clone-failure pattern expanded, new WARN-rate rule added |
| 2026-06-21 ~15:00 UTC | Icon labels committed for both stacks |
| 2026-06-21 ~15:30 UTC | Hermes-Agent gateway race condition root-caused |
| 2026-06-21 15:54 UTC | Fix v1: `command: sleep infinity` — race eliminated; single clean gateway start |
| 2026-06-21 16:00 UTC | Komodo 5-min auto-poll redeploys container; gateway fails to auto-start (regression) |
| 2026-06-21 ~16:07 UTC | Gateway manually started; regression diagnosed |
| 2026-06-21 16:40 UTC | Fix v2 deployed: `sh -c 'sleep 10 && hermes gateway start; exec sleep infinity'` |
| 2026-06-21 ~16:41 UTC | Telegram round-trip confirmed after clean auto-redeploy with no manual intervention |

---

## Incident 1: Unraid-MCP Crash Loop

**Severity:** High — Unraid MCP tooling unavailable for the duration.

**Root cause:** The upstream `jmagar/unraid-mcp:latest` image added a security gate: if
`DISABLE_HTTP_AUTH=true` and `HOST=0.0.0.0` are both set, the process exits unless
`TRUST_PROXY=true` is also present. The compose file had the first two but not the third.

**Discovery:** 98 restarts; `docker logs` contained an explicit message naming the missing
env var.

**Fix:** Added the missing `TRUST_PROXY` environment variable to the container
configuration. Committed and redeployed.

**Lesson:** Image updates can introduce silent behavioral gates that don't surface as
deprecation warnings. Renovate weekly PRs will flag future version bumps before they
reach production.

---

## Incident 2: GitHub PAT Auto-Revocation

**Severity:** Medium — git push and Hermes GitHub MCP both broken.

**Root cause — two-step exposure chain:**

*Step 1 — Credential surfaced in tool output:*
A `git push` was run with the PAT embedded in the remote URL
(`https://<token>@github.com/...`). Git echoed the authenticated URL in its output. That
output landed verbatim in a Bash tool result inside the AI-assisted session's conversation
context. A `sed` redaction on push stdout was the intended safeguard, but git can surface
the URL in stderr or progress output that a stdout-only redaction does not catch.

*Step 2 — Context compression swept the tool result into API traffic:*
The session grew long enough to hit context limits. Context compression generated a summary
by passing the full conversation — including the tool result containing the plaintext token
— through the Anthropic API. GitHub's secret-scanning partnership detected the `ghp_`
pattern in that API traffic and auto-revoked both tokens. The revocation was silent: no
notification, no immediate failure, just credentials that quietly stopped working and
weren't noticed until something downstream broke.

**Remediation:**
- Plaintext token appearances scrubbed from two locations in the repository and filesystem.
- New PATs generated; all credential stores updated.
- Push procedure hardened: token extracted server-side, kept in shell variable, remote URL
  reset to tokenless form immediately after push, redaction extended to cover combined
  stdout and stderr.

**Prevention — the broader lesson:**

The narrow lesson here is "don't let credentials appear in push output." The broader lesson
is more important: **credential redaction has to cover every surface where a credential can
appear in tool output — including outputs nobody anticipated would contain one — because
anything present in a long-running session's context window is a candidate for context
compression and onward API transmission.**

This changes the threat model. The risk is not limited to credentials deliberately printed
or pasted into a conversation. It extends to credentials that surface transiently in tool
output — in a subprocess's progress line, in an error message, in a URL echoed by a library
— during a session that runs long enough to hit context limits. By the time context
compression fires, the tool result is already in the window, and its contents go wherever
the summary goes.

Specific rules that follow:
- Embed credentials in shell variables; never let them reach the printable output of any
  command.
- If a command is known to echo a URL (like `git push` with an HTTPS remote), pipe its
  combined output through a redaction filter covering both stdout and stderr before the
  result is captured.
- Reset any embedded-credential remote URL to the tokenless form before the session
  accumulates more context after a push.
- Treat partial or annotated tokens (e.g., `[REDACTED-ghp_abc123...]`) as plaintext — the
  literal string triggers detection regardless of surrounding text.
- If a credential stops working shortly after a long session, assume auto-revocation and
  rotate rather than debugging auth.

---

## Incident 3: Hermes-Agent Gateway Double-Start Race + Post-Redeploy Silence

**Severity:** Medium — log noise on every container restart; Telegram goes silent after
every Komodo auto-redeploy.

### Phase 1: Double-start race

**Root cause:** Two competing gateway starters fire simultaneously on every container boot:

1. **`container_boot`** (a container init script) reads a persistent state file from the
   appdata volume, recreates an s6 `gateway-default` service on the ephemeral tmpfs
   service tree, and starts the gateway if prior state was `running`.
2. **The compose `command:`** — the image default was `gateway run --no-supervise`, which
   also starts a gateway process via the ENTRYPOINT wrapper.

One wins the gateway lock; the other enters a restart loop under s6-supervise, emitting
"Gateway already running" on every iteration indefinitely.

**Fix v1:** Changed `command:` to `sleep infinity`. The CMD becomes a no-op;
`container_boot` is the sole starter. Race eliminated — single gateway banner per boot,
no loop noise.

### Phase 2: Post-redeploy silence (regression from Fix v1)

**Root cause:** Komodo's 5-minute auto-poll fired 6 minutes after Fix v1 was deployed.
Docker's graceful shutdown sequence lets the gateway write `gateway_state: stopped` to its
state file before exiting. The next container's `container_boot` reads `stopped` and
registers the s6 service with a `down` flag — correctly, from its perspective. The gateway
does not start. Telegram goes silent.

With `sleep infinity` as the CMD, the system is entirely dependent on `container_boot`'s
prior-state logic, which cannot distinguish "user deliberately stopped the gateway" from
"Docker shut the container down for a redeploy."

**Final fix:**
```yaml
command:
  - sh
  - -c
  - "sleep 10 && hermes gateway start; exec sleep infinity"
```

The 10-second sleep lets s6 complete initialization. `hermes gateway start` is idempotent
— exits 0 if already running — so it is safe in both paths:
- `container_boot` already started the gateway (prior state `running`) → no-op
- `container_boot` skipped it (prior state `stopped`, clean redeploy) → starts it

Verified: after a clean Komodo auto-redeploy with no manual intervention, the gateway
started automatically and a Telegram round-trip confirmed message delivery.

---

## Monitoring Improvements

**Clone-failure alert (updated):**
The existing Komodo repo-pull failure alert matched only one error string and missed clone
failures that occur on first-time stack registration. Pattern expanded to cover both error
paths.

**Sustained deploy-failure alert (new):**
A new alert fires when Periphery's WARN message rate exceeds a data-calibrated threshold
sustained over a 20-minute window. Threshold was set based on observed healthy-state data
(zero WARNs per day) versus failure-state data (~13 WARNs per poll cycle per failing
stack). Routes to priority push notification. Catches sustained deploy failures that
generate ongoing WARN-rate noise rather than a single discrete loggable event.

---

## Changes Made

**Unraid-MCP stack configuration:** Added the missing `TRUST_PROXY` environment variable
and the Unraid icon label.

**Hermes-Agent stack configuration:** Added the Unraid icon label. The `command:` field
was changed twice during the session — first to `sleep infinity` to eliminate the gateway
race, then to the final `sh -c` wrapper that guarantees gateway startup on every boot
regardless of prior recorded state.

**Loki alert rules:** Expanded the existing clone-failure rule to match both error strings.
Added a new sustained-WARN rate rule for Periphery with a data-calibrated threshold.

**Operations documentation:** Removed a credential artifact that had survived a prior
failed redaction attempt. Added a section documenting the Hermes-Agent gateway startup
mechanism, the race condition, the fix, and the state-file behavior that caused the
post-redeploy regression.

---

## Open Items

- **HA MCP stale token:** `home_assistant` MCP fails initial connection on every gateway
  start (3 retries, gives up). Pre-existing; not addressed this session.
- **Vaultwarden:** Komodo stack secrets exist only on disk in a gitignored file. Should be
  mirrored to Vaultwarden.
- **cAdvisor mount persistence:** `mount --make-rshared /` not persistent across Unraid
  reboots. Needs entry in the host startup script.
