# Phase 4 — Implement Fixes (Same Chat as Phase 2)

Apply the approved fixes from Phase 3, run tests and linters, and produce an implementation summary.

**This phase should run in the SAME chat as Phase 2** (`/pr-fix plan`). That chat retains context about the codebase and the reasoning behind each fix, making implementation more accurate.

## Prerequisites

- `.pr-fix/context.json` must exist (written by Phase 1)
- `.pr-fix/plan-approved.md` must exist (written by Phase 3)

If `.pr-fix/plan-approved.md` is missing, tell the user:
> Missing approved plan. Run `/pr-fix review-plan` in a fresh chat first, then come back here.

If `.pr-fix/plan-approved.md` says "No fixes to apply", tell the user:
> The review found no fixes to implement. The pipeline is complete.

## Step 1 — Load Approved Plan

If text followed `implement` in `$ARGUMENTS`, that's the handoff brief from Phase 3 (or from Phase 5
on an iteration round) — read it first, per `Workflows/_handoff.md` § Receiving a brief. Where it
says the review changed a fix, apply what `plan-approved.md` says, not your memory of `plan.md`.

1. Read `.pr-fix/plan-approved.md` to get the list of approved fixes.
2. Read `.pr-fix/context.json` for PR metadata (including `head_sha` and `head_branch`).
3. Note the execution order specified in the approved plan.

## Step 1.5 — Anchor the working tree (REQUIRED before editing anything)

This phase edits files in place and the review diff in Phase 5 is computed from the working tree,
so the tree must (a) contain the PR's code and (b) start clean, or the reviewed diff will blend the
fixes with unrelated work and reverts (Phase 5) become unsafe.

4. **Confirm the right code is checked out.** Compare the local tree to the PR:
   ```bash
   git rev-parse --abbrev-ref HEAD
   git rev-parse HEAD
   ```
   - If the branch isn't `{HEAD_BRANCH}`, or `HEAD` differs from `head_sha` in `context.json`,
     **stop** and tell the user:
     > ⚠️ Working tree is on `{CURRENT}` but the approved plan was written against PR head
     > `{HEAD_SHA:0:8}` (`{HEAD_BRANCH}`). Run `gh pr checkout {PR_NUMBER}` first so fixes land on
     > the right code. (If the PR has legitimately advanced, re-run `/pr-fix report` to refresh.)
     Do not apply fixes until this matches.

5. **Require a clean baseline.** Check for uncommitted work (ignore the `.pr-fix/` bus itself):
   ```bash
   git status --porcelain | grep -v '^?? .pr-fix/' || true
   ```
   - If the output is **non-empty**, the tree is dirty. Warn the user:
     > ⚠️ You have uncommitted changes. If I apply fixes now, Phase 5's review diff will mix them
     > with your existing work, and per-file reverts could discard your changes. Recommended: commit
     > or `git stash` your work first, then re-run `/pr-fix implement`.
     Ask whether to proceed anyway. Only continue if the user explicitly accepts.

6. **Record the baseline** so every later diff is exactly the fix set:
   ```bash
   git rev-parse HEAD > .pr-fix/baseline_sha.txt
   ```
   All `git diff` commands in this phase and Phase 5 use this baseline (`git diff $(cat
   .pr-fix/baseline_sha.txt)`), not a bare `git diff`.

## Step 2 — Apply Fixes

7. For each approved fix, **in the specified order** (dependencies first):

   a. Read the target file in full.

   b. Locate the exact lines specified in the fix. If the code has changed since the plan was written (e.g., line numbers shifted), find the correct location by matching the "Current code" snippet.

   c. Apply the proposed change. Replace the "Current code" with the "Proposed change" from the plan.

   d. If test changes are specified:
      - If adding a new test: create the test function/method in the appropriate test file
      - If modifying an existing test: update the relevant assertions or parameters
      - Follow the project's test patterns (check existing tests for style: parametrization, fixtures, shared helpers)

   e. After each fix, run a cheap syntax check for the file's language if one exists — e.g.
      `python3 -m py_compile {FILE}`, `node --check {FILE}`, `bash -n {FILE}`. The full suite runs
      in Step 3; this only catches a broken edit before the next fix builds on it.

