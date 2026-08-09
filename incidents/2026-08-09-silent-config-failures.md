# Silent-Success Failure Modes in Agent Configuration Management

**Date:** 2026-08-09
**Severity:** P2 Medium
**Status:** Resolved
**Affected:** Hermes-Agent, honcho-api, honcho-deriver
**Duration:** Vision subsystem degraded ~13 days (since the 2026-07-27 model change); deprecation exposure standing since 2026-07-22

---

## Summary

A planned configuration audit across two LLM agent services surfaced a broken vision subsystem, a
model tier running on deprecated and partially degraded upstream endpoints, and — more significantly —
seven distinct configuration paths that would accept a change, report success, and produce no effect on
the running system. The audit was triggered by a scheduled deprecation (a model family reaching
end-of-life in October) but the deprecation turned out to be the least interesting finding. Four
proposed changes were withdrawn after inspection of the running code showed they would either do
nothing or cause a hard failure. Three were applied and verified. One was deliberately not applied
after inspection proved it was architecturally unreachable regardless of what the config file said.

The through-line: in every case the configuration layer accepted input the runtime never read.

---

## Timeline

| Time (local) | Event |
|------|-------|
| 2026-08-08 | Agent framework upgraded to a new minor version; post-deploy verification clean |
| 2026-08-08 | Vision capability check observed returning `False`; deferred |
| 2026-08-09 ~11:00 | Read-only audit phase begins — key paths enumerated, no writes |
| ~11:30 | Two of four proposed changes withdrawn: one already tighter than proposed, one unverifiable by the intended method |
| ~12:28 | Pre-change snapshot taken |
| ~12:35 | Three scalar changes applied to the agent framework, verified per-key |
| ~12:50 | Cost-impact review; a proposed cost guard found to be actively harmful |
| ~13:01 | Second service model migration begins |
| ~13:09 | Migration applied; parsed-key diff clean (58/58) |
| ~13:15 | Cross-container introspection verified; new baseline written |
| ~13:55 | Baseline annotated to record an architectural gap explicitly |
| ~14:05 | Introspection script parse-verified against the runtime's own interpreter |

---

## Root Cause

Not one root cause — one recurring *shape*. Seven instances:

**1. Verification command read different keys than the ones being changed.**
The framework's human-readable config summary reports a compression *ratio* and a *protect-last-N*
value. The keys actually being modified were an absolute token threshold and a guaranteed-tail count.
A correct write would have produced zero diff in the summary output, and the planned verification
would have interpreted that as a failed write.

**2. Type coercion silently defeated an integer setting.**
The CLI can write a numeric value as a string. The consuming code performs an `isinstance(value, int)`
check with a silent fallback to the default. A stringified `"8"` reads back as set in the CLI and does
nothing at runtime. Detected only by requesting JSON output and confirming the value returned unquoted.

**3. A proposed cost guard would have broken every affected slot.**
A global "free models only" flag evaluates its check against a *global* default model, before per-slot
model overrides are applied. Enabling it would have failed the check against a non-free default, marked
the entire provider unhealthy, and dropped the client for all five slots using that provider — not the
graceful downgrade the setting name implies. The apparently safer variant (free flag plus a free global
model) is worse: the gate passes, per-slot explicit models still win, and the result is a cost guard
that guards nothing while appearing active.

**4. Free-form dict passthrough accepted a key the runtime never reads.**
Nine call sites route provider preferences through a passthrough dictionary. A tenth — the embedding
path — exposes the same dictionary, so a privacy setting written there parses, stores, and displays
correctly under introspection. But the embedding client constructs its request kwargs without
consulting that dictionary. The setting would have been visible, green, and inert.

**5. Schema validation configured to ignore unknown keys.**
Settings objects are configured with `extra="ignore"`, so any misnamed key is dropped without warning.
A previously-written config stanza had been dead text for an unknown period because the underlying
settings class used different field names than the config file assumed.

**6. In-place edit replaced the file inode and broke a bind mount.**
`sed -i` writes a new file and renames it over the original. For a single file bind-mounted into a
container, this leaves the container holding a stale file handle pointing at the old inode. The
container continues serving the pre-edit content with no error.

**7. Metadata field hardcoded as a literal.**
The introspection script's output carries a `captured` date as a fixed string rather than a generated
value, so every future dump would misreport its own capture date — quietly undermining the field's
use for provenance.

Alongside these, two substantive defects: the vision subsystem was pinned to `auto`, which resolved to
the main chat model, which is text-only — so the capability check failed and vision-dependent tooling
was unavailable. And the second service's model tier was running on a family whose lighter variant had
already passed its studio shutdown date and whose primary variant was scheduled for end-of-life in
October, with the surviving endpoint measured at 36.5% uptime over a 30-minute window.

---

## Remediation

**Applied — agent framework (three scalar changes, one at a time, verified per-key before and after):**

