# Stale Integrity Pin Silently Degraded the Credential Guard for Nine Days

**Date:** 2026-08-05
**Severity:** P2 Medium
**Status:** Resolved
**Affected:** Claude Code sandbox credential guard (`credential_guard.sh`), `ssh_exec.mjs` SSH transport, sanctioned redaction path
**Duration:** 2026-07-24 → 2026-08-05 (9 days from hash divergence to fix; 7 days from confirmed diagnosis)

---

## Summary

The credential guard in the Claude Code sandbox maintains a node-script allowlist that pins each permitted script by absolute path plus SHA-256. On 2026-07-24, `ssh_exec.mjs` — the sandbox's only SSH transport to the server — was edited to add output redactors. The edit was correct and intended. The pin was not updated. From that moment the guard no longer recognised its own sanctioned transport and began treating it as an opaque interpreter, failing closed on any invocation whose argument text also named a credential-bearing path. The net effect was that the approved, redacting route to read configuration files was blocked while the underlying identity retained full permission to read them. The condition was diagnosed on 2026-07-27 but could not be self-remediated: the pin file is root-owned and write-denied to the sandbox user, which is precisely why it sat stale. Remediation on 2026-08-05 established the provenance of the hash change by diff before re-pinning, removed an unpinned and unredacted sibling transport discovered during recon, and verified the fix with a controlled before/after test.

---

## Timeline

| Time | Event |
|------|-------|
| 07-24 02:48 | `ssh_exec.mjs` rewritten to add output redactors; SHA-256 changes, allowlist pin not updated |
| 07-27 | Over-blocking observed and confirmed empirically; root cause identified as pin divergence |
| 07-27 → 08-05 | Condition persists — remediation requires root write to a managed path, deferred |
| 08-05 15:33 | Sandbox container starts (relevant to later cache question) |
| 08-05 15:35 | Recon begins: allowlist read byte-exact, live script hashed, provenance traced |
| 08-05 15:36 | Deny reproduced against the stale pin, verbatim reason captured |
| 08-05 15:44 | Unpinned sibling transport removed; backup taken outside the executable directory |
| 08-05 15:45 | Three stale permission grants stripped; recreation-vector memo corrected |
| 08-05 15:49 | Pin rewritten to the provenance-verified hash |
| 08-05 15:51 | Verified — both command forms now allowed, no restart required |

---

## Root Cause

The guard's node-script allowlist is a single-line file keyed on absolute path plus SHA-256:

```
/path/to/ssh_exec.mjs <64-char sha256>
```

Because the pinned script lives in a directory writable by the sandbox user, **the hash pin is the integrity control** — root ownership is not protecting that file. The guard is therefore correct to fail closed when the hash does not match.

The 07-24 redactor work changed the script from 1159 to 3071 bytes. The delta was purely additive: a header comment, a `REDACTORS` array, a `lineRedactor()` function, and the substitution of redactor-wrapped stream handlers for the raw `stdout`/`stderr`/`close` handlers. The SSH connection block — host, port, username, key path, algorithms, host verifier — was untouched between versions. The edit was legitimate; only the pin update was missed.

With the hash diverged, `ssh_exec.mjs` fell into the guard's opaque-interpreter branch. Observed deny reason:

> Command invokes an interpreter (`eval`/`sh -c`/`bash -c` with a dynamic payload, or `python3`/`perl -e`/`node -e`/`ruby`/`xargs`) whose actual behaviour this hook cannot statically verify, and a credential-bearing reference (…) is present in its argument text. Fails closed rather than assuming the nested/interpreted command is safe.

The blast radius was narrower than it first appeared. Plain reconnaissance against benign paths still worked; only the combination of the now-opaque interpreter *and* a credential-bearing path in the same argument text tripped the rule. The practical cost was the loss of the sanctioned redaction route — the one path the guard's own design intends operators to use.

**Directionally, this failed safe.** No credential was exposed by the divergence; the control became more restrictive, not less. The defect is availability of a security-approved workflow, and — more importantly — nine days of silent drift between an integrity control and the artifact it protects.

### Secondary finding: an unpinned, unredacted sibling transport

Recon surfaced a second script in the same directory, differing from the sanctioned transport by one character in its filename (hyphen rather than underscore). It carried the same SSH capability to the server as root and **no redaction at all**. It was not in the allowlist, so the guard treated it as opaque and failed closed on it — which is why it had gone unnoticed since June.

A search for callers came back negative across the session-start script, the tunnel process, both settings files, cron, and all scripts in the user's Claude directory. The only apparent caller was a wrapper in a since-deleted session scratchpad. However, the project settings file carried **three live permission grants** referencing it, including a wildcard pre-authorising arbitrary commands through the unredacted helper.

The risk here required careful calibration rather than an alarming first reading. **The permission allowlist does not bypass the credential guard** — the hook runs regardless, confirmed by audit-log denies on permission-granted invocations of that very script. The real exposure was narrower and more interesting: for commands the guard *allows* (no gate matched — a large set), output would reach the transcript unredacted, where the sanctioned transport would have redacted it. That is precisely the failure class of the 2026-07-24 `git remote -v` incident: a benign, unlisted command carrying a credential in its output.

---

## Remediation

Recon before change, with an explicit stop condition: if the live script's contents could not be accounted for, the work would stop and be reclassified as a security finding rather than maintenance. An unexplained change to a hash-trusted file in a user-writable path must never be laundered into a trusted pin.

**1. Provenance established by diff, not inference.** File history showed a single recorded edit on the same path: v1 hashing to exactly the currently pinned value, v2 byte-identical to the live file. The chain was closed end to end with no window for an unreviewed change, and the delta matched the known 07-24 redactor work.

