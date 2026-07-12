# Credential Guard — Remediation Amendment

**Amends:** `2026-07-12-credential-guard-textual-enforcement.md`
**Date:** 2026-07-12 (same day; remediation session)
**Status of parent report:** was *Root cause identified — remediation pending decision*.
**Status now:** Remediated in layers; residual limitations disclosed (see Residual Risk). The
control is now a *sound lint rule on a real immutability boundary*, not a boundary against an
adversarial agent.

---

## Why this is an amendment, not a new report

The parent report ended at diagnosis: the guard enforced command *text*, not credential
*behaviour*, and lived in a directory the constrained agent could freely rewrite. This
amendment records the remediation — which did not proceed as the parent's three-item plan
anticipated. The plan's first item (a session-marker check) turned out to be structurally
impossible in a PreToolUse hook, and applying the fix surfaced **six further defects**, each
invisible until the layer beneath it was made to actually run.

That sequence is the substance of this amendment. Every fix that worked exposed the next thing
that didn't — which is the parent report's own thesis, demonstrated against itself.

---

## Correction to the parent's remediation plan

The parent proposed, as fix (1): *have the library write a session marker on success; have the
guard check the marker instead of a substring.*

**This cannot work as stated.** The guard is a **PreToolUse** hook — it fires *before* the
command runs. It cannot check whether the safe path executed, because execution hasn't
happened yet. A marker check would also deadlock the first legitimate `cred_*` call of every
session (no marker exists yet → block).

The correct decomposition, applied below:

- **Pre-gate on shape and precondition** (what the hook *can* assess before execution).
- **Post-verify on behaviour** (a PostToolUse hook — where "did the safe path actually run, or
  fail and fall through?" is answerable).
- **Make the whole control immutable to the agent** (the parent's real finding, and the only
  part that changes what the control *is*).

---

## What was applied, in layers

Each layer was applied by re-seeding a root-owned Docker named volume
(`claude-code-managed-settings`) mounted at `/etc/claude-code`; the agent cannot write there.
Each was verified live against the re-seeded hook, not a scratch copy — a distinction that
mattered, because scratch-level confidence twice reported "done" on code that then failed or
surfaced new defects when run for real.

### Layer 1 — Immutability (the core fix)

The parent's headline finding was that the guard, its wiring (`settings.local.json`), and its
pin file were all writable by the agent they constrain — *"not a boundary, a lint rule."*

Fixed via Claude Code's **managed-settings layer**: `/etc/claude-code/managed-settings.json`,
highest precedence, with `"allowManagedHooksOnly": true` — which makes the runtime ignore
hooks wired anywhere else entirely. The hook scripts, the pin, and the wiring moved to
root-owned `/etc/claude-code` (mode `640`, dir `750`, owner `root:node`). `settings.local.json`
— which the agent legitimately writes (its auto-growing `permissions.allow` list) — was left
untouched; only the hook *wiring* moved.

**Verified by first breaking out of the old control, then failing to break the new one:**

| Attack (as the agent, uid 1000) | Old hook (before) | Managed hook (after) |
|---|---|---|
| Append to guard script | succeeded | Permission denied |
| `chmod` the guard | succeeded | Operation not permitted |
| Rename / delete the guard | succeeded | Permission denied |
| Replace the pin file | — | Permission denied |
| Edit `managed-settings.json` | — | Permission denied |
| Shadow-hook via `settings.local.json` | — | injected, **did not fire** (old wiring inert) |

The "before" column is the parent report's vulnerability, demonstrated rather than asserted:
three attacks that all succeeded in the morning return `Permission denied` in the afternoon,
against a real target, with `Permission denied` distinguished from `No such file` (the latter
nearly produced a false "fix held" verdict across two failed deployment attempts — see
Deployment notes).

**Operational cost, accepted deliberately:** the operator is now the only party who can edit
the guard. Future hook changes require a root-side volume re-seed. This is the point, not a
regression.

### Layer 2 — PostToolUse extraction (the field bug)

With immutability in place, the PostToolUse hook — intended to verify behaviour and scan output
for leaked secret-shapes — was found to read a field that **does not exist in the payload**.

The extraction was `jq -r '.tool_output // empty'`. There is no `tool_output` key. Established
off the wire (not from docs — docs are what produced the wrong field) via an instrumented
read-only probe:

- Bash output lives at `.tool_response.stdout` (and `.tool_response.stderr`).
- TaskOutput lives at `.tool_response.task.output` — nested one level deeper, a **different**
  field.

Because extraction was always empty, the post-hook was **broken in both directions
simultaneously**:

- Every legitimate `cred_*` call **false-alarmed** "ROTATE" (empty output never contains the
  success marker).
- Every real leak passed **silently** (empty output never matches a leak-shape).

This is worse than an absent control: it trains the operator to dismiss "ROTATE" as noise while
missing the exact leaks it exists to catch. Fixed with tool-conditional extraction (Bash →
`stdout`+`stderr`; TaskOutput → `task.output`; unknown tool → recursive string walk, so a
future output-bearing tool is scanned by default rather than silently skipped). All four cases
flipped and were verified live.

### Layer 3 — Audit-log redaction (#2) and structural detection (#3)

Turning the post-hook on surfaced two defects that had been dormant while it read nothing:

**#2 — the leak detector was itself a leak sink.** On a ROTATE, the offending output was
written to the audit log *verbatim* — so every real trip wrote the leaked value to disk. Fixed:
all excerpts pass through redaction (`scheme://user:REDACTED@host` shape) *before* any write.
Redact-then-truncate, not truncate-then-redact — the latter (which the pre-hook uses) can cut a
match mid-secret and leave it unmasked; noted as a pre-hook improvement, out of scope here.

**#3 — the detector self-triggered.** The connection-string check matched on *length* (8+ chars
after a delimiter), and the redaction placeholder is 8 characters — so tailing the audit log
re-tripped the scanner against the hook's own output, an infinite false-ROTATE loop. Fixed by
matching on *structure* (`scheme://…@host:port` shape) rather than length, plus an explicit
placeholder exclusion. Regression guarded: the empty-username form `redis://:pass@host` (the
shape that actually leaked in a prior session) still fires.

