# Credential Guard: Word-Count Exemption Permits Credential-Path Writes

**Date:** 2026-09-02
**Severity:** P1 High
**Status:** Open — finding confirmed, mitigation not yet applied
**Affected:** `credential_guard.sh` (PreToolUse managed hook), agent file-write path
**Duration:** Present since the exemption branch was introduced; discovered 2026-09-01

---

## Summary

The credential guard blocks an agent from writing files whose content references credential-bearing paths. It carries an exemption intended to permit prose: if the line containing the matched path token runs to six words or more, and the token is not the entire trimmed value, the write is allowed. The exemption was added after a legitimate write — documentation describing the guard's own patterns — was denied for mentioning a credential filename in a sentence. The threshold is trivially satisfiable. Any content can be padded to six words, so the check does not distinguish prose from payload; it distinguishes short strings from long ones. The branch is marked unverified in its own source comment. Confirmed live during unrelated probe work by writing a two-line file whose body was an eleven-word sentence containing a credential filename: allowed, with its own audit entry recording the exemption category.

---

## Root Cause

### The control

The guard runs as a PreToolUse managed hook with a wildcard matcher, so it sees every tool call including `Write`. Its path check is a substring match, not path-anchored — a filename that merely *contains* a credential-path token matches, which is a separate over-broad behaviour and the reason a prose exemption was wanted in the first place.

### The exemption

The allow branch fires when two conditions hold:

- the line containing the matched token has a word count at or above six, and
- the trimmed value is not identical to the matched token

Content meeting both is allowed and audited under a category whose own name records that it is unverified.

### Why the threshold does not work

Word count is a proxy for "this is a sentence, not a value." It is a poor one. The distinction it actually draws is between short strings and long ones, and an agent producing content controls the length of what it produces. Six words of padding around a payload satisfies the check completely. No adversarial sophistication is required — the condition is met incidentally by almost any natural writing.

The guard's own source comment marks this branch as unverified beyond the test matrix of the pass that introduced it. That is an accurate self-assessment that was never followed up.

### How it was introduced

The branch was added in response to a false positive: a write whose body *explained the guard's patterns* was denied for naming a credential file in a sentence. The fix was correct in intent — that write should have been permitted — but the remedy widened the rule rather than narrowing the match. This is the recurring shape: a control produces a false positive, the false positive is real and annoying, and the relaxation applied is broader than the case that motivated it.

### Scope of what was actually tested

The finding rests on one exercised branch. Stated plainly, because a single allow is not a general statement that the write path is ungated:

| Variable | Status |
|---|---|
| Guard live and gating writes | Isolated — confirmed three ways |
| Path token in a ≥6-word prose body | Isolated — allowed via the exemption |
| Path token as a bare single-token value | Not tested — source says it still denies |
| Path token in the destination path rather than the body | Not tested |
| Body under six words containing the token | Not tested — the boundary is unexercised |
| A real credential value rather than a path string | Not tested — says nothing about value-shape detection |

Guard liveness was established before the test, not assumed: the managed settings wire the hook with a wildcard matcher, the audit log was actively being written during the session, and it holds several hundred historical entries for the write tool specifically.

---

## Remediation

Not yet applied. Three candidates, in ascending cost:

**Narrow the match instead of widening the allow.** The exemption exists because the path check is a substring match. Anchoring it to path-like positions removes most of the false positives the exemption was built to absorb, which in turn allows the exemption to be tightened or dropped. This addresses the cause rather than the symptom.

**Replace word count with a value-shape test.** The question the branch is trying to answer is "is this a reference or a payload." Word count cannot answer it. Whether the line contains something matching a credential *value* shape can — and that machinery already exists elsewhere in the same control.

**Reduce the branch to detection.** If neither is affordable, downgrade the exemption from allow to allow-and-alert, so an exempted write is at least visible rather than merely logged under a category nobody reads.

Any change here requires a re-pin under the existing hook procedure, and the guard's own history is a warning against a fast fix: this control has been hardened multiple times, and more than once the hardening introduced the next defect.

---

## Prevention

- Finding recorded in the operational trap notes alongside the untested variables above, so the next session inherits the boundary rather than rediscovering it.
- The untested rows are the follow-up work. In particular, the bare-single-token and destination-path cases are asserted from source and have never been exercised; source-asserted behaviour has been wrong before on this control.
- Open question worth its own pass: the audit category for this branch names itself unverified. A search for other categories carrying similar self-flagged uncertainty would likely surface siblings.

---

## Lessons Learned

**A control relaxed to fix a false positive should be narrowed at the match, not widened at the allow.** The false positive here was genuine — documentation about the guard was being blocked by the guard. But widening the allow to admit anything sentence-shaped permitted a far larger class than the one case that motivated it. When a control produces a false positive, the useful question is why the *match* was wrong, not what exception can be carved for it.

**A heuristic that measures the wrong property fails silently and looks fine in its own tests.** Word count as a proxy for "is this prose" passes every test case a well-intentioned author would write, because well-intentioned writing is verbose. The failure mode only appears against content that is deliberately or accidentally shaped to the threshold — which is to say, it never appears in testing and always appears in production. The branch's own comment flagged it as unverified, which is the system telling the truth about itself and being ignored.

**An in-process heuristic is not a boundary, and this is what that means concretely.** The guard's purpose is to stop an agent moving credentials into places they should not go. It runs in the same process as the thing it constrains, so it is a guardrail against a *complying* agent — useful for catching mistakes, worthless against intent. Documenting the exemption publicly costs little for exactly that reason: an agent that wanted to bypass it would not need to be told how.

---

*Environment: KONYKS-SERVER (Unraid) · credential_guard.sh managed hook · agent sandbox · homelab-incident-reports*
