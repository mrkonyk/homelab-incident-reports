# Centralized Secrets Migration — Five Independent Verification Failures Caught Before Any Reached Production

**Date:** 2026-06-27 to 2026-06-28
**Severity:** P2 Medium
**Status:** Resolved
**Affected:** SWAG, Authelia, Grafana, centralized secrets manager, GitOps source repository
**Duration:** Two sessions, roughly six hours of active work

---

## Summary

A planned migration of three services from flat-file secrets to a centralized secrets manager turned into a much longer exercise than expected, because nearly every layer of the chain — a command-line tool's own documented example, a web UI's apparent state, an assumption about a credential's character set, a same-value comparison check, and a git push's own reported success — turned out to need independent verification rather than being trustworthy on its face. None of the five issues ever reached a live, broken state in production; each was caught by a deliberate verification step before it could cause real harm. The migration's actual value ended up being less about the secrets themselves and more about how many places a confident-looking signal can be subtly wrong.

---

## Timeline

| Stage | Event |
|---|---|
| Planning | Investigation into the centralized secrets manager's existing setup finds it was never actually finished being configured — no working example to copy from existed, despite prior documentation implying it was in use |
| Setup | A dedicated machine identity is created for the migration scripts; the first authentication attempt is built around the wrong credential type entirely, traced to a UI panel that looked like the right one but belonged to a different authentication method |
| Setup, continued | The correct credential location is found; a second wrong value (a generic internal identifier mistaken for the project identifier) causes a further authentication failure before the right value is identified |
| Secret upload | Nine secret values are pushed to the secrets manager using a file-reference syntax taken directly from the tool's own help text; a later verification pass finds seven of the nine were stored as the literal placeholder text from that example, not the actual file contents — the documented syntax does not do what its own example implies |
| Secret upload, continued | The seven incorrect values are re-pushed using a different, verified-correct method; a byte-for-byte round-trip check confirms correctness this time rather than trusting the fix on faith |
| Script writing | Three fetch-and-restart scripts are written, one per service, each reviewed as literal code rather than as a description of what the code does — a review standard adopted specifically because an earlier description-only review missed a real bug |
| Script writing, continued | The first script's safety guard, written to reject any secret value containing unexpected characters, immediately rejects the real production secret, because the guard assumed an alphanumeric-only format that the actual key does not use |
| Documentation | A scripted documentation edit using a string-replacement tool corrupts a file from roughly 60KB to over 160KB; the anomaly is caught by its file-size jump before being committed, traced to a special-character collision in the replacement logic, and fixed |
| Verification | A script's own "did anything change" comparison reports a change on every run, including ones where nothing changed; traced to a newline-handling mismatch between the two values being compared |
| Closeout | A routine push of accumulated commits finds six commits — including all three finished scripts — had never actually been pushed to the remote repository that the deployment system polls, despite having been committed and gated as complete |

---

## Root Cause

No single root cause explains this arc — it's better understood as five independent instances of the same underlying pattern: a signal that looked authoritative (a tool's documented example, a UI's visible state, an assumption about data format, a comparison's result, a command's success message) was treated as sufficient evidence on its own, when in every case the actual ground truth required one more, independent check to confirm.

The most consequential of the five was the file-reference syntax issue, because it silently corrupted the majority of the secrets being migrated, using exactly the syntax the tool's own documentation presented as correct. Nothing about the push operation's own output indicated anything had gone wrong; the corruption was only found because a verification pass was run as a matter of discipline, not because anything signaled a problem.

---

## Remediation

- The secrets manager's actual setup state was confirmed directly rather than assumed from prior documentation, and the missing pieces (administrative account, project, machine identity, network reachability) were completed before any service migration began.
- The correct authentication credential and project identifier were found by direct inspection of the platform's own state, after two wrong values were identified and corrected.
- The seven corrupted secret values were re-pushed using a verified-correct method, with a byte-for-byte comparison run afterward to confirm correctness rather than relying on the fix's plausibility.
- Each of the three fetch-and-restart scripts was reviewed as literal source code before being run against production secrets, and the one script with an incorrect safety-guard assumption was corrected to match the real format of the credential it was protecting, rather than having its guard loosened indiscriminately.
- The corrupted documentation edit was reverted before it was ever committed, and the underlying tooling issue (a string-replacement pattern colliding with reserved characters) was fixed by switching to a literal-replacement method.
- The comparison-logic bug was fixed by aligning how both sides of the comparison handled trailing whitespace, so the check would correctly report "no change" on a genuine no-op run going forward.
- The six unpushed commits were identified during a routine closeout check, verified against the actual remote repository state using an authenticated query rather than trusting a prior command's own success message, and pushed.

---

## Prevention

- The secrets manager's setup state is no longer assumed from documentation; any future onboarding of a new service starts with confirming the platform's actual current state directly.
- Literal-content review (the actual code, not a description of it) is now the standing requirement before running any script against production secrets or configuration, not just a preference.
- Any safety guard built around an assumed data format is now expected to be verified against a real example of that data before being trusted, rather than discovered to be wrong only when it fires.
- Documentation or configuration edits performed via scripted string replacement are checked for an unexpected output-size change before being committed — a large, unexplained size delta is now treated as a hard stop, not a detail to investigate later.
- Routine session closeout now includes an explicit check of whether all local commits have actually reached the remote repository the deployment system depends on, rather than assuming "committed" implies "synced."

---

## Lessons Learned

1. **A tool's own documentation, a platform's visible UI state, and a command's own success message are all artifacts describing the system — none of them are the system itself, and any one of them can be wrong in a way that only an independent check against the actual underlying state will catch.** Every one of the first four issues in this arc trace back to trusting one of these artifacts past the point where it was actually telling the truth: a help-text example that didn't match the tool's real behavior, a UI panel that looked like the right one but wasn't, an assumption about a credential's format that had never been checked against a real example. The lesson isn't "be more careful reading documentation" — it's that documentation, UI state, and success messages are all claims, and claims about a system are not a substitute for checking the system.

2. **A safety guard that rejects unexpected input is the system working correctly, and the right response is to fix the assumption behind the guard, not to loosen the guard until it stops complaining.** The character-set guard did exactly its job — it stopped a write from happening when something didn't match what was expected. The temptation in that moment is to treat the guard itself as the obstacle; the actual fix is recognizing the guard's underlying assumption about the data was incomplete, correcting that assumption specifically, and keeping every other protection the guard still offered.

3. **A commit existing in a local repository and that same change being visible to whatever system depends on the remote are two different facts, and the gap between them is invisible until someone explicitly checks for it.** Every gate in this migration passed — the scripts worked, the services restarted cleanly, the secrets were correct — while the actual source-of-truth repository silently lacked any record of the work, for long enough that it could easily have gone unnoticed past the session it happened in. A system can be fully correct in its running state and still be out of sync with the artifact everything else is supposed to be rebuilt from.

---

*Environment: KONYKS-SERVER (Unraid) · SWAG · Authelia · Grafana · centralized secrets manager · Komodo/Periphery GitOps · homelab-incident-reports*
