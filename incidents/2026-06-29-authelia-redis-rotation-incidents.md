# Authelia Secrets Rotation — Three Incidents: Redis Credential Exposure, CACHE_URL Colon-Stripping, SHA256 Newline Artifact

**Date:** 2026-06-29
**Severity:** P1 High (credential exposure); P1 High (CACHE_URL corruption); P2 Medium (verification artifact)
**Status:** Resolved
**Affected:** Redis shared credential (Authelia + Immich + honcho-api + honcho-deriver), honcho CACHE_URL
**Duration:** Exposure during Redis rotation; honcho auth failure ~20 min; all consumers confirmed correct by end of session

---

## Summary

A planned coordinated rotation of the Redis password shared between Authelia, Immich, and the Honcho
agent memory stack surfaced three distinct failures.

**Incident 1 (P1):** During the live Redis requirepass update, `redis-cli CONFIG GET requirepass`
was used to verify the new password was applied. This echoed the plaintext new credential into
session tool output — a persistent, network-transiting log. The established verification rule
(PING pass/fail only) was violated. Immediate remediation: a replacement credential was generated
and applied via `CONFIG SET requirepass` before any files were written; the original generated
value never reached any config file.

**Incident 2 (P1):** A sed pattern used to rewrite `CACHE_URL` values of the form
`redis://:pass@host` stripped the leading colon from the credential. The pattern
`s|redis://[^@]*@|redis://<new>@|` matched `redis://:pass@host` and replaced with
`redis://newpass@host`. redis-py parses `redis://pass@host` as `username=pass, password=None`,
causing WRONGPASS. The honcho-api container failed Redis authentication. The Immich container's
credential was in a discrete `REDIS_PASSWORD` env var (not a URL) and was not affected.
Remediation: honcho CACHE_URL rewritten via Node.js `URL` class.

**Incident 3 (P2):** A sha256 verification pipeline (`grep ... | cut ... | sha256sum`) added a
trailing newline before hashing — producing `sha256(value + "\n")` instead of `sha256(value)`.
This caused persistent false MISMATCH signals throughout the rotation. All values were correct;
verification via Node.js inside containers confirmed sha256 `3390b864...` for all four consumers.
The mismatch signals added operational confusion and delayed closure.

---

## Timeline

| Time (UTC) | Event |
|------------|-------|
| ~10:00 | Coordinated Redis rotation initiated. Ground truth established: 4 consumers found (Redis .env, Authelia secrets, Immich compose, honcho CACHE_URL) |
| ~10:06 | New Redis credential generated; `redis-cli CONFIG SET requirepass <new>` applied live |
| ~10:06 | `redis-cli CONFIG GET requirepass` called for verification — echoes plaintext new credential into tool output (Incident 1) |
| ~10:07 | Replacement credential generated; `CONFIG SET requirepass <replacement>` applied before any files written; original temp file deleted; PING confirms replacement accepted |
| ~10:10 | Files written with replacement value: Redis .env (sed), Authelia secrets (cat), Immich compose (sed), honcho .env (sed — colon-stripping bug applies here, Incident 2) |
| ~10:15 | Redis redeployed via Komodo; Authelia restarted |
| ~10:20 | honcho containers recreated; WRONGPASS on Redis auth (Incident 2 manifests) |
| ~10:25 | sha256 checks show `e514f406...` for redis .env — MISMATCH flagged (Incident 3 — false positive) |
| ~10:30 | honcho root cause identified: colon-stripping in CACHE_URL; Node.js rewrite applied |
| ~10:35 | honcho Redis PING OK; Hermes memory cache round-trip confirmed PASS |
| ~10:40 | Immich REDIS_PASSWORD verified inside container via Node.js: sha256 = `3390b864...`. AUTH+PING OK |
| ~10:45 | redis .env verified with `tr -d '\n'` before sha256: `3390b864...` (correct) — Incident 3 false positives explained |
| ~10:50 | All four consumers confirmed correct. Authelia 2FA login confirmed. Session closed |

---

## Root Cause

