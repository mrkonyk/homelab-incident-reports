# Six Controls That Reported Healthy: Instance-Scoped Verification Across a Container Stack

**Date:** 2026-07-23 — 2026-07-24
**Severity:** P1 High
**Status:** Resolved
**Affected:** Hermes-Agent, all 25 Komodo-managed compose stacks, credential rotation tooling, agent session transcripts
**Duration:** Code staleness persisted 68 days (2026-05-16 → 2026-07-23), across two prior incident reports. Credential embedding persisted ~6 weeks past its supposed remediation. Stack git authentication outage: ~1 hour, self-inflicted during remediation.

**Related:** [2026-07-10 — Hermes named-volume staleness](2026-07-10-hermes-named-volume-staleness.md) · [2026-07-12 — Named volume masked image, GitOps no-op](2026-07-12-named-volume-masked-image-gitops-noop.md) · [2026-07-12 — Credential guard textual enforcement](2026-07-12-credential-guard-textual-enforcement.md)

---

## Summary

On 2026-07-12 this homelab filed an incident report identifying a pattern across four separate findings: controls that reported healthy while doing nothing, caught by inconsistencies rather than by alarms. That same report closed a second incident as Resolved, stating that a masking Docker volume had been removed and the image digest-pinned. Neither had happened. The Resolved status held for eleven days and the underlying fault persisted the entire time.

A routine version bump of the Hermes-Agent container revealed the container had not been upgraded in 68 days. An external Docker volume mounted over the image's application directory was masking the image code, so every image pull since mid-May had updated metadata without changing a line of running code — despite two incident reports already filed against this exact volume, the second of which declared it fixed. Investigating the deployment path surfaced a live GitHub fine-grained access token embedded in a stack clone's git remote URL and printed into an agent session transcript. Tracing that to its class showed the same shape again: a June remediation for credential embedding had been applied to two components and never propagated to the other twenty-five, while the rotation script itself re-embedded the token on every run. Widening the search found a fourth: a 2026-07-03 transcript cleanup, reported complete, had enumerated by known credential values rather than credential shapes, leaving nine unique tokens across twelve files uncatalogued. All were remediated. The controls examined shared one property — each verified the instance it already knew about rather than the class it was responsible for — and one of them was the incident-closure process itself.

---

## Timeline

| Date | Event |
|------|-------|
| 2026-05-16 | Last image pull that actually reaches running code. Application freezes at this version. |
| 2026-07-10 | First incident report filed on named-volume staleness for this container. |
| 2026-07-12 | Second report filed on the same volume and closed as **Resolved**, stating the mount was removed and the image digest-pinned. Neither held: the mount remained and the tag was still floating. Same session files a P1 on the credential guard enforcing command text rather than behaviour, designs a three-part remediation, and applies it the same day. |
| 2026-07-13 | The guard's post-execution hook is found to have been silently reading nothing since deployment. Extraction fixed; behavioural verification becomes functional. |
| 2026-07-14 | Newer image pulled and digest-pinned. Shadowed by the volume; never executes. |
| 2026-07-23 ~16:00 | Pre-upgrade assessment. `hermes --version` inside the container reports the May build and "update available: 5463 commits behind". |
| 2026-07-23 ~17:00 | Volume shadow confirmed as the mechanism. Deployment path verified as GitOps-managed. |
| 2026-07-23 ~19:00 | Backups taken and restore-verified. Volume diffed against image. Pin bumped and mount removed via GitOps commit. Volume deleted. |
| 2026-07-23 ~19:54 | Upgrade verified at the code layer. Scheduled job executed. All five MCP servers connected. |
| 2026-07-23 ~20:30 | `git remote -v` on a stack clone prints a live access token into the session transcript. |
| 2026-07-23 ~21:30 | Class investigation: all 25 stack clones carry the token in-URL. June's remediation reached only the native components. |
| 2026-07-24 ~00:00 | Token revoked ahead of propagation. All 25 stacks lose git authentication. |
| 2026-07-24 ~00:30 | Strategy pivot: embedding established as unavoidable at the orchestration layer. Hermes-Agent recovered first as a single-stack proof deploy, confirming the orchestrator re-embeds the new token on deploy. |
| 2026-07-24 ~01:00 | Remaining 24 stacks restored via the sanctioned rotation script. Revocation doubles as an unplanned scope test — nothing outside the known touchpoints was affected. |
| 2026-07-24 ~02:00 | Shape-based transcript sweep finds 9 unique tokens across 12 files. All confirmed dead. |
| 2026-07-24 ~03:30 | Redaction and detection controls built, scoped, and labelled. Inactive transcripts scrubbed. |

---

## Root Cause

### 1. Volume shadow (the code staleness)

The compose definition mounted an external named volume over the image's application directory:

