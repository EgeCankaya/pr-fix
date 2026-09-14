# Phase 5 — Review Implementation (Fresh Chat)

Independently review the implemented fixes from Phase 4 with fresh eyes. Verify correctness, run tests, and produce a final verdict.

**This phase MUST run in a fresh Claude Code chat** — the fresh context ensures unbiased evaluation of the implementation.

## Prerequisites

- `.pr-fix/context.json` must exist (written by Phase 1)
- `.pr-fix/plan-approved.md` must exist (written by Phase 3)
- Either `.pr-fix/changes.md` or `.pr-fix/changes-failed.md` must exist (written by Phase 4)

If any is missing, tell the user:
> Missing prerequisite files. Make sure you've completed Phases 1–4 first.

## Step 1 — Load Context (Fresh)

If text followed `review-impl` in `$ARGUMENTS`, that's the implementing chat's handoff brief — read
it per `Workflows/_handoff.md` § Receiving a brief. It points at where the implementer is least
sure; it does not shrink what you verify in Steps 3–4, and its claims are hypotheses, not findings.

1. Read `.pr-fix/context.json` for PR metadata.
2. Read `.pr-fix/plan-approved.md` for the approved fix plan (what was supposed to be done).
3. Check which implementation summary exists:
   - If `.pr-fix/changes.md` exists and does NOT say "REDIRECTED": read it.
   - If `.pr-fix/changes-failed.md` exists: read it instead. Note that this means Phase 4 failed its verification loop.
4. Read `CLAUDE.md` for project conventions.

## Step 2 — Inspect the Actual Changes

Diff against the **baseline** Phase 4 recorded, so you review exactly the fix set — not any
pre-existing work the user kept in the tree. Fall back to a bare `git diff` only if the baseline
file is missing (older run), and note that caveat in the verdict.

```bash
BASE=$(cat .pr-fix/baseline_sha.txt 2>/dev/null)
```

5. View the implementation changes independently:
   ```bash
   git diff "${BASE:-HEAD}"
   ```

6. Also check the file-level summary:
   ```bash
   git diff --stat "${BASE:-HEAD}"
   ```

7. For **each modified file**, read the full file to understand the changes in context — not just the diff hunks.

## Step 3 — Independent Verification

8. For **each applied fix**, verify:

   a. **Plan compliance**: Does the implementation match what was approved in `plan-approved.md`? Any deviations?

   b. **Correctness**: Does the change actually fix the underlying issue? Read the surrounding code — could the fix introduce a new bug?

   c. **Code quality**: Is the code clean, readable, and consistent with project conventions (`CLAUDE.md`)? Check:
      - Naming consistency
      - Appropriate error handling
      - No unnecessary complexity added
      - Comments where needed (but not excessive)

   d. **Test coverage**: Were the specified tests added/modified? Do they actually test the fix? Are there edge cases not covered?

   e. **Side effects**: Does the change affect any other code paths? Check imports, function signatures, and callers.

## Step 4 — Run Tests Independently

