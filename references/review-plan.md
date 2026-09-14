# Phase 3 — Review Plan (fresh context)

Review the fix plan from Phase 2 with fresh eyes: independently verify each proposed fix against
the actual code and PR diff, then produce an approved subset for implementation.

Steps below are named, not numbered.

## Execution context

**This phase must run with fresh context** — a fresh Claude Code chat in manual mode, or a subagent
under `/pr-fix run`. The isolation is the point: a reviewer carrying the planner's reasoning cannot
independently catch the planner's mistakes.

If you are a **subagent**, no human is reachable. Never call `AskUserQuestion`. Where a step below
asks the user to decide, write the decision set to `state.json` → `pending_decisions[]`, apply the
**recommended default** so the pipeline can continue, and lead your final report with every
decision you defaulted so the orchestrator can put it to the user.

## Prerequisites

- `.pr-fix/state.json`, `.pr-fix/report.md`, `.pr-fix/plan.md`

If any is missing:
> Missing prerequisite files. Run `/pr-fix report` and `/pr-fix plan` first — `/pr-fix status`
> shows where the pipeline stands.

## Step: Load context

If text followed `review-plan` in `$ARGUMENTS`, that's the planning session's brief — read it per
`_handoff.md` § Receiving a brief. It tells you where the planner is unsure; it does **not** shrink
what you verify below, and its claims are hypotheses, not findings.

Resolve the access path per `_github.md`, then run the staleness check per `_staleness.md`. A PR
that advanced since Phase 1 may have invalidated the plan's line numbers and snippets — warn and
recommend re-running `/pr-fix report` before continuing.

Read `state.json`, `report.md`, and `plan.md`.

## Step: Verify each fix independently

Fetch the PR diff **yourself** per `_github.md` (*PR diff*) — do not rely on Phase 2's reading of it.

For **each proposed fix**:

- **Does the code match?** Read the actual file at the stated line range. Does the plan's "Current
  code" match what's there? An abbreviated or drifted snippet means the fix was designed against
  something else.
- **Is the diagnosis real?** Re-read the surrounding context. Could this be a false positive?
- **Will the change work?** Does it actually fix the issue? Does it introduce new problems? Is
  there a simpler or more idiomatic approach the plan missed?
- **Is the risk assessment accurate?** Are there downstream effects the plan didn't consider?
- **Do the tests cover it?** Are the right tests added or modified? Do they exercise the actual fix
  or just its happy path?
- **Do the fixes interact?** Where several touch the same file or related code, will they compose,
  or does applying one invalidate another's "Current code"?

## Step: Assign verdicts

For each fix, one of:

- ✅ **Approve** — correct, well-designed, safe to apply as-is
- ⚠️ **Needs revision** — the diagnosis is right, the fix needs adjustment. Write specific guidance.
- ❌ **Reject** — false positive, wrong fix, or risk outweighs benefit. Explain why.

Record each on the fix in `state.json` → `fixes[].review_verdict` / `review_note`. Phase 3 updates
fixes **in place**; there is no separate approved-plan JSON.

## Step: Reconcile skipped findings

The plan deliberately scopes out some findings. Those are the planner's judgment calls, and the
**human** owns the final include/skip decision — so surface them rather than inheriting them. Do
not rely on the plan having flagged which ones need review.

**Diff report against plan.** Enumerate every finding ID in `report.md`. Any finding not addressed
by a fix in `plan.md` is a skipped finding, whether or not the plan listed it as one.

**Re-derive blocking coverage from the live reviews.** Independently fetch the reviews per
`_github.md` (*Reviews*) and take the latest non-superseded `CHANGES_REQUESTED` per reviewer —
whoever, human or bot, no privileged names. Decompose each into its distinct points, and confirm
every point maps to a planned fix or a justified `already-resolved` / `false-positive` skip.

Check that against **both** `state.json` → `blocking_coverage[]` **and** the live review bodies. A
blocking point the report itself missed must still be caught here — that is why this re-derivation
exists rather than a re-read of Phase 1's map. Any blocking point without a planned fix is a
**mandatory** decision item below; it may not be skipped on severity grounds.