**Incident 1 — CONFIG GET credential echo:**
`CONFIG GET requirepass` returns the current requirepass value in plaintext. It was used with the
intent to confirm the new password was applied. The established verification protocol specified
PING-only verification (old credential → WRONGPASS; new credential → PONG). `CONFIG GET` was used
in place of PING, echoing the credential into session output.

**Incident 2 — Colon-stripping sed pattern:**
The pattern `s|redis://[^@]*@|redis://<new>@|` was designed to replace the credential segment of
a Redis URL. For `redis://:pass@host` (standard redis-py format: empty username, password only),
the match includes `://:<pass>@` but the replacement is `://<new>@` — omitting the colon. This
produces `redis://newpass@host`, which redis-py parses as `username=newpass, password=None`.
The correct pattern anchors to `://:`  (e.g., `s|redis://:[^@]*@|redis://:<new>@|`) to preserve
the colon. Better: use a URL parser rather than a regex for any credential embedded in a compound
value.

**Incident 3 — sha256 newline artifact:**
The pipeline `grep "^KEY=" file | cut -d= -f2- | sha256sum` appends a trailing newline (grep adds
`\n` to each output line), producing `sha256(value + "\n")`. File-based `sha256sum /path/to/file`
on a file with no trailing newline, and container-internal hashing via
`crypto.createHash('sha256').update(process.env.KEY)`, both omit the newline. The consistent
`e514f406...` throughout the session was always `sha256(3390b864_actual_value + "\n")`, not a
wrong value.

---

## Remediation

1. **Incident 1:** Replacement generated and applied before any file writes. Exposed temp file
   deleted. All four consumers received the replacement. No exposed value appears in any config
   file. Verification rule restated: PING-only, CONFIG GET permanently banned from procedures.

2. **Incident 2:** honcho CACHE_URL rewritten via Node.js `new URL(url).password` extraction,
   which correctly strips the colon prefix. honcho containers recreated; Redis PING OK; cache
   round-trip PASS. Immich unaffected (uses discrete `REDIS_PASSWORD` env var, not URL format).

3. **Incident 3:** No config changes needed — all values were correct. Verification pipelines
   updated: use `| tr -d '\n' | sha256sum` when extracting values via grep, or use file-based
   `sha256sum` directly.

---

## Prevention

- **CONFIG GET is permanently banned from rotation procedures.** PING (old cred → WRONGPASS; new
  cred → PONG) is the only acceptable Redis verification that does not echo the secret.
- **CACHE_URL sed patterns must anchor to `://:`** to preserve the empty-username colon required
  by redis-py. Preferred: use Node.js `URL` class for any CACHE_URL rewrite.
- **sha256 pipeline verification must strip trailing newlines** before hashing when using
  grep-based extraction: `cat file | tr -d '\n' | sha256sum`.
- **Coordinated multi-service rotations require an explicit consumer map before any credential is
  changed.** This rotation discovered a fourth consumer (honcho) mid-rotation. A pre-rotation
  grep across all compose files and appdata `.env` files prevents discovering consumers after
  the new credential is live.

---

## Lessons Learned

1. **Verification methods that return the secret value are not verification methods — they are
   exposure vectors.** `CONFIG GET requirepass` and PING both confirm the password was applied;
   only one echoes it. The implementation must use a channel that returns a boolean, not the
   secret.

2. **Regex-based secret substitution in structured URLs is fragile.** `redis://:pass@host` and
   `redis://user:pass@host` are both valid but look identical to a naive `[^@]*@` pattern.
   Whenever a secret lives inside a compound value, prefer a parser that understands the
   structure.

3. **A false MISMATCH signal is an incident.** The sha256 newline artifact produced the same
   wrong hash consistently across multiple values, making correct data look corrupted. This
   forced repeated re-verification and delayed closure. Verification tools should be calibrated
   before use in high-stakes rotations.

---

*Environment: KONYKS-SERVER (Unraid) · Redis (bitnami/redis) · Authelia · Immich · Honcho (honcho-api + honcho-deriver) · redis-py · homelab-incident-reports*
