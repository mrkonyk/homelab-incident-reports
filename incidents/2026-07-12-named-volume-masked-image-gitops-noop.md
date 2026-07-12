# Named Volume Masked the Image: Two Months of No-Op GitOps Deploys

**Date:** 2026-07-12
**Severity:** P2 Medium
**Status:** Resolved
**Affected:** Hermes-Agent (container runtime code), Komodo GitOps deploy path
**Duration:** 2026-05-28 → 2026-07-12 (45 days of frozen runtime code; at least one "successful" image pull in that window changed nothing)

---

## Summary

Hermes-Agent's runtime code lives at `/opt/hermes` inside the container. A Docker named volume,
`hermes_shared_volume`, was mounted over that path on 2026-05-28. Docker does not reseed an
already-populated named volume when a container is recreated from a newer image — the volume's contents
win, silently.

The result: every subsequent deploy of Hermes-Agent — including a deliberate fresh image pull on
2026-07-05 — recreated the container successfully, reported success, and continued executing the
May 28 code. The image changed underneath it and had no effect. The standard homelab pattern (bump the
tag, let Komodo redeploy) was a guaranteed no-op for this stack, and would have remained one
indefinitely.

The condition was detected not by any monitoring signal but by an inconsistency: after the July 5 pull,
`hermes --version` still reported the container as significantly behind upstream — a number that should
not have been possible if a fresh image had actually taken effect. Resolution was to remove the volume
mount entirely, allowing `/opt/hermes` to be plain image content, and to pin the image by digest so the
running code is reproducible.

---

## Timeline

| Date | Event |
|------|-------|
| 2026-05-28 | `hermes_shared_volume` created and populated with the then-current build. Mounted at `/opt/hermes`. |
| 2026-05-28 → 2026-07-05 | Multiple container recreations and deploys. All succeed. None change the running code. |
| 2026-07-05 | Deliberate fresh image pull performed to pick up an upstream fix. Reported success. Running code unchanged. |
| 2026-07-05 | Inconsistency noticed: `hermes --version` still reports a large commit gap after a fresh pull. Flagged as "investigate later," deliberately not chased mid-session. |
| 2026-07-10 | Root cause identified during a scoped assessment: the named volume masks the image. A second finding — two separate source checkouts inside the container, only one of which is executed — complicates the upgrade path. Go/no-go deferred pending one unanswered question. |
| 2026-07-12 | Read-only investigation resolves the remaining question with source evidence. Volume inventory confirms it holds only build artifacts, no state. |
| 2026-07-12 | Remediation applied. Verified. |

---

## Root Cause

**The masking mechanism.** When a Docker named volume is mounted over a directory that the image also
populates, Docker seeds the volume from the image *only if the volume is empty*. Once populated, the
volume takes precedence permanently. Recreating the container from a newer image does not re-seed it,
does not warn, and does not fail. The deploy is genuinely successful — it just doesn't do what the
operator believes it does.

`hermes_shared_volume` was populated on 2026-05-28 and mounted at `/opt/hermes`, the container's runtime
code path. From that moment, `/opt/hermes` was frozen. Image tag changes, digest changes, and full
`--force-recreate` cycles all had zero effect on executed code.

**Why it was invisible.** Every observable signal said the deploy worked:

- The container recreated cleanly and reported healthy.
- The image pull genuinely fetched a new image.
- Komodo reported a successful deploy.
- Container uptime, restart count, and health checks were all normal.

The *only* signal that something was wrong was a version string that didn't move. Nothing was configured
to look at it.

