# Phase 5 — Review Implementation (fresh context)

Independently review the implemented fixes from Phase 4: verify correctness, re-run the project's
checks, and produce a final verdict.

Steps below are named, not numbered.

## Execution context

**This phase must run with fresh context** — a fresh Claude Code chat in manual mode, or a subagent
under `/pr-fix run`. A reviewer carrying the implementer's reasoning cannot independently catch the
implementer's mistakes.

If you are a **subagent**: no human is reachable, so never call `AskUserQuestion`, and **never
revert anything**. Audit, write `verdict.md`, record each recommended keep/revert decision in
`state.json` → `pending_decisions[]`, and lead your final report with them. The orchestrator puts
them to the user and applies the reverts. Reverting on a guess is the one mistake in this pipeline
that destroys work.

## Prerequisites

- `.pr-fix/state.json`, `.pr-fix/plan-approved.md`, `.pr-fix/changes.md`

If any is missing:
> Missing prerequisite files. Complete Phases 1–4 first — `/pr-fix status` shows where things stand.

## Step: Load context

If text followed `review-impl` in `$ARGUMENTS`, that's the implementing session's brief — read it
per `_handoff.md` § Receiving a brief. It points at where the implementer is least sure. It does
**not** shrink what you verify, and its claims are hypotheses.

Resolve the access path per `_github.md`, then run the staleness check per `_staleness.md`. This
phase proceeds on drift — you are auditing the tree as it stands — but records it under Remaining
Concerns, because the fixes were designed against an older head.

Read `state.json`, `plan-approved.md`, `changes.md` (note its `Verification` field — PASSED or
FAILED — which tells you whether Phase 4 finished cleanly), and `CLAUDE.md` for conventions.

## Step: Inspect the changes

Diff against the baseline Phase 4 recorded, so you review exactly the fix set — not pre-existing
work the user chose to keep:

```bash
BASE=$(python3 -c 'import json;print(json.load(open(".pr-fix/state.json"))["baseline_sha"])' 2>/dev/null)
git diff "${BASE:-HEAD}"
git diff --stat "${BASE:-HEAD}"
```

If `baseline_sha` is missing (a run from before it existed), fall back to a bare `git diff` and
note the caveat in the verdict.

**Review fix-by-fix, not as one blended diff.** Phase 4 wrote an incremental patch per fix in
`.pr-fix/patches/`, and `state.json` → `fixes[].patch` names each one:

```bash
cat .pr-fix/patches/fix-01.patch
```

Reviewing `FIX-03`'s patch alone tells you what *that* fix did, which a combined diff of six fixes
across three files does not.

For each modified file, read it in full to see the change in context — subject to the same budget
rule as Phase 1: at or under `config.full_read_file_budget` modified files, read them all; above
it, read in full the files carrying non-trivial logic changes and read hunk-plus-context for the
rest, saying which in the verdict.

## Step: Verify each fix

For each applied fix:

- **Plan compliance** — does the implementation match what `plan-approved.md` approved? Note every
  deviation, including ones that look like improvements.
- **Correctness** — does it actually fix the underlying issue? Read the surrounding code: could it
  introduce a new bug?
- **Code quality** — clean, readable, consistent with `CLAUDE.md`? Check naming, error handling,
  unnecessary complexity, comments where needed and not where they aren't.
- **Test coverage** — were the specified tests added or modified? Do they exercise the fix, or only
  its happy path? What edge case is still uncovered?
- **Side effects** — check imports, signatures, and callers. Does this change any other code path?

## Step: Run the gates independently

**Resolve the suite yourself**, per `_verify.md` § Resolve the suite. Do **not** read
`.pr-fix/verify-resolved.md` first — that independence is the point of this phase. A reviewer that
inherits the implementer's idea of "the tests" cannot catch a gate the implementer never ran.

Run the **Step B gates** and do the **Step C** code-health self-check on the changed files.

> You are auditing, not fixing. Run the gates **as-is** and never run the Step A auto-fixers —
> they would mutate the tree you're reviewing. If a format check reports a diff, that is a finding
> against the implementation, not something to quietly fix.

Then make three comparisons, each of which can produce a finding:

1. **Against `gates.baseline`** — a gate red both now and at baseline with the same signature is
   pre-existing; say so and don't hold it against the implementation. A gate green at baseline and
   red now is a **regression**, and it is this PR's to fix.
2. **Against Phase 4's reported results** in `changes.md` — a gate Phase 4 claimed green that fails
   now is a finding, and a significant one.
3. **Against `.pr-fix/verify-resolved.md`** — now read it, and diff it against your own resolution.
   A gate you found that Phase 4 never ran is a finding, usually "CI enforces this and the
   implementation was never checked against it."

## Step: Assign per-fix verdicts

- ✅ **Good** — correctly implemented, gates pass, no issues
- ⚠️ **Needs work** — partially correct, or minor issues worth addressing
- ❌ **Should revert** — incorrect, introduces bugs, or worse than what it replaced

Record each in `state.json` → `fixes[].impl_verdict`. Present:

```
| # | Fix | File | Verdict | Notes |
|---|-----|------|---------|-------|
| FIX-01 | {title} | `{file}` | ✅ Good | Matches the approved plan |
| FIX-02 | {title} | `{file}` | ⚠️ Needs work | {issue} |
| FIX-03 | {title} | `{file}` | ❌ Revert | {reason} |
```