```
auxiliary.vision.provider            auto  ->  openrouter
auxiliary.vision.model               ""    ->  <vision-capable model>
compression.min_tail_user_messages   1     ->  8
```

The vision model was verified against the live provider catalogue for image input support and minimum
context length before being written, rather than taken from documentation. The capability check was
then confirmed through the explicit resolver rather than the top-level boolean, because the resolver
retries with `auto` on failure — a bare `True` would not have distinguished a working pin from a
fallback rescuing a broken one.

**Applied — second service (eight call sites migrated off the deprecated tier):**

Deriver, summariser, both dream paths, and four of five dialectic reasoning levels. The fifth was left
untouched: it is unreachable by auto-selection because the reasoning-level cap defaults below it. One
variable at a time.

**Withdrawn after inspection:**

- A proposal to lower the tool-iteration ceiling. The new upstream default had been assumed to apply;
  the running config carried an explicit value already tighter than the proposed one. The change would
  have *loosened* the tightest gate in the stack.
- The global cost guard, per root cause 3.
- The tenth-site privacy setting, per root cause 4. Writing it would have produced a verification
  baseline asserting a control that was not in force — worse than recording no control at all.

**Verification artifacts:**

- Pre-change snapshots taken for every mutated file, byte-identity confirmed before editing.
- New resolved-config baseline generated: ten introspected sites, provider-preference blocks captured,
  credential fields recorded as environment-variable *names* plus a boolean presence flag — no key
  material. Byte-identical on re-dump.
- The baseline was then annotated to state explicitly that the tenth site's empty parameters represent
  an architectural impossibility rather than pending work.
- The introspection script was parse-checked using the container's own interpreter — the same one that
  will execute it — after the host was confirmed to have no Python installed at all.

---

## Prevention

**Standing procedure changes:**

- Numeric configuration values are confirmed via JSON output with an unquoted-value check, not by
  reading the value back through the CLI.
- Configuration verification uses per-key reads before and after, never a summary command, because
  summary views may report adjacent keys.
- Files bind-mounted individually into containers are edited with in-place writes (`cat >`), never
  with tools that replace the inode.
- Cost-affecting model changes pass a read-only pricing gate against current spend before any write.

**Artifact changes:**

- The verification baseline now records the *absence* of a control explicitly, rather than leaving an
  empty parameter set that reads identically to "not yet configured."
- The hardcoded note string in the introspection script was synchronised to the baseline by extracting
  it from the baseline file rather than retyping it, making drift between the two structurally
  impossible.

**Open follow-ups:**

- The `captured` metadata field remains a fixed literal. Making it dynamic requires pairing with a diff
  convention that excludes the metadata block, since a live date would otherwise differ on every
  comparison by design. Deferred deliberately until after the next scheduled rebuild rather than
  changing the comparison's shape immediately before the comparison runs.
- Embedding traffic carries verbatim message content and remains outside provider-preference coverage.
  Closing it requires a source-level change upstream, not configuration.
- The runtime's retry budget for empty upstream completions is a hardcoded literal and not
  configuration-reachable. The model migration removes the model that produced them; it does not widen
  the budget. Upstream issue to be filed.
- An unexplained container recreation occurred between two known-good points. Writable-layer
  instrumentation applied earlier was found already gone despite only restarts being recorded.

---

## Lessons Learned

**A configuration system that reports acceptance is not reporting effect, and the gap between them is
where silent failures live.** Seven distinct paths in this audit accepted input the runtime never
consumed — through type coercion, ignored schema extras, free-form passthrough dictionaries, and
summary views reading adjacent keys. Each would have produced a system that looked configured and
behaved as though it wasn't. The systemic response is not more care at write time but a verification
discipline that observes the *effect* of a change on running state, through the same code path the
runtime uses. Every check in this audit was moved to that standard.

**Verification artifacts must record absence explicitly, or they become assertions of coverage that
does not exist.** The privacy control that could not reach the embedding path would have appeared in
the baseline as a configured setting. Its omission, left unannotated, would have appeared as an empty
parameter set — indistinguishable from work not yet done, and an open invitation for a future session
to "complete" it and reintroduce the exact inert setting that was deliberately avoided. A monitoring
artifact that cannot distinguish *not applicable* from *not yet applied* will eventually be misread.

**Documentation describes intent; only the running system describes state.** Four proposals in this
audit were derived from release notes and vendor documentation, and all four were wrong against the
deployed system — a default that had been overridden locally, a verification command that read
different keys, a configuration flag whose evaluation order inverted its meaning, and a passthrough
that stopped short of the wire. None were errors in the documentation. All were errors in assuming
documentation and deployment had converged. Every proposed change in the back half of this audit was
gated behind a read-only inspection phase for that reason, and the withdrawal rate — four of eight —
suggests the gate should be permanent rather than situational.

---

*Environment: KONYKS-SERVER (Unraid) · Hermes-Agent · honcho-api / honcho-deriver · homelab-incident-reports*