**Two source checkouts (compounding factor).** Investigation also found two copies of the source inside
the container: the runtime path (`/opt/hermes`, no `.git`, the volume's contents) and a separate checkout
under the appdata mount that *did* have a `.git` and sat far ahead of the running code. This created a
credible trap: the project's built-in `update` command is a git-pull-and-reinstall, which — if it
targeted the second checkout — would report a successful update while the container continued booting
from the first.

Source review resolved this: the update command detects its install method, finds a `docker` stamp, and
refuses to run a git-based update inside a container at all (the published image excludes `.git` by
design). It touches neither checkout. The second checkout is an orphan that nothing in the running system
consumes. The trap was real but not sprung — and confirming that took reading the code path rather than
trusting the command's behaviour.

---

## Remediation

**Preconditions established before touching anything:**

- Full inventory of `hermes_shared_volume`: 100% source and build artifacts (`.venv`, `node_modules`,
  package metadata, module directories, build SHA marker). **No state.** All service state — config,
  sessions, memories, skills, personality files, databases — lives in a separate appdata bind mount,
  untouched by any of this. Drop-and-reseed therefore loses nothing.
- Recognised that the cached image had **never executed its own `/opt/hermes` code** — the volume had
  masked it for the image's entire life. It was therefore *not* a known-good fallback. The frozen volume
  was the only known-good artifact in existence.

**Sequence:**

1. **Snapshot first, as a hard gate.** Tar the volume contents to the backup share; verify the archive
   lists real build artifacts. Separately, clone the volume under a preserved name rather than deleting
   it. Record the pre-upgrade version and build SHA as the comparison baseline.
2. Check the registry for published version tags rather than accepting a floating tag. Real release tags
   existed — a rolling tag pushed minutes earlier was available and was **rejected** in favour of the
   most recent deliberate release cut.
3. Stop the container.
4. Pull the chosen release tag; capture its digest.
5. Edit compose: **remove** the `hermes_shared_volume:/opt/hermes` mount; **pin** the image as
   `repo:tag@sha256:...` (matching the existing digest-pinning convention used across the other stacks);
   remove the now-orphaned top-level volume declaration. Leave the appdata bind mount alone.
6. Recreate the container. `/opt/hermes` is image content for the first time in 45 days.

**Verification — nine explicit checks, each pass/fail:** build SHA changed against baseline; `/opt/hermes`
confirmed as fresh image content; config schema migrated cleanly (with automatic backups); all cron jobs
present **and their toolset references resolved live** (presence alone was explicitly rejected as
evidence — see Lessons Learned); all MCP servers connected and live-tested; model routing and fallback
chain intact; disabled toolsets still disabled; tool-count pruning not reverted; memory backend connected.

All nine passed. Rollback was not needed. All four rollback artifacts (tar, cloned volume, orphaned
original volume, pre-upgrade compose backup) were deliberately **retained**, not cleaned up, pending
several days of real cron cycles.

---

## Prevention

- **Volume mount removed.** `/opt/hermes` is now plain image content. GitOps deploys of this stack now do
  what they claim to.
- **Image pinned by digest.** The stack was on a floating tag — the one exception among the digest-pinned
  images in this environment, and precisely the case where floating tags cause the most damage, since a
  no-op deploy of a moving tag is unfalsifiable. Now reproducible.
- **Audit for the same shape elsewhere.** The general defect is *any* named volume mounted over a path the
  image populates. Every stack should be checked for this pattern — not just this one.
- **The version string needs to become a monitored signal.** It was the only indicator that anything was
  wrong, and it was only read by a human, by chance, once. A post-deploy assertion that the running build
  identifier actually *changed* would have caught this on 2026-05-28 rather than 45 days later.

---

## Lessons Learned

**A successful deploy is not a changed deploy.** Every layer of the deploy pipeline reported success and
every layer was telling the truth — the pull happened, the container recreated, the health check passed.
The pipeline verified that *steps executed*, never that the *outcome changed*. Any deploy system that
cannot distinguish "new code is running" from "the deploy command exited zero" will eventually ship this
bug, and will do so silently and indefinitely.

**Detection came from an inconsistency, not an alarm.** Nothing in the stack was watching for this,
because nothing was designed to consider it possible. What surfaced it was a number that shouldn't have
existed — a version gap that survived a fresh pull. The operational habit worth generalising is treating
*impossible-looking readings as findings rather than noise*, because in a system where every alarm is
silent, the anomaly is the only signal left.

**The rollback artifact was the frozen bug.** The most counter-intuitive finding: the cached "newer" image
had never once executed its own code, so it was not a fallback — the only known-good state in existence
was the stale volume that *was* the defect. Removing the defect and preserving it as the rollback path
are the same action. Any remediation that had begun by deleting the volume (the intuitive first move)
would have left no way back at all.

---

*Environment: KONYKS-SERVER (Unraid) · Hermes-Agent · Komodo GitOps · homelab-incident-reports*
