# Credential Guard Enforced Text, Not Behaviour

**Date:** 2026-07-12
**Severity:** P1 High
**Status:** Root cause identified — remediation pending decision
**Affected:** `credential_guard.sh` PreToolUse hook (Claude Code, workstation-side); credential-bearing paths on KONYKS-SERVER
**Duration:** ~8 days as a live gap (2026-07-04 → 2026-07-12); design flaw present since the hook was written

---

## Summary

A PreToolUse hook (`credential_guard.sh`) was built to prevent AI-assisted sessions from reading raw
credential material — `.env` files, service config fields, Infisical bootstrap paths — by forcing those
reads through a safety library (`credential_safety.sh`) that redacts values instead of printing them.
The hook was written, hardened across two review rounds, and documented in `SYSOPS.md` as an enforced
control.

It was never an enforced control. The hook does not source the safety library, does not check that the
library exists, and does not verify that any safe-path function ever executed. It runs a regular
expression against the *text* of the proposed command and allows the call if that text contains a
`cred_*` function name. The library's own filename appears in the script only inside denial-message
strings — text printed to the caller after a block, never executed.

The gap was discovered incidentally: during an unrelated database backup task, an agent session went to
use the library, found `/root/.credential_lib.sh` absent from KONYKS-SERVER, and flagged it. Follow-up
investigation established that the missing symlink was not the bug — it was the symptom that made a
pre-existing design flaw visible. No evidence of credential exposure was found, and the discovering
session had already routed around the gap on its own (using Postgres local trust auth rather than
reading `.env`).

---

## Timeline

| Date | Event |
|------|-------|
| 2026-07-03 | `scripts/lib/credential_safety.sh` committed to `homelab-infra`. Commit message and `SYSOPS.md` both document the intended deployment: symlinked to `/root/.credential_lib.sh` on the server. |
| 2026-07-04 | Hook hardened (round 2): begins requiring textual `cred_*` evidence before allowing credential-adjacent commands. This is the start of the exposure window. |
| 2026-07-04 | `SYSOPS.md` records the hook being "bypassed the same day it was built," and separately notes the deeper structural fix — verifying execution rather than text — as *deliberately deferred*. The flaw was seen and shelved. |
| 2026-07-12 | Agent session performing a Honcho DB backup attempts to use the library, finds `/root/.credential_lib.sh` does not exist, flags it rather than routing around it silently. |
| 2026-07-12 | Read-only investigation opened. Root cause established: enforcement is textual, not behavioural. |
| 2026-07-12 | Remediation options drafted. Fix deliberately **not** applied pending a decision on ordering (see Prevention). |

---

## Root Cause

Two independent defects, one masking the other.

**1. The documented symlink was never created.**

`credential_safety.sh` exists exactly once, in the `homelab-infra` repo checkout on KONYKS-SERVER, added
in a single commit and never moved or removed. The convenience symlink at `/root/.credential_lib.sh` —
specified in the commit message, in `AGENTS.md`, and in `SYSOPS.md` — was never made. Other symlinks in
the same directory confirm the mechanism works there; this one simply wasn't run. No dangling symlink
exists anywhere on the host, ruling out "created, then broken by a repo move."

**2. The hook never depended on the library existing.**

This is the finding that matters. Across the hook's 219 lines, the only filesystem check is against its
own session-scoped state file — used for tracking scratch secrets, unrelated to the library. The library
is never sourced. Its existence is never tested. Every reference to it in the script sits inside a
`block()` message string: instructional text shown to the caller *after* a denial.

The actual gate is a regex over the proposed command's text:

```bash
LIB_EVIDENCE_RE='\bcred_(exists|field|env_field|toml_field|url_field|infisical_login|infisical_get|api_call)\b'

if ... grep -qE "$RAW_TEXT_CMD_RE" && ! grep -qE "$LIB_EVIDENCE_RE" ...
```

If the command text matches a raw-read pattern *and* contains no `cred_*` substring → block (exit 2).
Otherwise → allow (exit 0). There is no `set -e`; nothing fails closed, because nothing is executed to
fail.

The consequence is that the presence of a `cred_*` token anywhere in the command — in a real function
call, in a comment, in an `echo` string, or in a call that is immediately followed by an unguarded
fallback — satisfies the guard. A command shaped like:

```
source <library>; cred_field <config> <key> || cat <config>
```

is permitted today, on a host where the library does not exist, because the regex is satisfied long
before the fallback `cat` is reached. The safe call fails; the unsafe fallback runs; the guard reports
success.