These two shipped **together, atomically** — #2's fix writes a `:REDACTED@host` line on every
trip, which is exactly the shape #3's old detector re-tripped on; deploying #2 alone would have
amplified the loop.

### Layer 4 — Shell-aware matching (Phase 3)

The pre-hook still matched `cred_*` and paths as **substrings against command text**, with two
consequences of one root cause (regex is not shell-aware):

- **False negative:** `cat config.yaml # cred_field` — the decoy comment satisfies the evidence
  regex, so the raw read is allowed.
- **False positive:** prose mentioning a credential path (an `echo` string, a comment in a file
  being written) is blocked as though it were a raw read. This fired repeatedly during the
  remediation itself — the guard blocked legitimate edits to its *own* source because the source
  contains the path patterns as string literals.

Fixed by replacing regex evidence-detection with a real tokenizer (`bash-parser`, Node — `bashlex`
was unavailable, no python3 in the sandbox). The tokenizer strips comments, normalizes
quote-splitting (`c""at` → `cat`), and treats `cred_*` as evidence only when it is a segment's
**command word**, not a substring anywhere. A raw read in *any* chain segment denies regardless
of the others. Both directions closed; the fallback-shape gate bug from Layer 2 (the hook's own
documented `cred_field x y || cat x` example did not actually deny) was fixed in the same pass.

### Layer 5 — Phase 3 cleanup (three defects surfaced by Phase 3 going live)

Running the tokenizer for real surfaced three more:

- **Defect A (priority) — the tokenizer failed *open* and *silent*.** If `node`/`bash-parser`
  became unresolvable, the hook reverted to the old regex — re-opening the bypasses just
  closed — with no log line. This is the day's entire thesis reappearing inside the fix itself.
  Fixed: on tokenizer unavailability the hook now **fails closed** (fallback regex may deny but
  never grant an allow) and writes a `tokenizer-fallback-engaged` audit line on every
  invocation. Verified by forcibly breaking the tokenizer in a scratch copy and confirming both
  the deny and the log line.
- **Defect B — the prose false-positive relocated to the post-hook.** Phase 3 only tokenized the
  *pre*-hook; the post-hook still substring-matched, so prose it should ignore fired a real
  ROTATE. Fixed: post-hook Check 1 now uses the same structural assessment.
- **Defect C — grep's search pattern was treated as a file target.** `grep -n "config.yaml" file`
  denied on the *pattern*, not a real path — friction on an everyday command, the kind that
  trains an operator to route around the guard. Fixed at the tokenizer by modelling grep's own
  argv grammar (first positional is the pattern unless `-e`/`-f`); only genuine file arguments
  are path-checked. Disclosed limitation: long-form flags with a separate value word
  (`--after-context 3`) aren't modelled; the attached form (`=3`) is.

---

## Verification discipline (and where it nearly failed)

Two verification near-misses are worth recording, because both are instances of the parent's
own failure class occurring *during its remediation*:

1. **Deployment reached the wrong target twice.** The immutability fix was first applied to the
   host's `/etc/`, not the container's (the container's `/etc` is an ephemeral overlay layer, not
   a bind mount). Then it was applied to a named volume that the *running* container had never
   mounted (Docker binds volumes at container start; the process was never restarted). Both times
   every file "existed" somewhere and the change "succeeded" — and both times the agent session
   caught it by checking `/proc/self/mountinfo` and distinguishing `No such file` from
   `Permission denied`, rather than trusting that a successful copy meant a live control. A
   conflated `No such file`/`Permission denied` would have reported "fix held" against
   infrastructure that wasn't there.

2. **Scratch-level confidence twice preceded live defects.** Both the initial Phase 3 rollout and
   this cleanup reported "verified" against scratch harnesses, then surfaced fresh defects the
   moment they ran against the re-seeded hooks (the post-hook prose ROTATE and the grep
   conflation were both found only live). The rule adopted: *"verified in scratch" is not
   "verified"; the gate is the live re-seeded hook.*

Provenance was checked on every re-seed by SHA-256 matching the patch's base against the
currently-deployed file — catching, among other things, a bundle that shipped a `node_modules`
symlink which a naïve recursive copy would have dereferenced back over the live dependency tree.

---

## Residual Risk (disclosed, not closed)

The immutability boundary (Layer 1) is real. The matching logic on top of it is **a lint rule
against a cooperating agent, not a boundary against an adversarial one.** `SYSOPS.md` now states
this plainly. Specifically open:

- **Surviving pre-hook bypasses**, disclosed rather than chased with more regex (chasing
  individual shapes is how the original defect was built): `command cat <path>` (builtin
  unwrap), `… | xargs cat` (path arrives via piped stdin), `find -exec`, embedded sub-language
  string concatenation (`awk 'BEGIN{…system()}'`), variable reassembly (`$a$b`), `eval` / `sh -c`
  re-entrancy, and aliasing. Multi-hop across *separate* tool calls is out of scope pending a
  session-state mechanism.
- **Fail-closed cost:** during a tokenizer outage, a *genuine* `cred_*` call is denied because it
  is indistinguishable from a decoy without the parser. Accepted — failing toward blocking is
  correct for a credential control.
- **Provenance gap:** the locked hooks were seeded from an agent-writable directory. The
  content was checksum-matched at seal time, but an agent-writable source has unverifiable
  provenance by definition. This does not fully close until the canonical hook source lives in
  the git repo and the container build pulls from there rather than from a live-edited seed.
- **`credential_safety.sh` symlink** (the parent's incidental trigger) was created; the library
  is now genuinely present and the guard's precondition check depends on it.
- **Pattern duplication:** `CRED_PATH_RE` / `SCRIPT_PATH_RE` / `LIB_EVIDENCE_RE` and the
  redaction logic are duplicated across both hook files — should consolidate into a shared
  `hooks/lib/credential_patterns.sh`. Tech debt, not a vulnerability.

---

## Lessons Learned (extending the parent's)

The parent's thesis was *a control that reports healthy is not a control that works*, evidenced
by four independent instances. The remediation added a fifth, sharper form:

**Every layer of this fix, when made to actually run, revealed the next layer's silent failure.**
Immutability held — and exposed that the post-hook read a nonexistent field. Fixing extraction
turned the post-hook on — and exposed that it logged secrets verbatim and self-triggered.
Tokenizing the pre-hook closed the prose bug — and revealed the same bug had relocated to the
post-hook, and that the tokenizer itself failed open and silent. The defects were not
introduced by the fixes; they were *pre-existing and masked*, each by the inert layer above it.

The operational rules this produced:

- **A control must not degrade to a weaker control without a trace.** The tokenizer's silent
  regression to regex (Defect A) was the purest re-instance of the whole failure class, caught
  only by reading whether a computed `TOKENIZER_PARSE_OK` was ever *used*. It wasn't. Now it is,
  and the degradation is logged.
- **"Verified" means verified against the live artifact.** Scratch confidence, host-vs-container
  target confusion, and volume-not-mounted-yet each produced a green result against something
  that wasn't the running control. The gate is the re-seeded hook, tested live, with
  `Permission denied` distinguished from `No such file`.
- **Disclose surviving bypasses; do not chase them with patterns.** The original guard was built
  by accreting regex patterns one clever shape at a time — which is precisely why it was
  theatre. The tokenizer is a large improvement and *still not a boundary*; every remaining
  bypass is written down in `SYSOPS.md`, not papered over.
- **False positives are a slow failure mode.** The grep conflation and the prose ROTATE don't
  leak anything — they train the operator to resent and route around the guard, which ends in
  the same place as no guard at all. Fixing them is security work, not polish.

---

*Environment: Edith (Claude Code sandbox, Debian 12, uid 1000, no docker socket, zero caps) ·
managed-settings immutability layer · `claude-code-managed-settings` volume · KONYKS-SERVER ·
homelab-incident-reports*