```yaml
services:
  hermes-agent:
    volumes:
      - app_shared_volume:/opt/hermes

volumes:
  app_shared_volume:
    external: true
```

A volume mounted over a path in the image masks whatever the image ships at that path. The volume was populated in May and persisted across every subsequent recreate. It contained no `.git` directory, so the application's own self-update mechanism — which performs a git pull — could not function either. The result was a container whose image label, digest pin, and GitOps repository all agreed on a version that the running process was not.

Two remediations were filed against this volume before it was fixed. The first recorded the staleness. The second asserted that the mount had been removed, that the application directory was now plain image content, and that the image was digest-pinned — and closed the incident as Resolved. The mount was still present eleven days later, and the digest pin did not land until two days after that report, whereupon it was shadowed too. Had the mount genuinely been removed, the subsequent recreate would have run image code; it did not. The closure was a claim, and its acceptance ended the investigation.

### 2. Credential embedding (the leak)

The container-orchestration agent authenticates git operations for managed stacks by writing the access token directly into each clone's remote URL. A June remediation moved the *repository* and *agent-configuration* components onto the platform's native git-provider mechanism, which stores credentials outside the URL. The 25 stack clones were never migrated. Additionally, the credential rotation script — the control that exists to bound credential lifetime — re-embeds the current token into all 25 clone configurations as a designed step of every rotation.

The rotation procedure was therefore the propagation mechanism for the exposure it was intended to bound.

### 3. Incomplete transcript cleanup (the residue)

A cleanup performed on 2026-07-03 enumerated affected files by grepping for the specific credential values known to have leaked, catalogued only the files that matched, and deferred the remainder as out of scope. Files containing *other* credentials of the same shape were structurally invisible to that method, and the deferred deletions never ran. A shape-based sweep — matching token formats rather than token values — found nine unique tokens across twelve files, including one with 108 occurrences and another holding classic-format tokens predating the fine-grained migration.

### 4. The guard that scans intent

Documented separately on 2026-07-12 and remediated the same day: the credential guard enforces command *text*. It matches the shape of a safe invocation rather than confirming a safe invocation occurred. The remediation added a fail-closed precondition on the safety library, denial of safe-call-chained-to-raw-read shapes, and a post-execution hook performing behavioural verification. The pre-execution parts were enforcing immediately and fired repeatedly against the work described here. The post-execution hook was not: it read nothing for its first day of deployment, verifying behaviour it could not observe, and was corrected on 07-13.

They did not catch this leak, and could not have. A pre-execution guard inspects the command about to run; the credential in this incident appeared in a command's *output*. No amount of input-side hardening reaches that. The guard's remaining scope limitation — that invoking a script by path presents no credential reference to match — is real but secondary; the architectural boundary is input versus output, and it was never a defect in the guard's implementation.

---

## Remediation

**Change management applied throughout:** backups before changes, restore verification before migration, staged single-target rollout before fleet-wide application, and an explicit rollback anchor at each step.

### Hermes-Agent upgrade

1. Backed up the application appdata tree, snapshotted the shadowing volume, and recorded the current image digest as a rollback anchor.
2. **Verified the backup by restoring it**, not by creating it. The state database was extracted to a clean path, opened, and checked: integrity check passed, schema version read, 12,651 messages and 325 sessions readable. Application state migrations are forward-only with no automatic downgrade, so this backup was the only rollback path available and was proven before it was needed.
3. Diffed the volume contents against the target image to confirm it held pure upstream code and no runtime state. Runtime state lived in a separate bind mount and was unaffected.
4. Applied via GitOps: a single commit bumped the digest pin, removed the volume mount, and removed the external volume declaration. Push triggered a webhook redeploy. No container was edited directly.
5. Deleted the volume once the recreated container no longer referenced it. Because the volume was declared `external: true`, removing the mount from compose would have left it on the host, re-mountable by any future compose edit — leaving the failure mode reachable.

### Credential remediation

6. Established that token embedding is performed by the orchestration agent itself and is not avoidable without changing that component. The strategy was revised from *eliminating the embed* to *bounding its exposure*. The rotation script's embedding step was reclassified as the correct mechanism rather than a defect, and deliberately left in place.
7. Restored the stacks that lost authentication. Hermes-Agent was recovered first as a single-stack proof, confirming the orchestrator re-embeds the current token on deploy. The remaining 24 were restored by running the sanctioned rotation script, which re-embedded the new token across all clones; its redeploy step found nothing stale to redeploy, so the re-embed alone constituted recovery. All 25 verified by name with the git credential helper disabled, so a cached credential could not mask a bad embed.
8. Swept all transcripts by token shape. Confirmed all nine recovered tokens dead by authenticating each rather than inferring from dates. Scrubbed the eleven inactive files; verified token count zero, line counts unchanged, JSON structure intact.

### What did not work, and why

