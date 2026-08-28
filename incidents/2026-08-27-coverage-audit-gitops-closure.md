# Container Management Coverage Audit and GitOps Closure

**Date:** 2026-08-27 → 2026-08-28
**Severity:** P2 Medium
**Status:** Resolved
**Affected:** Whole-host container inventory; three stacks outside GitOps
**Duration:** Two stacks had been outside version control since deployment; one had been
undeployable from its own repository for an unknown period

---

## Summary

An audit was run to answer a question that had never been checked directly: how many containers
on the host are actually under GitOps management, and what manages the rest. The answer was not
in any existing record. Fifty running containers were classified by reading each container's
`com.docker.compose.project.config_files` label — the compose file used at deploy time — and
comparing it against what the declarative resource file claimed.

Three stacks disagreed with the repository. One was declared as GitOps-managed but was actually
running from an Unraid template, meaning the orchestrator believed it owned a stack it was not
running. Two were orchestrator-managed but had never been committed to the repository at all.
All three were brought into version control and redeployed from it. Nine reference figures
carried in prior documentation were checked; five had drifted.

The audit also produced four withdrawn claims, three of which had been load-bearing for
planning. That thread is documented below because it was the more valuable outcome.

---

## Root Cause

**Why the gap existed.** A full GitOps migration had been completed in June. Three stacks were
excluded for reasons that were correct at the time — one was managed by a host template, two
kept their compose files on the host rather than in a repository clone. None of the three was
ever revisited, and nothing in the monitoring stack could detect the divergence: existing checks
covered hook drift and deploy lag, but nothing compared the orchestrator's view of a stack
against the repository's.

**Why prior audits missed it.** An earlier audit walked the stacks directory and tested each
entry for a `.git` subdirectory. That guard silently skipped exactly the stacks that had no
clone — the ones the audit existed to find. The correct method is the inverse: enumerate running
containers and read what each was actually deployed from. Container labels are ground truth;
directory structure is not.

**A second-order trap.** A path-prefix guard would have failed the same way. Three directories
sit under the stacks root without being clones, including one that is a build artifact rather
than a stack. The classification must test each directory individually.

---

## Remediation

Classification result across 50 running containers, 0 stopped:

| Management | Count |
|-----------|-------|
| GitOps, repository-backed | 39 |
| Orchestrator-managed, compose on host | 8 |
| Host template (dockerMan) | 3 |
| No management labels at all | 0 |

The three divergent stacks were remediated in sequence, lowest risk first:

1. **Configuration files committed.** One stack's compose bound two paths relatively —
   a configuration file and an entrypoint directory — neither of which existed in the
   repository. A deploy from a clean clone would have caused the container runtime to
   auto-create both as empty directories, one of them where a file was required. A
   non-printing scan of all run-directory files for credential-shaped content was run
   before committing anything; it came back clean, and the architecture turned out to be
   correct already — the configuration referenced a relative path that the image
   materialises from an environment variable, so no secret had ever been in a
   bind-mounted file.

2. **Environment files positioned.** The post-clone run directory is a subpath *inside* the
   cloned tree, not the clone root. Both remaining stacks kept their environment file at the
   clone root, one level above where the orchestrator would read it. Left alone, both would
   have started without their environment and failed — one loudly, one silently.

3. **Redeployed and verified against the label**, then against function: for the agent stack,
   confirmed enrolment and an active connection to its manager, not merely a running container.

Final split: 45 repository-backed, 3 host-compose, 2 host-template. The three remaining
host-compose containers are the orchestrator's own core, database and agent — which cannot
redeploy themselves. That is the terminal state.

---

## The Measurement-Error Thread

Four claims made during this work were later withdrawn. They are recorded because the pattern
matters more than any individual finding.

**Claim: an initial clone into a non-empty directory destroys its contents.** Derived from
strings in the shipped orchestrator binary — an ordered stage list including a "failed to remove
existing repo root before clone" error and a linked recursive-delete call. This was labelled
static evidence at the time, then allowed to drive a hazard warning that shaped three sessions
of planning, including out-of-band backups of encryption-key-bearing environment files. The
first real deploy into such a directory preserved every untracked file. Whatever those code
paths guard, it is not that one. The real hazard was placement, not destruction.

**Claim: version tags are being re-pushed, so tag pinning is not pinning.** A registry sweep
reported that 19 of 19 tag-only images had moved, including immutable-looking version tags. A
whole-population positive is a bug, not a finding. The cause: `docker inspect` reports the
per-platform manifest digest, while the registry serves a multi-architecture manifest list
digest. The two never match. Compared correctly, 18 of 19 were unchanged.

**Claim: a floating tag had moved to a newer major version within the same day**, creating
urgency around a deploy. Same measurement error. It had not moved.

**Claim: a stack deploy had failed verification.** The check polled on a clone directory
appearing, and read the state before the container had been recreated. A clone appearing means
the deploy started; a container creation timestamp means it finished.

One genuine float survived the corrected sweep: the orchestrator's own database image, pinned
only to a major version tag that had in fact moved. It has been pinned to the running digest.

---

## Prevention

- All three divergent stacks are repository-backed and verified.
- The one remaining floating image is pinned by digest.
- The drift check described in the companion report was designed off the method that worked
  here, and covers both the stale-orchestrator-object case and the never-redeployed-since-the-
  repository-changed case.
- Registry comparisons now use manifest list digests. `docker inspect` output is not comparable
  to what a registry serves for a tag.
- Retired containers and their host templates are now removed in the same session as a cutover.
  Deferring them left a stopped predecessor visible in the host UI while the live
  orchestrator-managed container was not — inverting what the interface showed and producing a
  reported outage for a service that was running normally.

Open follow-ups: scheduled archives of the orchestrator directory contain seven environment
files, five secret files and six private keys, with no retention logic and no exclusion
mechanism; the drift check is unbuilt.

---

## Lessons Learned

**An audit's guard clause can be the thing that hides its target.** Testing directories for a
`.git` subdirectory excluded precisely the stacks that lacked one. When an audit exists to find
absent structure, do not key it on that structure being present. Enumerate the running system and
work backwards.

**Static evidence is a hypothesis.** Reading a binary's error strings, or a configuration file,
produces a plausible model of behaviour — not an observation of it. The distinction was noted
honestly at the time and then eroded under planning pressure, where a labelled hypothesis was
treated as a constraint. A cheap live test existed throughout and would have settled it in one
deploy.

**Population statistics catch instrumentation bugs.** Nineteen of nineteen images reporting as
changed was implausible on its face, and that implausibility — not any domain reasoning — is
what exposed a broken comparison method. When a check flags everything, suspect the check.

---

*Environment: KONYKS-SERVER (Unraid) · Komodo GitOps · 50-container inventory · homelab-incident-reports*
