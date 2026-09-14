# Phase 4 — Implement Fixes

Apply the approved fixes from Phase 3, run the project's checks, and produce an implementation
summary.

Steps below are named, not numbered.

## Execution context

**This phase runs in the same session as Phase 2** — the chat where you ran `/pr-fix plan`, or the
orchestrator's own session under `/pr-fix run`. That session holds the reasoning behind each fix,
which makes implementation more accurate than re-deriving it from the plan text.

This is the only phase that edits source files. It never commits and never pushes.

## Prerequisites

- `.pr-fix/state.json` and `.pr-fix/plan-approved.md`

If `plan-approved.md` is missing:
> Missing approved plan. Run `/pr-fix review-plan` first, then come back here.

If it says "No fixes to apply":
> The review found no fixes to implement. The pipeline is complete.

## Step: Load the approved plan

If text followed `implement` in `$ARGUMENTS`, that's the brief from Phase 3 (or from Phase 5 on an
iteration round) — read it first per `_handoff.md` § Receiving a brief. Where it says the review
changed a fix, apply what `plan-approved.md` says, **not your memory of `plan.md`**. Your own
memory of planning is an asset for understanding the code and a liability for recalling verdicts.

Read `plan-approved.md` and `state.json` (fixes, their verdicts, execution order, config).

## Step: Anchor the working tree

This phase edits files in place, and Phase 5 computes its review diff from the working tree. The
tree must hold the PR's code and start from a known baseline, or the reviewed diff blends the fixes
with unrelated work and per-fix reverts become unsafe.

Run the staleness check per `_staleness.md`. This phase is one of the two that **stops** on drift:
never apply fixes designed for code that has since moved. On an iteration round the tree is
legitimately dirty with round-1 fixes — `_staleness.md` covers that exception.

**Require a clean baseline.** Check for uncommitted work, ignoring the message bus itself:

```bash
git status --porcelain | grep -v '^?? \.pr-fix/' || true
```

If non-empty (and this is not an iteration round):

> ⚠️ You have uncommitted changes. If I apply fixes now, Phase 5's review diff will mix them with
> your existing work, and per-fix reverts could discard your changes. Recommended: commit or
> `git stash` first, then re-run `/pr-fix implement`.

Ask whether to proceed anyway; continue only on an explicit yes, and record the accepted dirty
baseline in the eventual handoff brief.

**Record the baseline** so every later diff is exactly the fix set:

```bash
git rev-parse HEAD
```

Write it to `state.json` → `baseline_sha`. Every `git diff` in this phase and Phase 5 is taken
against that SHA, never a bare `git diff`.

## Step: Run the baseline gates

**Before applying anything**, resolve the suite and run the Step B gates once against the untouched
tree.

Read `.pr-fix/verify-resolved.md` (written by Phase 2). If it's missing, resolve the suite per
`_verify.md` and write it now. Do not re-derive it when the file exists — Phase 2 designed the
fixes against that resolution, and quietly using a different one is how a plan and its verification
drift apart.

Run the Step B gates and record each result in `state.json` → `gates.baseline` with its failure
signature per `_verify.md` § Failure signatures.

This costs one suite run and buys the rest of the phase: from here, "pre-existing failure" is a set
difference rather than a judgment call. Without it, a red gate in untouched code is a guess that
either burns the retry budget on someone else's bug or hides a real regression.

If a gate is already red at baseline, say so now:

> ℹ️ {Gate} is already failing before any fix is applied ({signature}). It won't count against this
> run, and Phase 5 will see the same baseline.

## Step: Apply fixes

Set up a temporary index so each fix can be captured as its own patch without touching the user's
real index:

```bash
BASE=$(python3 -c 'import json;print(json.load(open(".pr-fix/state.json"))["baseline_sha"])')
mkdir -p .pr-fix/patches
export GIT_INDEX_FILE=.pr-fix/tmp.index
git read-tree "$BASE"
PREV_TREE=$(git write-tree)
```

For each approved fix, **in the plan's execution order** (dependencies first):

- **Read the target file in full.**
- **Locate the exact lines.** If line numbers have shifted since the plan was written, find the
  right place by matching the "Current code" snippet rather than trusting the numbers.
- **Apply the change** — replace "Current code" with "Proposed change".
- **Apply the test changes.** Adding a test: create it in the appropriate test file. Modifying one:
  update the relevant assertions or parameters. Follow the project's existing test patterns —
  parametrization, fixtures, shared helpers — rather than introducing your own.