9. Resolve the suite per `Workflows/_verify.md` yourself — don't copy Phase 4's list — then run its
   **Step B gates** and do the **Step C** code-health self-check on the changed files.

   > In review you are auditing, not fixing: run the Step B gates **as-is**. Do **not** run the
   > Step A auto-fixers (they would mutate the tree you're reviewing). If a format check reports a
   > diff, that is a finding against the implementation, not something to silently fix.

10. Compare results with Phase 4's reported results in `changes.md` (or `changes-failed.md`). Any
    discrepancies — e.g. Phase 4 claimed green but a gate fails now, or Phase 4 never ran a gate
    your resolution found — are themselves findings.

## Step 5 — Assign Per-Change Verdicts

13. For each applied fix, assign:
    - ✅ **Good** — correctly implemented, tests pass, no issues
    - ⚠️ **Needs work** — partially correct or has minor issues that should be addressed
    - ❌ **Should revert** — incorrectly implemented, introduces bugs, or the fix is worse than the original

14. Present the results:

```
## Implementation Review

| # | Fix | File | Verdict | Notes |
|---|-----|------|---------|-------|
| 1 | {title} | `{file}` | ✅ Good | Correctly implemented |
| 2 | {title} | `{file}` | ⚠️ Needs work | {issue} |
| 3 | {title} | `{file}` | ❌ Revert | {reason} |
```

## Step 6 — User Decision (for ⚠️ and ❌ items)

15. For each ⚠️ or ❌ item, ask the user how to proceed:

    For ❌ **Should revert**:
    - "Revert this fix" → see step 16 for the safe, hunk-scoped revert
    - "Accept as-is despite the concern"
    - "Flag for another iteration of Phase 4"

    For ⚠️ **Needs work**:
    - "Accept as-is — the issue is minor"
    - "Flag for another iteration of Phase 4"

16. **Apply reverts safely.** A whole-file `git checkout -- {file}` is unsafe here: fixes are
    grouped by file (Phase 2), and the file may also hold pre-existing work the user chose to keep
    (Phase 4 warns but allows it). So:
    - **If the file's only changes-since-baseline are this one fix** → revert the whole file to the
      baseline state:
      ```bash
      BASE=$(cat .pr-fix/baseline_sha.txt 2>/dev/null || git rev-parse HEAD)
      git checkout "$BASE" -- {file}
      ```
    - **If the file has other fixes or pre-existing changes you must preserve** → revert only this
      fix's hunks interactively (never the whole file):
      ```bash
      git restore -p --source="$(cat .pr-fix/baseline_sha.txt)" -- {file}
      ```
      and select only the hunks belonging to the rejected fix. If interactive selection isn't
      practical, re-edit the file by hand to undo just that fix.
    - After any revert, re-run the Step 4 gates to confirm the tree is still green.

## Step 7 — Write Final Verdict

17. Write `.pr-fix/verdict.md`:

```markdown
# Implementation Verdict — {PR_TITLE}

**PR**: {PR_URL}
**Reviewed**: {TIMESTAMP}

---

## Overall Verdict: {ACCEPT / NEEDS_ITERATION / REJECT}

---

## Per-Fix Results

| # | Fix | File | Verdict | Action Taken |
|---|-----|------|---------|-------------|
| 1 | {title} | `{file}` | ✅ Good | Kept |
| 2 | {title} | `{file}` | ⚠️ Needs work | Accepted as-is / Flagged for iteration |
| 3 | {title} | `{file}` | ❌ Revert | Reverted / Accepted |

---

## Verification Results

| Gate | Command | Result |
|------|---------|--------|
| Tests | `{command}` | ✅ {n} passed / ❌ {n} failed |
| Lint | `{command}` | ✅ Clean / ❌ {n} issues |
| Types | `{command}` | ✅ Clean / ❌ {n} errors |
| {other gate} | `{command}` | ✅ / ❌ {detail} |
| Code-health self-check (Step C) | — | ✅ No new duplication / deep nesting / ❌ {detail} |

---

## Remaining Concerns

{Any issues that weren't fully resolved. Things to watch for. Suggestions for follow-up.}

---

## Next Steps

{Based on the verdict:}

- **ACCEPT**: "All fixes are good. You can commit and push when ready."
- **NEEDS_ITERATION**: "Some fixes need another pass. Go back to the Phase 2 chat and run `/pr-fix implement` again after updating `.pr-fix/plan-approved.md`."
- **REJECT**: "The implementation has significant issues. Consider re-running the pipeline from Phase 2."
```

## Step 8 — Final Message

18. Based on the verdict:

    **If ACCEPT**:
    > ✅ All fixes verified. Verdict written to `.pr-fix/verdict.md`.
    >
    > You can now commit and push when ready. The changes are in your working tree.

    **If NEEDS_ITERATION** — compose the `implement` (iteration) handoff prompt per
    `Workflows/_handoff.md`, then:
    > ⚠️ Some fixes need another pass. Verdict written to `.pr-fix/verdict.md`.
    >
    > Go back to your Phase 2 chat and paste the prompt below to re-apply the adjusted fixes.

    ```
    /pr-fix implement {BRIEF}
    ```

    **If REJECT** — compose the `plan` (re-plan) handoff prompt per `Workflows/_handoff.md`, then:
    > ❌ Significant issues found. Verdict written to `.pr-fix/verdict.md`.
    >
    > To redesign the approach, open a **fresh Claude Code chat** and paste the prompt below. That
    > chat becomes your new Phase 2/4 chat.

    ```
    /pr-fix plan {BRIEF}
    ```

## Error Handling

- If `git diff` shows no changes: the implementation may have been reverted or committed already — ask the user what happened
- If tests fail that weren't failing before Phase 4: clearly flag these as regressions
- If a file in changes.md no longer has modifications (user may have manually reverted): note the discrepancy
