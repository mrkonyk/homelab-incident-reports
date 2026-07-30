# SessionEnd Redaction Hook Silently Blocked by Policy for Five Days

**Date:** 2026-07-29
**Severity:** P2 Medium
**Status:** Resolved (control re-enabled in observation mode; write-enable pending evidence)
**Affected:** Claude Code sandbox container — transcript redaction hook (`scrub_session_transcript.mjs`)
**Duration:** 5 days (2026-07-24 → 2026-07-29) during which the control never executed

---

## Summary

A credential-redaction hook written on 07-24 to scrub session transcripts at close had never
executed a single time. It was declared in user settings, but a hardening change applied twelve
days earlier had enabled `allowManagedHooksOnly`, which silently blocks all user-defined hooks
in favour of hooks declared in managed policy settings. The control was believed to be in place;
it was inert. No credential exposure resulted — an independent same-day sweep of all 63 session
transcripts confirmed zero live values on disk — which is why this is classified P2 rather than
higher. The remediation moved the hook into managed settings, where it fired for the first time.
Its first real execution caught zero credentials and instead rewrote two occurrences of a source
comment that merely *illustrated* a credential format, corrupting documentation it had no business
touching. The control was converted to a non-destructive observation mode pending evidence about
what its patterns actually match.

---

## Timeline

| Time | Event |
|------|-------|
| 07-12 | Sandbox hardening sets `allowManagedHooksOnly` in managed settings — pre-dates the hook |
| 07-24 03:00 | `scrub_session_transcript.mjs` authored and wired into user settings; never fires |
| 07-29 ~17:50 | Review of a separate redaction-coverage gap surfaces the hook as the candidate fix |
| 07-29 ~18:00 | Pre-change recon: script read in full, managed settings baselined, allowlist checked |
| 07-29 18:07 | Hook copied to managed hook directory, declared under `SessionEnd`; smoke test fires |
| 07-29 18:10 | First real execution — reports a transcript rewritten |
| 07-29 18:20 | Second execution — fires but finds no target file |
| 07-29 ~18:28 | Inspection: zero credential matches, two documentation rewrites |
| 07-29 ~18:32 | False positive confirmed by diffing the transcript copy against live source |
| 07-29 ~18:36 | Control redeployed in dry-run mode with per-pattern counters |

---

## Root Cause

Two independent faults, discovered in sequence.

**1. The hook was declared in a settings tier that policy had disabled.**

Managed settings carried:

```json
{
  "hooks": {
    "PreToolUse":  [ { "matcher": ".*", "hooks": [ /* credential guard */ ] } ],
    "PostToolUse": [ { "matcher": "Bash|TaskOutput", "hooks": [ /* post-guard */ ] } ]
  },
  "allowManagedHooksOnly": true
}
```

`allowManagedHooksOnly` blocks hooks from user, project, and local settings files. The scrub hook
lived in user settings, so it was never registered. The failure was completely silent: no error,
no warning, no log line. The obvious verification surface was also useless — the `/hooks` menu
reported *"0 hooks configured"* both before and after the fix, even though the two managed guard
hooks in the same file were demonstrably enforcing. That menu does not enumerate managed-settings
hooks at all, so it could not distinguish "blocked" from "working" in either direction.

**2. The hook's pattern set could not match the shapes that motivated enabling it.**

Reading the script in full before wiring it revealed three substitutions only: two GitHub token
formats and a URL-userinfo form (`scheme://user:pass@host`). The redaction gap this work set out
to close involved a tokenised internal service URL where the secret is an opaque *path segment* —
no `@`, therefore unmatchable by the userinfo pattern. Had the hook been firing perfectly for the
whole five days, it would have caught **zero** of the 122 occurrences found by that day's manual
sweep. The firing fault and the coverage fault were entirely independent, and fixing one did
nothing for the other.

A third fault surfaced only on execution. The userinfo pattern matched this line from a sibling
script's header comment:

```
//   server/refresh), and scheme://user:token@host (URL-embedded creds).
```

That is documentation describing a credential format, not a credential. The hook rewrote it to
`scheme://SCRUBBED@host` inside the stored transcript. The script had no equivalent of the
placeholder and entropy guards already present in the shell-side redactor, so it could not
distinguish a secret from an example of one. The mangled text was, with some irony, the header
of the tunnel redactor documenting the same substitution it had just been victim of.

---

## Remediation

Pre-change verification, because a malformed managed settings file would take the credential
guard down with it:

```bash
cp -p /etc/claude-code/managed-settings.json \
      /etc/claude-code/managed-settings.json.bak.$(date +%Y%m%d-%H%M%S)
node -e "JSON.parse(require('fs').readFileSync('/etc/claude-code/managed-settings.json','utf8'))"
```

The hook was copied into the managed hook directory as root-owned `750 root:node` — matching the
convention of the existing guard scripts — rather than being referenced in place. Policy keys on
*where a hook is declared*, not on who owns the script, so pointing managed settings at a
sandbox-writable file would have run it while handing back the integrity property the original
hardening bought.

The `SessionEnd` declaration:

```json
"SessionEnd": [
  { "matcher": "", "hooks": [
      { "type": "command", "command": "node",
        "args": ["/etc/claude-code/scrub_session_transcript.mjs"],
        "timeout": 60 }
  ]}
]
```

The explicit `timeout` is load-bearing. `SessionEnd` hooks share a 1.5-second budget unless a
longer per-hook timeout raises it — an order of magnitude below the default that applies to other
hook events. Without it, a scrub of a multi-megabyte transcript would be cancelled partway,
producing a truncated file rather than merely an unredacted one. That would have been a second
silent-failure path installed directly on top of the first.

Verification proceeded in two stages deliberately kept separate: a piped synthetic event to prove
the script executes at all, then a real session close to prove the harness invokes it. Separating
them matters — "the script is broken" and "the harness never calls it" are different problems, and
a combined test cannot tell them apart.

After the false positive was confirmed, the control was redeployed in observation mode: it counts
matches per named pattern and logs `WOULD-SCRUB <path> <pattern>=<count>`, but performs no writes.

---

## Prevention

**Changed**

- Hook relocated to managed settings, the only tier policy permits.
- Liveness marker written by the control itself on every invocation — the only available proof of
  registration, given the hooks menu cannot report on managed hooks.
- Target path now taken from the harness-supplied `transcript_path` rather than reconstructed from
  a hardcoded project directory. The hardcoded form had already failed once in observed traffic.
- Missing targets logged explicitly, removing the ambiguity between "ran and found nothing to do"
  and "ran and looked in the wrong place" — the two had been indistinguishable.
- Every swallowed error path (`catch {}` on stdin read, directory read, file read, and write)
  replaced with a logged reason.
- Explicit hook timeout set against the reduced `SessionEnd` budget.
- Writes disabled pending evidence.

**Open follow-ups**

- Add placeholder and entropy guards so credential-shaped *examples* in documentation are not
  rewritten.
- Constrain the userinfo pattern's second character class, which currently permits `/` and can
  therefore run past the authority component into a path or query. Not the cause of the observed
  false positive, but a live defect capable of destroying ordinary URLs containing a later `@`.
- Extend coverage to the tokenised service-URL shape that motivated this work, plus the session
  token shape identified on 07-26.
- Re-enable writes only after logged observations show what the patterns match in real traffic.
- Two occurrences of rewritten documentation in one stored transcript are unrecoverable; no backup
  was taken before the first execution.

---

## Lessons Learned

**A control's declaration is not its execution, and the difference can be invisible for weeks.**
Every artifact said this hook was in place: it existed, it was syntactically valid, it was wired
to the right event, and it was referenced in the settings file. It had simply never run. The
policy that disabled it was applied twelve days before the hook was even written, so nothing about
authoring it looked wrong at the time. Trust boundaries changed underneath a component that was
added later — a class of failure that no amount of reviewing the component itself would surface.
Controls need a liveness signal they emit themselves, because the platform's own status surface
reported the same "0 hooks configured" whether the control was blocked or working.

**Enabling an untested control is a change, not a restoration.** The instinct on finding a dead
safety mechanism is to switch it on and call the gap closed. Its first execution mutated a file,
kept no backup, and logged nothing about what it had changed — so "what did it just do?" was
unanswerable from the control itself and had to be reconstructed by diffing stored output against
live source. Reversing the order — observe first, write later — costs one session and converts a
control that acts on guesses into one designed against evidence. That discipline had already been
applied to the manual sweep earlier the same day; it had not been applied to the automated
equivalent.

**Coverage faults and execution faults are independent, and fixing one advertises progress on the
other.** Reading the pattern set before enabling the hook was what revealed it could not match
the shapes it was being enabled to catch. Had that read been skipped, the session would have ended
with a firing hook, a closed ticket, and the original exposure entirely untouched — a false
resolution more dangerous than the known gap it replaced, because it would have stopped anyone
from looking again.

---

*Environment: Claude Code sandbox container · managed policy settings · transcript redaction ·
homelab-incident-reports*