**Sanity-check each skip** against the actual code, the same way you verified the fixes. Is
"Low/Info, not worth it" actually true? Could it be folded cheaply into an approved fix touching
the same file?

### Present the decisions in one batch

Sort the skips into three groups:

| Group | Handling |
|---|---|
| **Must decide** — blocking points without a fix; skips your check contradicts | One `AskUserQuestion` with these as options. Never auto-ratify. |
| **Worth offering** — cheap to fold into an approved fix in the same file | Fold into the same question as a multi-select. |
| **Routine** — Low/Info skips your check agrees with | Auto-ratify. List them in one summary line: "{n} Low/Info skips ratified: F-05, F-07, F-11." |

Use **one** `AskUserQuestion` with a multi-select for the first two groups, not one question per
finding. A report with nine Low/Info skips should not produce nine prompts before the user reaches
the actual plan. Offer at minimum **Keep skipped** vs. **Include**, plus **Fold into FIX-NN** where
it shares a file with an approved fix. Recommend the planner's call unless your check contradicts
it — and when it does, say so in the option description.

Record every skip in `state.json` → `skips[]` with its `decision` and whether the user ratified it.

## Step: Confirm with the user

Present the review as a table:

```
| # | Fix | Verdict | Notes |
|---|-----|---------|-------|
| FIX-01 | {title} | ✅ Approve | Looks correct |
| FIX-02 | {title} | ⚠️ Revise | {brief reason} |
| FIX-03 | {title} | ❌ Reject | {brief reason} |
```

For each ⚠️, show the specific revision guidance: what needs to change, what the revised approach
is, and an updated snippet where it helps.

Then ask the user to confirm. They may override any verdict (approve a rejected fix, reject an
approved one) and add revision notes. Fold in the skip decisions: anything they chose to
**Include** becomes a new approved fix or an addition to the fix it folds into; the rest stay
skipped with their ratification noted.

## Step: Write the approved plan

Write `.pr-fix/plan-approved.md`:

```markdown
# Approved Fix Plan — {PR_TITLE}

**PR**: {PR_URL}
**Reviewed**: {TIMESTAMP}
**Fixes approved**: {count_approved} of {count_total}
**Blocking points covered**: {n} fixed, {m} justified skips, 0 unmapped

---

## Approved Fixes (apply in this order)

{For each ✅: the fix block from plan.md, as-is}

{For each ⚠️: the fix block with the REVISED proposed change, rationale, and test changes}

---

## Rejected Fixes

| # | Fix | Reason |
|---|-----|--------|
| FIX-03 | {title} | {why} |

---

## Skip Decisions

| Finding | Decision | Ratified by |
|---------|----------|-------------|
| F-05 | Keep skipped | user |
| F-09 | Folded into FIX-02 | user |
| F-11 | Keep skipped | auto (Low, review agreed) |

---

## Review Notes

{Observations from the cross-check. Patterns. Concerns about the plan as a whole. Anything the
implementer should watch for.}
```

Set `state.json` → `phases.review-plan` to complete.

## Step: Hand off

Compose the `implement` brief per `_handoff.md`. The planning session remembers its own `plan.md`,
so the brief's main job is to say **what this review changed**.

> Approved plan written to `.pr-fix/plan-approved.md`.
>
> **Next step**: Go back to your **Phase 2 chat** (where you ran `/pr-fix plan`) and paste the
> prompt below. That chat already holds the reasoning behind each fix.

```
/pr-fix implement {BRIEF}
```

As a subagent, return the brief in your final report; the orchestrator resumes Phase 4 in its own
session.

## Error handling

- **Plan has no fixes** ("No actionable findings") — write a `plan-approved.md` saying "No fixes to
  apply" and hand off; don't manufacture work.
- **A file in the plan no longer exists or has changed** — flag the discrepancy and suggest
  re-running Phase 1 rather than approving a fix against code that moved.
- **The PR advanced since Phase 1** — handled by `_staleness.md`; record the drift in the review
  notes either way.