## Step: Decide what to keep

For each ⚠️ and ❌, put the choice to the user in **one** `AskUserQuestion` (multi-select), not one
prompt per fix:

- ❌ **Should revert** → *Revert this fix* / *Accept as-is despite the concern* / *Flag for another
  Phase 4 iteration*
- ⚠️ **Needs work** → *Accept as-is, the issue is minor* / *Flag for another Phase 4 iteration*

As a subagent, skip the question: record the recommendation and return it for the orchestrator.

### Applying a revert

Each fix has its own patch, so a revert is deterministic — no interactive hunk selection, no
re-editing by hand:

```bash
git apply -R --check ".pr-fix/patches/fix-{NN}.patch"   # verify it applies cleanly first
git apply -R ".pr-fix/patches/fix-{NN}.patch"
```

The `--check` run matters. If it fails, a later fix touched the same lines and the patches no
longer compose in reverse. In that case **stop and tell the user** which fixes overlap, and offer
to revert the overlapping set together (in reverse order, latest first) rather than forcing one
patch and leaving the file inconsistent:

```bash
for n in 05 04 03; do git apply -R ".pr-fix/patches/fix-$n.patch"; done
```

Record the outcome in `state.json` → `fixes[].action_taken`. After any revert, **re-run the Step B
gates** to confirm the tree is still green — a revert can break a later fix that depended on it.

## Step: Write the verdict

Write `.pr-fix/verdict.md`:

```markdown
# Implementation Verdict — {PR_TITLE}

**PR**: {PR_URL}
**Reviewed**: {TIMESTAMP}
**Phase 4 reported**: {PASSED | FAILED — {why it stopped}}

---

## Overall Verdict: {ACCEPT | NEEDS_ITERATION | REJECT}

---

## Per-Fix Results

| # | Fix | File | Verdict | Action taken |
|---|-----|------|---------|-------------|
| FIX-01 | {title} | `{file}` | ✅ Good | Kept |
| FIX-02 | {title} | `{file}` | ⚠️ Needs work | Accepted as-is |
| FIX-03 | {title} | `{file}` | ❌ Revert | Reverted via `patches/fix-03.patch` |

---

## Verification Results

| Gate | Command | At baseline | Now | Verdict |
|------|---------|-------------|-----|---------|
| Tests | `{command}` | ✅ pass | ✅ {n} passed | — |
| Types | `{command}` | ❌ fail | ❌ fail (same signature) | pre-existing |
| {gate} | `{command}` | ✅ pass | ❌ fail | **regression — blocks** |
| Code-health self-check (Step C) | — | — | {result} | — |

**Suite resolution**: {matches `verify-resolved.md` | differs — {which gates, and what that means}}

---

## Blocking-Review Coverage

Whether the fixes actually clear the reviews that blocked the PR.

| Review point | Finding | Fix | Status |
|--------------|---------|-----|--------|
| {point} | F-01 | FIX-01 | ✅ addressed |
| {point} | F-02 | — | ⚠️ justified skip — {reason} |

---

## Remaining Concerns

{Anything unresolved. Drift since Phase 1, if the staleness check found it. Edge cases the tests
still don't cover. Deviations from the approved plan that were accepted.}

---

## Next Steps

- **ACCEPT** — "All fixes verified. Run `/pr-fix respond` to draft the commit message and the
  reviewer reply, or commit and push yourself."
- **NEEDS_ITERATION** — "Some fixes need another pass. Go back to the Phase 2/4 chat and run
  `/pr-fix implement` with the brief below."
- **REJECT** — "The implementation has significant issues. Re-plan from `/pr-fix plan`."
```

Set `state.json` → `phases.review-impl` to complete.

## Step: Final message

**ACCEPT** — compose the `respond` brief per `_handoff.md`:
> ✅ All fixes verified. Verdict written to `.pr-fix/verdict.md`. The changes are in your working
> tree — nothing has been committed.
>
> **Next step**: `/pr-fix respond` drafts a commit message and a reviewer-facing comment from the
> blocking-review coverage map, so you don't have to re-explain the pipeline's reasoning by hand.

```
/pr-fix respond {BRIEF}
```

**NEEDS_ITERATION** — compose the `implement` (iteration) brief:
> ⚠️ Some fixes need another pass. Verdict written to `.pr-fix/verdict.md`.
>
> Go back to your Phase 2/4 chat and paste the prompt below.

```
/pr-fix implement {BRIEF}
```

**REJECT** — compose the `plan` (re-plan) brief:
> ❌ Significant issues found. Verdict written to `.pr-fix/verdict.md`.
>
> To redesign the approach, open a **fresh chat** and paste the prompt below. That chat becomes
> your new Phase 2/4 session.

```
/pr-fix plan {BRIEF}
```

## Error handling

- **`git diff` shows no changes** — the work may have been reverted or already committed. Ask the
  user what happened rather than reporting a clean review of an empty diff.
- **Gates fail that were green at baseline** — regressions. Flag them prominently; they are the
  single most important output of this phase.
- **A file in `changes.md` no longer shows modifications** — the user may have reverted manually.
  Note the discrepancy.
- **A patch won't reverse-apply** — never force it. Report the overlap, as above.