- **Syntax-check the file** if a cheap check exists: `python3 -m py_compile {FILE}`,
  `node --check {FILE}`, `bash -n {FILE}`. The full suite runs later; this catches a broken edit
  before the next fix builds on it.
- **Capture the fix's patch**:

  ```bash
  git add -A .
  TREE=$(git write-tree)
  git diff "$PREV_TREE" "$TREE" > ".pr-fix/patches/fix-{NN}.patch"
  PREV_TREE=$TREE
  ```

  Record the path in `state.json` → `fixes[].patch` and set `fixes[].applied`.

When every fix is applied, release the temporary index:

```bash
unset GIT_INDEX_FILE
rm -f .pr-fix/tmp.index
```

Per-fix patches exist so Phase 5 can revert a single rejected fix deterministically with
`git apply -R`, instead of driving an interactive hunk selector or re-editing a file by hand. They
also let Phase 5 review fix-by-fix rather than reading one blended diff.

> If the temporary-index approach isn't available, fall back to copying each file the fix will
> touch to `.pr-fix/tmp/before/` beforehand and writing `diff -u` output afterwards. The patches
> matter; the mechanism doesn't.

If a fix **cannot** be applied (code doesn't match, file missing, conflicts with an earlier fix):
skip it, set `fixes[].applied = "skipped"` with the reason, and continue with the rest.

## Step: Verify, with a no-progress stop

Run the suite from `.pr-fix/verify-resolved.md` — the same commands on every attempt; do not
re-derive them between attempts.

**Step A auto-fixers run first on every attempt and never count against the budget.** They are
deterministic formatting, not substantive failures. The budget covers only Step B gate failures and
Step C code-health reworks.

### The stop condition

The old rule was a flat "maximum 3 attempts", which counts the wrong thing: it kills a run that is
converging (12 failures → 4 → 1 → stopped) and wastes three full suite runs on one that isn't
moving at all. The rule is now **progress**, with a ceiling:

**Stop when either holds:**

1. **No progress** — the set of failing gates *and* their signatures are identical to the previous
   attempt. Two identical results mean the last edit changed nothing that matters; a third won't
   either.
2. **Ceiling reached** — `config.max_retry_attempts` (default 3) attempts have run.

Record every attempt in `state.json` → `gates.implement.attempts[]` with its failing gates,
signatures, and the action taken. That list *is* the stop condition's input, so write it before
deciding whether to continue.

### Each attempt

- Run the **Step A auto-fixers** (free).
- Run the **Step B gates**, then do the **Step C** code-health self-check on the changed files.
- **Subtract the baseline.** Per `_verify.md` § The baseline run: a gate failing now *and* at
  baseline with the same signature is pre-existing — note it, don't fix it, don't count it as a
  failure of this run. Only regressions against the baseline are yours.
- **Evaluate:**
  - **No regressions and no Step C concern** → `gates.implement.final = "PASSED"`. Go to *Show
    changes*.
  - **Regressions, and neither stop condition met** → diagnose and retry (below).
  - **Regressions, and a stop condition met** → `final = "FAILED"`. Stop. Go to *Show changes*.

### Diagnose and retry

- **Diagnose** — read the error output precisely. Which test, which assertion, which file:line;
  which lint rule; which type mismatch.
- **Attribute** — map the failure to a specific fix. A failure in code you just changed is almost
  certainly that fix. A failure in untouched code that is *not* in the baseline set is still yours:
  something you changed broke a caller.
- **Fix, minimally** — change only the lines causing the failure. Do not rewrite the whole fix.
  - Lint error surviving Step A's `--fix` → fix that rule by hand.
  - Type error → add the correct annotation or cast.
  - Complexity / code-health failure → flatten with early returns, extract a helper. Never suppress
    it with an inline ignore comment.
  - Test failure → fix the implementation, not the test, unless the plan establishes the test's
    expectation was wrong.
- **Log it** to `gates.implement.attempts[]`, then run the next attempt.

> **Never let the loop run unbounded.** Both stop conditions are hard. After stopping, hand the
> failure to Phase 5 for triage rather than continuing to edit.

## Step: Show changes

Diff against the baseline so the output is exactly the fix set:

```bash
BASE=$(python3 -c 'import json;print(json.load(open(".pr-fix/state.json"))["baseline_sha"])')
git diff "$BASE"
git diff --stat "$BASE"
```

## Step: Write the implementation summary

Write `.pr-fix/changes.md`. **There is one summary file in both outcomes** — the status lives in a
field, not in the filename. (Earlier versions wrote `changes-failed.md` plus a `changes.md` stub
saying "REDIRECTED"; Phase 5 then had to branch on file content to find the real one.)

````markdown
# Implementation Summary — {PR_TITLE}

**PR**: {PR_URL}
**Implemented**: {TIMESTAMP}
**Verification**: {✅ PASSED | ❌ FAILED}
**Stopped after**: attempt {N} of {max} — {all gates green | no progress between attempts {N-1} and {N} | ceiling reached}
**Fixes applied**: {count_applied} of {count_approved}

{If FAILED:}
> **This implementation has unresolved failures.** The working tree holds the partially-applied
> changes. Phase 5 will triage what to keep and what to revert.

---

## Applied Fixes

| # | Fix | File | Status | Patch |
|---|-----|------|--------|-------|
| FIX-01 | {title} | `{file}` | ✅ Applied | `patches/fix-01.patch` |
| FIX-02 | {title} | `{file}` | ❌ Applied but causing failures | `patches/fix-02.patch` |
| FIX-03 | {title} | `{file}` | ⚠️ Skipped — {reason} | — |

---

## Files Modified

| File | Lines changed | What changed |
|------|--------------|-------------|
| `{file}` | +{n}/−{m} | {brief description} |

---

## Verification Results

**Suite**: `.pr-fix/verify-resolved.md` ({override / CI + project instructions + tool config})

| Gate | Command | At baseline | Now | Verdict |
|------|---------|-------------|-----|---------|
| Tests | `{command}` | ✅ pass | ✅ {n} passed | — |
| Lint | `{command}` | ✅ pass | ✅ clean | — |
| Types | `{command}` | ❌ fail | ❌ fail (same signature) | pre-existing, not this run |
| {gate} | `{command}` | ✅ pass | ❌ fail | **regression** |
| Code-health self-check (Step C) | — | — | ✅ no new duplication / deep nesting | — |

{Mark a gate the project doesn't have as N/A, and one that couldn't run locally as
⏭️ Not run — {reason}. Add short output excerpts for anything notable.}

{If any gate was red at baseline and is now green: "Incidentally repaired: {gate} ({signature})."}

---

## Attempt Log

| Attempt | Failing gates | Signature | Action taken |
|---------|--------------|-----------|--------------|
| 1 | tests | `test_backoff.py::test_ceiling AssertionError` | clamped the exponent |
| 2 | — | — | all green |

{If no retries: "All gates green on the first attempt."}

---

{If FAILED, add:}

## Remaining Failures

### {Gate} — `{command}`
```
{failing output}
```

## Root Cause Assessment

- {Is the fix approach wrong?}
- {Is there a deeper architectural issue?}
- {Did fixes interact unexpectedly?}
- {Which failures are pre-existing per the baseline, and therefore not this run's?}

## Recommendation

{One of: revert specific fixes and keep the rest (name them) / re-plan with a different approach
(`/pr-fix plan`) / the remaining failures are pre-existing, proceed to Phase 5.}

---

## Notes

{Fixes skipped and why. Deviations from the approved plan. Anything the reviewer should watch.}
````

Set `state.json` → `phases.implement` to complete (or `failed`).

## Step: Hand off

Present: fixes applied vs. skipped, attempts used and why it stopped, gate results against the
baseline.

Compose the `review-impl` brief per `_handoff.md` — a fresh-eyes review, so follow the
attention-not-conclusions rule. Point the reviewer at `gates.baseline` rather than characterizing
any failure as pre-existing yourself.

**If PASSED:**
> ✅ Changes applied and verified. Summary written to `.pr-fix/changes.md`.
>
> **Next step**: Open a **fresh Claude Code chat** and paste the prompt below.

**If FAILED:**
> ❌ Implementation has unresolved failures ({why it stopped}). Details in `.pr-fix/changes.md`.
>
> **Options**: open a fresh chat and paste the prompt below so Phase 5 can triage keep-vs-revert;
> run `/pr-fix plan` to redesign the approach; or fix the remainder here and re-run the suite.

```
/pr-fix review-impl {BRIEF}
```

**Do NOT commit or push.** Phase 6 (`/pr-fix respond`) drafts the commit message when the user asks
for it; the user decides when anything lands.

## Error handling

- **Approved plan references a missing file** — skip that fix, note it in the summary.
- **"Current code" doesn't match the file** — skip that fix; the code moved. Note the mismatch.
- **A syntax error makes a file unparseable** — counts as a gate failure; enter the retry loop.
- **Both stop conditions are hard ceilings** — never exceed them.