The hook enforces *intent-shaped text*. It does not enforce credential safety. Creating the missing
symlink would not have changed this — it would only have made the safe path succeed when it happened to
be taken.

---

## Remediation

Not yet applied. Three items, deliberately ordered against the intuitive sequence:

**1. Structural fix — make enforcement behavioural (do this first).**

The guard should require evidence that the safe path *ran*, not that it was *typed*. The mechanism for
this already exists in the same file: the session-scoped state file used for scratch-secret tracking is
the only real filesystem check in the script. Extend it — have the library write a session-scoped marker
on successful execution, and have the guard check for that marker instead of a substring match. This
converts the control from "did you type the right words" to "did the safe path actually execute." No new
machinery required.

**2. Create the documented symlink.**

Deploy `/root/.credential_lib.sh` → the `homelab-infra` checkout of `scripts/lib/credential_safety.sh`,
so the functions the guard will now genuinely require actually exist.

**Ordering matters, and is the substantive judgement call in this report.** Doing (2) alone is the worst
available action: it makes the library callable, which makes the guard *appear* correct, which removes
the only pressure currently driving the real fix. It restores the illusion rather than the control. If
only one of these is ever done, it must be (1).

**3. Bounded transcript review.**

Eleven sessions in the exposure window textually reference `cred_*` patterns. This is an upper bound —
the hook's own denial message contains those same substrings, so blocked calls inflate the count. The
review is not a breach hunt; it is a search for the specific fallback shape (`cred_* || <raw read>`) in
commands that actually executed against credential-bearing paths.

---

## Prevention

- **Enforcement must be observable.** The hook currently emits no signal distinguishing "allowed because
  the safe path ran" from "allowed because the string matched." Post-fix, the marker mechanism provides
  exactly this signal, and it should be logged.
- **Deployment steps documented in a commit message are not deployment steps performed.** The symlink was
  described identically in three places (`commit`, `AGENTS.md`, `SYSOPS.md`) and executed in none. Any
  control whose deployment is documented rather than automated needs a verification step that asserts the
  end state, not the intent.
- **Deferred structural fixes need an expiry.** `SYSOPS.md` recorded this exact flaw on 2026-07-04 and
  marked it "deliberately not done yet." Eight days later it was rediscovered from scratch, by accident.
  A known-deferred weakness in a security control is a scheduled item, not a note.

**Open follow-ups:**
- Apply structural fix (1), then symlink (2), then scoped review (3).
- One unresolved thread: `SYSOPS.md` records `cred_env_field` executing successfully against a real
  `.env` file on 2026-07-04, which the "symlink never existed" finding cannot by itself explain. Most
  likely that session sourced the library directly from its repo path, bypassing the symlink entirely —
  plausible, unproven from filesystem forensics alone. Worth confirming; not worth blocking on.

---

## Lessons Learned

**A control that reports healthy is not a control that works.** This is the fourth instance of that exact
failure class in this environment in a single month, and the pattern is now the finding:

| Control | Reported | Actually did |
|---------|----------|--------------|
| Hermes cron fleet | 5 jobs present, healthy | Every MCP call silently failing on an invalid toolset name, for weeks |
| GitOps image deploys | Deploy successful | Nothing — a populated named volume masked the image, freezing the code for two months |
| `credential_guard.sh` | Enforced, hardened ×2, documented | Regex match on command text |
| `.credential_lib.sh` | Documented as deployed | Never created |

Each of these passed every check that was actually being run. Each was caught by an *inconsistency*, not
by an alarm: a version number that shouldn't have been possible after a fresh pull; a library that
should have existed. The common root is that verification targeted the *artifact* (the job is listed, the
deploy succeeded, the hook is installed) rather than the *behaviour* (the job's tools resolve, the
running code changed, the guard blocks an unsafe call).

**Security controls degrade toward theatre unless they are adversarially tested.** The guard was hardened
twice, and both rounds improved its *messaging and pattern coverage* — neither round asked the only
question that mattered: *what does a bypass look like?* One adversarial read of the control's own logic
found the bypass in minutes. That read should have been part of hardening, not a consequence of an
unrelated backup task.

**The discovery mechanism is the reusable asset here.** The gap surfaced because an agent session, asked
to take a database backup, chose to flag a missing dependency rather than silently route around it — even
though it had already found a clean alternative path and could have said nothing. Systems that make it
cheap and expected to surface "this thing I was told exists, doesn't" find this class of defect. Systems
that only surface task failure never will.

---

*Environment: KONYKS-SERVER (Unraid) · Claude Code PreToolUse hooks · homelab-infra · homelab-incident-reports*