One trap worth recording: the script's own header comment documents a URL-embedded-credential pattern, and the script's redactors match that documentation. Reading the file *through* its own transport renders the line redacted and produces a spurious mismatch. It must be read directly.

**2. Sibling transport removed**, with the backup deliberately placed outside the executable directory:

```bash
# a .bak beside the original stays runnable via `node ssh-exec.mjs.bak`
B="$HOME/.claude/removed/ssh-exec.mjs.removed.$(date +%Y%m%d-%H%M%S)"
cp -a "$T" "$B"; chmod 0400 "$B"; rm -f "$T"
```

**3. Three permission grants stripped** by exact whole-line match, each asserted to occur exactly once beforehand, out of 540 entries. Adjacent unrelated grants and the underscore-form grant for the sanctioned transport were explicitly verified present afterward. `JSON.parse` validated before and after; written truncate-in-place to preserve inode, owner, and mode.

**4. Recreation vector closed.** A project memo documented the access route using the *hyphen* form. Left uncorrected, a future session would have read that memo and rebuilt the file just deleted. The memo was corrected to name the sanctioned transport and given an explicit do-not-recreate note stating the reason and the backup location — closing the vector by explanation rather than by substitution alone.

**5. Pin rewritten**, guarded against drift between recon and apply:

```bash
LIVE=$(sha256sum "$S" | cut -d" " -f1)
[ "$LIVE" = "$EXPECTED" ] || { echo "ABORT: live hash differs from provenance-verified value"; exit 1; }
cp -a "$F" "$F.bak.$(date +%Y%m%d-%H%M%S)"
P=$(cut -d" " -f1 "$F")                 # read the path field, never retype it
[ "$P" = "$S" ] || { echo "ABORT: unexpected path field"; exit 1; }
printf "%s %s\n" "$P" "$LIVE" > "$F"    # truncate-in-place; sed -i would replace the inode
```

Truncate-in-place preserves the file's inode, ownership, and mode without explicit `chown`/`chmod`. Reading the path field from the existing file rather than retyping it keeps the format byte-identical — the file is 104 bytes and the guard parses it positionally.

---

## Verification

Two command forms were tested, on the hypothesis that the guard's redaction escape hatch requires the command text to *terminate* in redaction — meaning the original test form might deny on a different branch even after a correct re-pin.

| Form | Before pin | After pin |
|------|-----------|-----------|
| A — redaction first, credential path last | DENY 15:36:37 | ALLOW 15:50:56 |
| B — credential path first, redaction last | *never run* | ALLOW 15:51:20 |

**Three observed cells, not four.** The pre-pin state for form B was never captured and is now unrecoverable. This is stated rather than papered over: the causal claim rests on the form A pair, which is a valid controlled before/after on a single form, and form B's post-pin result is corroborating rather than load-bearing.

The predicted argument-order constraint **did not materialise** — form A did not fall through to the raw-read branch once the interpreter branch stopped firing. Both orderings are allowed. The two-form design still earned its place by preventing a partial success from being misread as a failure, but the constraint it was built around does not exist on this path.

**No container restart was required** — verified rather than asserted. The container's PID 1 start time predated both the recon and the pin write, with no recreation in between, and the allows landed after. The guard reads the allowlist at hook time.

---

## Prevention

- **Re-pinning is now part of the edit procedure for the transport script, not a separate discovery.** Any future edit — including the queued work to teach its redactors an additional credential shape — changes the hash and silently returns the guard to failing closed. This is the single highest-value change from this incident.
- Unpinned sibling removed; its permission grants stripped; the memo that would have recreated it corrected.
- Both the allowlist directory and the user's Claude directory were confirmed to sit on named volumes, so the fix and the removal are durable across the sandbox's per-session container recreation.
- The incident memo retains the original analysis below a dated observations block rather than overwriting it, so the diagnosis and the verification remain separately auditable.

**Open follow-ups:**
- The unredacted-output class this touched is not fully closed. The output-side redactor does not yet recognise every credential shape in circulation on this stack.
- The guard's host verifier accepts any host key on the hop to the server. Pre-existing baseline in both transports, deliberately out of scope here, backlogged.

---

## Lessons Learned

**An integrity pin is coupled to the artifact it protects, and nothing enforced that coupling.** The pin and the script were edited by different procedures on different schedules, so a legitimate improvement to one silently disabled the other. The failure was in the right direction — closed, not open — but "fails safe" is not the same as "fails visibly": nine days passed with a security-approved workflow unavailable and no signal beyond an operator hitting the wall. Controls that guard mutable artifacts need their update step inside the artifact's edit procedure, not in a separate memory.

**Deny message and logged category are different taxonomies, and one had been mistaken for the other.** The event displayed interpreter wording while the audit log recorded it under a different branch entirely. A prior memo had inferred the branch from the message text and got it wrong. Diagnostic surfaces that *look* authoritative but were never checked against the underlying record are how confident wrong conclusions get written down and inherited — the same class as an earlier "Resolved" status on this stack that suppressed investigation for eleven days.

**Deleting an artifact without deleting the instructions that recreate it is not remediation.** The memo pointing at the removed transport would have rebuilt it in the next session that read it. Documentation is executable in an environment where agents read it as guidance — it belongs in the blast-radius assessment of any deletion, not in a follow-up queue.

---

*Environment: KONYKS-SERVER (Unraid) · containerised Claude Code sandbox · credential guard managed-hooks layer · homelab-incident-reports*
