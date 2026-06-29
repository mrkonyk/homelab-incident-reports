# Credential Rotation Script Bugs — Three Compounding Failures Render do_push_rotation.sh Inoperable

**Date:** 2026-06-29
**Severity:** P2 Medium
**Status:** Resolved
**Affected:** do_push_rotation.sh, GitHub PAT rotation, git fetch verification step
**Duration:** Script had been inoperable since its initial commit (2026-06-23); no successful end-to-end rotation had ever been performed using it

---

## Summary

Routine investigation of the GitHub push-PAT rotation script (`do_push_rotation.sh`) found three
independent bugs, each of which would cause the script to fail in a distinct way. The first was an
unbound variable reference that caused the secrets-fetch step to either crash under `set -u` or pass
an empty token to the secrets manager CLI, silently returning no value. The second and third were in
the verification step (step 5): the main repository clone used a plain URL with no embedded
credentials, so the git fetch could never authenticate; and even if authentication had been possible,
the grep filter used to clean up the fetch output exits 1 when nothing passes through it, which
`pipefail` propagates as a script failure even on a fully successful rotation. A fourth issue was
identified during the fix: the most obvious remediation for the fetch-authentication bug — embedding
the PAT directly in the git URL — would have exposed the token in process argv, visible via `ps aux`
and `/proc/<pid>/cmdline` to any process on the host.

---

## Timeline

| Time | Event |
|------|-------|
| 2026-06-23 | `do_push_rotation.sh` initially committed as part of the PAT rotation tooling |
| 2026-06-29 | Script reviewed during a broader audit of Infisical integration scripts |
| 2026-06-29 | Bug 1 identified: `--token "$INFISICAL_TOKEN"` references a variable that is never set; `source /root/.infisical_env` provides client-id and client-secret but not a JWT token |
| 2026-06-29 | Reference fix pattern identified: `fetch_grafana_secrets.sh` uses `infisical login --method=universal-auth` to obtain a short-lived token before the secrets-get call |
| 2026-06-29 | Bug 2 identified: main repo remote URL is `https://github.com/mrkonyk/homelab-infra.git` with no embedded credentials; git cannot authenticate from shell for this clone |
| 2026-06-29 | Bug 3 identified: `grep -v '^From\|^$'` in the step-5 pipeline exits 1 when a clean fetch produces no refs to display (only the "From" line, which grep strips); `pipefail` propagates exit 1 as script failure |
| 2026-06-29 | Argv-exposure issue identified in the candidate fix: embedding `$NEW_PAT` in a git URL argument would expose the token in `/proc/<pid>/cmdline` and `ps aux` output |
| 2026-06-29 | GIT_ASKPASS pattern selected: temp script reads PAT from environment variable, not from argv |
| 2026-06-29 | All three bugs fixed; PAT argv-exposure independently verified via `/proc/<pid>/cmdline` probe during a live background fetch |
| 2026-06-29 | End-to-end test: `env -i` clean-shell run completes steps 1–4 successfully; step 5 fetch authenticates and exits 0; 3 of 25 stack `.git/config` files spot-checked and confirmed updated |

---

## Root Cause

**Bug 1 — Unbound `$INFISICAL_TOKEN`:** The script sourced `/root/.infisical_env` and then called
`infisical secrets get ... --token "$INFISICAL_TOKEN"`. The `.infisical_env` file sets
`INFISICAL_UNIVERSAL_AUTH_CLIENT_ID`, `INFISICAL_UNIVERSAL_AUTH_CLIENT_SECRET`, and metadata
variables, but does not set `INFISICAL_TOKEN`. Under `set -u` (which the script uses), referencing
an unbound variable is an immediate fatal error. The script would have exited at the first
`infisical secrets get` invocation on every run, before touching any of the five propagation targets.
The secrets manager CLI's universal-auth login pattern — calling `infisical login
--method=universal-auth` to exchange the client-id/secret pair for a short-lived JWT, then passing
that JWT via `--token` — was already established in a sibling script (`fetch_grafana_secrets.sh`)
but had not been carried over to this one.

**Bug 2 — No credentials in main repo remote URL:** The main repository clone at
`/mnt/cache_ssd/appdata/komodo/repos/homelab-infra` uses a plain `https://github.com/...` remote
URL with no embedded token. The 25 Komodo stack checkouts in `stacks/*/` all have
`https://token:<PAT>@github.com/...` URLs, which are set and maintained by Komodo's GitProvider
mechanism. The main clone predates the current credential management scheme and was never updated to
carry an embedded token. Step 5's `git -C "$REPO" fetch --dry-run` was therefore always going to
fail with a credentials prompt that had no answer.