- **Output-side redaction as a post-execution hook.** The obvious control — scan tool output for credential shapes and redact before the transcript records it — is not buildable at that layer. A post-execution hook runs after output has been persisted. It can detect, warn, and halt; it cannot unwrite. Building it as specified would have produced a control named "redaction" that performed only detection. This layer was already correctly scoped in July, when the post-execution hook was built for *behavioural verification* rather than interception; the error here was in the specification, which asked an audit layer to do prevention. It was caught by reading the existing hook implementation rather than trusting the interface name.
- **A universal pre-execution command rewriter** would cover every local command by routing output through a redactor at source. Rejected: it introduces quoting, pipefail, and binary-output fragility into the path every command traverses, in exchange for covering a vector where no leak had occurred.
- **Shortening token lifetime** is the natural mitigation for an unavoidable embed. It is blocked behind automating rotation, which is currently manual. Recorded as a dependency rather than built.

---

## Prevention

| Control | What it actually does | Scope | State |
|---|---|---|---|
| Redaction in the remote-execution helper | Redacts token formats and `scheme://user:token@host` URLs from output before it reaches the transcript | Server-side commands through this helper **only**. Blind to file reads, MCP tool results, and web fetches. | Live |
| Post-execution guard extension | Detects bare-token shapes the existing check misses and halts | **Detection, not prevention.** Cannot unwrite a persisted value. | Staged — pending deployment |
| Session-end transcript scrub | Scrubs a session's transcript on close — the only point at which the file is not being appended to | Per-session, at close | Live, pending first real-close verification |
| Volume-shadow drift check | Compares running code version, image label version, and repository pin; alerts on disagreement between any two. Generalises to flagging any named volume mounted over an image's install directory | Fleet-wide, read-only, daily | Proposed |

Each control is labelled in code with what it covers and what it is blind to. A narrow control that is understood is more useful than a broad one that is misjudged, and the labelling is what makes the difference.

**Open follow-up items:**

- Re-examine the incident-closure process. A report that asserts a remediation and marks itself Resolved currently requires no evidence that the asserted change took effect.
- Automate credential rotation, then shorten token lifetime.
- Convert the agent's anonymous 25 GB home volume — holding real state including SSH material and custom client code — to a named or bind volume. Same shadow class as the volume removed here, currently vulnerable to a recreate with anonymous volume renewal.
- Verify the session-end scrub actually fires on a real session close rather than assuming it did.
- Investigate whether a newer orchestration-agent release supports a credential-helper mechanism for stack deploys.

---

## Lessons Learned

**1. Closing an incident is itself a control, and it can report healthy while doing nothing.** The 2026-07-12 report asserted that the masking volume had been removed and the image digest-pinned, and closed as Resolved. Neither change had been made. Every downstream reader — human and automated — then treated the fault as fixed, and the shadow persisted eleven further days across a second image pull. A closure is a claim about the world, and this one was accepted on the strength of the report rather than the state of the system. It was also the most expensive failure documented here, because unlike a control that fails open, a false Resolved actively suppresses the next investigation. What changed: closure now requires evidence that the asserted change took effect, verified at the layer the change was supposed to alter — for this incident, the running version reported from inside the container and a mount inspection, not the compose file that was edited.

**2. A verification scoped to known instances cannot discover unknown ones, and reports success precisely because it found everything it was capable of looking for.** The digest pin verified the pin, not the running code. The June credential fix was applied to the two components in front of the engineer, not the twenty-five behind them. The 2026-07-03 cleanup enumerated by known credential values rather than credential shapes, making every unknown instance invisible by construction. Three independent controls, three subsystems, one identical error. Remediation must be defined over the property, not over the examples. What changed: the drift check compares three independent sources of version truth rather than confirming one, and the transcript sweep now matches credential *shapes* rather than credential *values*.

**3. A control's name describes its intent; only its mechanism describes its coverage.** The stack reported the correct version at every layer an operator would normally inspect — image label, digest pin, GitOps repository — while the process on disk was 68 days stale. Every signal was accurate about itself and collectively wrong about the system. The credential guard is a more instructive case: hardened, live, and enforcing throughout this incident, it still could not see this leak, because it inspects commands before they run and the credential appeared in output afterwards. That is not a defect to be patched — it is the boundary of what a pre-execution control can observe, and it was invisible for as long as the control was described by its purpose rather than its mechanism. The same reasoning caught a seventh control before it shipped: a proposed output-side redaction hook, specified in this incident, turned out to be architecturally incapable of redaction at the layer proposed. What changed: verification reads the layer beneath the one that reports, and each control now carries its blind spots documented in the same file as its implementation.

---

*Environment: KONYKS-SERVER (Unraid) · Komodo GitOps · Hermes-Agent · agent tooling hooks · homelab-incident-reports*