8. If a fix cannot be applied (code doesn't match, file missing, conflict with a previous fix): skip it, note it as "SKIPPED" in the summary, and continue with the remaining fixes.

## Step 3 — Run Verification (Bounded Retry Loop)

**CRITICAL: This step uses a strict retry budget. Maximum 3 attempts total.**

Resolve the suite per `Workflows/_verify.md` once, before attempt 1, and run the same commands on
every attempt; do not re-derive them here. Note its structure: **Step A auto-fixers (the project's
formatters and `--fix` linters) run first every attempt and do NOT count against the budget** —
they are deterministic formatting fixes, not substantive failures. The 3-attempt budget covers only
Step B gate failures (tests / lint / types / complexity) and Step C code-health reworks.

Track the current attempt number, starting at 1.

### Attempt {N} of 3

9. Run the **Step A auto-fixers** from `Workflows/_verify.md` (free — never counts as an attempt).

10. Run the **Step B gates** from `Workflows/_verify.md`, then do the **Step C** code-health
    self-check on the changed files.

11. **Evaluate results**:

   - **All green** → proceed to Step 4 (Show Changes). Record `verification_status: PASSED`.

   - **Failures detected AND attempt < 3** → enter the fix-and-retry block:

     a. **Diagnose**: Identify which specific fix caused the failure. Read the error output carefully:
        - Test failure → which test, what assertion, which file/line
        - Lint error → which rule, which file/line
        - Type error → which type mismatch, which file/line

     b. **Attribute**: Map the failure back to a specific fix from the plan. If the failure is in code you just changed, it's almost certainly related. If the failure is in *untouched* code, it may be a pre-existing issue — note it but don't try to fix it.

     c. **Fix** (targeted, minimal):
        - Only modify the specific lines causing the failure
        - Do NOT rewrite the entire fix — make the minimum change needed
        - If a lint error survived Step A's `--fix`: fix the specific rule by hand
        - If a type error: add the correct type annotation or cast
        - If a complexity or code-health failure: flatten with early returns / extract a helper —
          don't just suppress it with an inline comment
        - If a test failure: fix the implementation (not the test) unless the test expectation is wrong per the plan

     d. **Log** the attempt:
        ```
        Attempt {N}: {test|lint|type|complexity} failure in {file}:{line}
        Cause: {brief description}
        Fix applied: {what was changed}
        ```

     e. **Increment attempt counter** and go back to step 9.

   - **Failures detected AND attempt = 3** → **STOP**. Do not retry further. Record `verification_status: FAILED`. Proceed to Step 4 but write `changes-failed.md` instead of `changes.md` (see Step 5).

> **HARD RULE**: Never exceed 3 verification attempts. This prevents infinite token-burning loops. After 3 attempts, accept the failure and hand off to Phase 5 for human triage.

## Step 4 — Show Changes

Diff against the **baseline** recorded in Step 1.5 so the output is exactly the fix set (not blended
with any pre-existing work the user chose to keep):

```bash
BASE=$(cat .pr-fix/baseline_sha.txt)
```

10. Show the full diff of all changes (including any retry fixes):
    ```bash
    git diff "$BASE"
    ```

11. Also show a file-by-file summary:
    ```bash
    git diff --stat "$BASE"
    ```

## Step 5 — Write Implementation Summary

### If verification PASSED (all green within 3 attempts):

12. Write `.pr-fix/changes.md`:

```markdown
# Implementation Summary — {PR_TITLE}

**PR**: {PR_URL}
**Implemented**: {TIMESTAMP}
**Fixes applied**: {count_applied} of {count_approved}
**Verification**: ✅ PASSED (attempt {N} of 3)

---

## Applied Fixes

| # | Fix | File | Status |
|---|-----|------|--------|
| 1 | {title} | `{file}` | ✅ Applied |
| 2 | {title} | `{file}` | ✅ Applied |
| 3 | {title} | `{file}` | ⚠️ Skipped — {reason} |

---

## Files Modified

| File | Lines changed | What changed |
|------|--------------|-------------|
| `{file}` | +{n}/−{m} | {brief description} |

---

## Verification Results

**Suite source**: {override / CI + project instructions + tool config — see `Workflows/_verify.md`}

| Gate | Command | Result |
|------|---------|--------|
| Tests | `{command}` | ✅ {n} passed |
| Lint | `{command}` | ✅ Clean |
| Format check | `{command}` | ✅ Clean |
| Types | `{command}` | ✅ Clean |
| {other gate} | `{command}` | ✅ {result} |
| Code-health self-check (Step C) | — | ✅ No new duplication / deep nesting in changed files |

{One row per gate in the resolved suite. Mark a gate the project doesn't have as N/A, and one that
couldn't run locally as ⏭️ Not run — {reason}. Add short output excerpts for anything notable.}

---

## Retry Log

{If any retries were needed:}

| Attempt | What failed | What was fixed |
|---------|------------|---------------|
| 1 | {description} | {fix applied} |
| 2 | ✅ All green | — |

{If no retries: "All verification passed on first attempt."}

---

## Notes

{Any issues encountered. Fixes that were skipped and why. Anything the reviewer should pay attention to.}
```

### If verification FAILED (still broken after 3 attempts):

13. Write `.pr-fix/changes-failed.md` (note the `-failed` suffix):

```markdown
# ❌ Implementation FAILED — {PR_TITLE}

**PR**: {PR_URL}
**Implemented**: {TIMESTAMP}
**Fixes applied**: {count_applied} of {count_approved}
**Verification**: ❌ FAILED after 3 attempts

> **This implementation has unresolved failures.** Phase 5 will review
> what went wrong. The working tree contains the partially-applied changes.

---

## Applied Fixes

| # | Fix | File | Status |
|---|-----|------|--------|
| 1 | {title} | `{file}` | ✅ Applied |
| 2 | {title} | `{file}` | ❌ Applied but caused failures |
| 3 | {title} | `{file}` | ⚠️ Skipped — {reason} |

---

## Retry Log

| Attempt | What failed | Fix attempted | Outcome |
|---------|------------|--------------|---------|
| 1 | {error description} | {fix applied} | Still failing |
| 2 | {error description} | {fix applied} | Still failing |
| 3 | {error description} | {fix applied} | Still failing |

---

## Remaining Failures

One subsection per gate that is still failing:

### {Gate} — `{command}`
```
{failing output}
```

{Plus any Step C code-health concern that couldn't be flattened.}

---

## Root Cause Analysis

{Best assessment of why the fixes couldn't be made to pass:}
- {Is the original fix approach wrong?}
- {Is there a deeper architectural issue?}
- {Did the fixes interact in unexpected ways?}
- {Is there a pre-existing failure unrelated to the fixes?}

---

## Recommendation

{One of:}
- "Revert all changes and re-plan with a different approach" → user should run `/pr-fix clean` and start over
- "Revert specific fix(es) and keep the rest" → list which fixes to revert
- "The failures are pre-existing / unrelated — proceed to Phase 5 review"
```

Also write `.pr-fix/changes.md` as a redirect so Phase 5 knows to look at the failed file:
```markdown
# Implementation Summary — REDIRECTED

**Status**: ❌ FAILED — see `changes-failed.md` for details.
```

## Step 6 — Hand Off

### If PASSED:

14. Present the summary to the user:
    - How many fixes were applied vs. skipped
    - How many retry attempts were needed
    - All verification green

15. Compose the `review-impl` handoff prompt per `Workflows/_handoff.md` — it feeds a fresh-eyes
    review, so follow that file's attention-not-conclusions rule. Then tell the user:
    > ✅ Changes applied and verified. Summary written to `.pr-fix/changes.md`.
    >
    > **Next step**: Open a **fresh Claude Code chat** and paste the prompt below. A fresh session
    > will independently verify the implementation.

    ```
    /pr-fix review-impl {BRIEF}
    ```

### If FAILED:

14. Present the failure summary:
    - Which fixes were applied
    - What's still failing after 3 attempts
    - Root cause assessment

15. Compose the `review-impl` handoff prompt the same way (the FAILED items in its row apply), then
    tell the user:
    > ❌ Implementation has unresolved failures after 3 attempts. Details in `.pr-fix/changes-failed.md`.
    >
    > **Options**:
    > - Open a **fresh chat** and paste the prompt below — Phase 5 can triage what to keep vs. revert
    > - Run `/pr-fix clean` to wipe state and start the pipeline over with a revised approach
    > - Manually fix the remaining issues in this chat, then re-run the verification commands above

    ```
    /pr-fix review-impl {BRIEF}
    ```

16. **Do NOT commit or push.** Leave that to the user unless they explicitly ask.

## Error Handling

- If the approved plan references files that don't exist: skip those fixes, note in summary
- If a fix's "Current code" doesn't match the file: skip that fix (code may have changed), note the mismatch
- If a syntax error makes a file unparseable: count it as a verification failure and enter the retry loop
- **Never exceed 3 retry attempts** — this is a hard ceiling, not a guideline