**Bug 3 — `grep -v` exits 1 on clean fetch:** The step-5 pipeline was
`git fetch --dry-run 2>&1 | grep -v '^From\|^$' | head -3`. On a successful fetch with no pending
changes, git emits only `From https://github.com/...` to stdout (via the `2>&1` redirect). `grep
-v` strips this line, produces no output, and exits 1. With `set -eo pipefail`, the rightmost
non-zero exit in the pipeline determines the pipeline's exit status: git exits 0, grep exits 1, head
exits 0 → pipeline exits 1 → `set -e` kills the script. A rotation that successfully propagated the
new PAT to all five targets would still exit non-zero if the repository was up to date — which it
would be on any rotation that didn't coincide with a pending upstream commit.

**Argv-exposure issue in candidate fix:** The straightforward fix for Bug 2 was to pass the
embedded-token URL as an argument to `git fetch --dry-run`. This would have put `$NEW_PAT` directly
in argv, visible to any process on the host via `ps aux` and `/proc/<pid>/cmdline` for the duration
of the fetch. This was specifically the pattern identified in a prior credential-exposure incident
involving a similar git push command. The fix used GIT_ASKPASS instead: a `chmod 700` temp script
that echoes the token from an environment variable (`$_GIT_PAT`) is registered as the credential
helper, git runs against the plain URL and calls the script for authentication, and the token never
appears in any process's argv.

---

## Remediation

1. **Bug 1:** Added an explicit Infisical login step before the secrets-get call, following the
   pattern already established in `fetch_grafana_secrets.sh`. The login call uses universal-auth with
   the client-id and client-secret from `.infisical_env` to obtain a short-lived JWT (`$TOKEN`),
   validates it starts with `ey`, passes it to the secrets-get call, and unsets it afterward.

2. **Bugs 2 and 3 (combined in step-5 rewrite):** The step-5 git fetch was rewritten to:
   - Create a `mktemp` askpass script (`chmod 700`) that reads `$_GIT_PAT` from its inherited
     environment and returns `token` for username prompts and the PAT for password prompts.
   - Set `_GIT_PAT="$NEW_PAT"` and `GIT_ASKPASS="$_ASKPASS"` as environment variables for the git
     invocation (not as argv).
   - Run `git fetch --dry-run origin` against the plain URL, letting GIT_ASKPASS handle
     authentication transparently.
   - Wrap the grep filter as `{ grep -Ev '^(From |$)' || true; }` to prevent grep's "no output"
     exit code from propagating through pipefail on a clean fetch.
   - Remove the askpass temp file after the fetch.

3. **End-to-end verification** in a clean shell (`env -i` with only `PATH` set) confirmed the login
   step, secrets fetch, and all four propagation targets completed successfully. The step-5 fetch
   authenticated correctly and exited 0. A live `/proc/<pid>/cmdline` probe during a background
   fetch execution confirmed the PAT was absent from argv.

---

## Prevention

- The universal-auth login pattern (`infisical login --method=universal-auth` → short-lived JWT →
  `--token "$TOKEN"` → `unset TOKEN`) is now established in both scripts that use the Infisical
  CLI and should be used in any future script that calls `infisical secrets get`.
- GIT_ASKPASS is now the required pattern for any git operation that needs to authenticate with a
  PAT from a script. Embedding tokens in git URLs as argv arguments reintroduces the exact exposure
  vector identified in the June 2026 credential-exposure incident.
- Any new rotation or configuration script should be run end-to-end (including its verification
  step) in a clean environment before being committed as working, not just reviewed for obvious
  logical errors. All three bugs in this script would have been caught on first execution.

---

## Lessons Learned

1. **A script that has never been run end-to-end is not a working script — it is a hypothesis.** `do_push_rotation.sh` was reviewed, committed, and referenced in documentation as the rotation mechanism. No successful end-to-end execution had ever been performed. All three bugs were in code paths that would have been reached within seconds of first real use. Writing a script, reviewing it, and committing it are necessary conditions for it being correct; they are not sufficient ones.

2. **"The existing pattern" is only a reference if it is actually followed.** The correct Infisical auth pattern existed in a sibling script in the same directory. The rotation script used a different, broken pattern that referenced a variable the sibling script never set. The sibling script was the established reference; the rotation script independently re-derived the auth approach and got it wrong. Consistency across scripts that call the same CLI reduces this class of bug to a copy-paste check rather than a re-derivation.

3. **Fixing one bug in a pipeline can reveal a second bug that the first bug had masked.** Bug 1 caused the script to crash before reaching step 5. Fixing Bug 1 allowed Bug 2 and Bug 3 to become visible. The same sequencing applied to the argv-exposure risk: it was only discoverable because the fix was being designed, not because it had ever been reached in production. Investigating a known bug in a sequence is an opportunity to check what the known bug had been hiding.

---

*Environment: KONYKS-SERVER (Unraid) · do_push_rotation.sh · Infisical secrets manager · GitHub PAT · GIT_ASKPASS · homelab-incident-reports*
