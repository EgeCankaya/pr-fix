# Phase 3 — Review Plan (Fresh Chat)

Review the fix plan from Phase 2 with fresh eyes. This phase independently verifies each proposed fix against the actual code and PR diff, then produces an approved subset for implementation.

**This phase MUST run in a fresh Claude Code chat** — the fresh context prevents anchoring bias from the planning session.

## Prerequisites

- `.pr-fix/context.json` must exist (written by Phase 1)
- `.pr-fix/report.md` must exist (written by Phase 1)
- `.pr-fix/plan.md` must exist (written by Phase 2)

If any is missing, tell the user:
> Missing prerequisite files. Make sure you've run `/pr-fix report` and `/pr-fix plan` first.

## Step 1 — Load All Context (Fresh)

If text followed `review-plan` in `$ARGUMENTS`, that's the planning chat's handoff brief — read it
per `Workflows/_handoff.md` § Receiving a brief. It tells you where the planner is unsure; it does
not shrink what you verify in Step 2, and its claims are hypotheses, not findings.

1. Read `.pr-fix/context.json` for PR metadata.
2. Read `.pr-fix/report.md` for the original issue report.
3. Read `.pr-fix/plan.md` for the proposed fixes.

## Step 2 — Independent Verification

4. **Staleness check.** Confirm the PR hasn't advanced since Phase 1 captured it. Compare the live
   head against `head_sha` in `context.json`:
   ```bash
   gh pr view {PR_NUMBER} --json headRefOid -q '.headRefOid'
   ```
   If it differs from `context.json`'s `head_sha`, the report and plan may be stale — warn the user
   and recommend re-running `/pr-fix report` before continuing.

5. Fetch the PR diff **independently** — do not rely on Phase 2's interpretation:
   ```bash
   gh pr diff {PR_NUMBER}
   ```

6. For **each proposed fix** in the plan, perform this verification:

   a. **Read the actual file** at the line range specified. Does the "Current code" in the plan actually match what's in the file?

   b. **Verify the diagnosis**: Is the underlying issue real? Re-read the surrounding code context. Could this be a false positive?

   c. **Evaluate the proposed change**: Will it actually fix the issue? Does it introduce any new problems? Is there a simpler or more idiomatic approach?

   d. **Check the risk assessment**: Is it accurate? Are there downstream effects the plan missed?

   e. **Verify test changes**: Are the right tests being added/modified? Do they cover the actual fix adequately?

   f. **Check for interactions**: If multiple fixes touch the same file or related code, will they conflict or interact in unexpected ways?

## Step 3 — Assign Verdicts

6. For each fix, assign one of:
   - ✅ **Approve** — the fix is correct, well-designed, and safe to apply as-is
   - ⚠️ **Needs revision** — the diagnosis is right but the proposed fix needs adjustment. Write specific guidance on what to change.
   - ❌ **Reject** — the finding is a false positive, the fix is wrong, or the risk outweighs the benefit. Explain why.

## Step 3b — Reconcile Skipped Findings (mandatory; do not rely on the plan flagging them)

The plan deliberately scopes out some findings (a "Skipped Findings" section). These are judgment
calls the planner made — the human, not the planner, owns the final include/skip decision. Surface
**every** one for ratification, even those the plan did not explicitly mark "flag for review-plan".

a. **Diff report vs. plan.** Enumerate every finding ID in `report.md` (F-01, F-02, …). Any finding
   that is **not** addressed by a fix in `plan.md` is a skipped finding — whether or not the plan
   listed it under "Skipped Findings".

a2. **Blocking-review completeness cross-check (reviewer-agnostic).** Independently re-derive the
   current blocking reviews: from `gh api repos/{OWNER}/{REPO}/pulls/{PR_NUMBER}/reviews --paginate`,
   take the latest non-superseded `CHANGES_REQUESTED` review per reviewer (whoever — human or bot; do
   not privilege a name). Decompose each into its distinct points and confirm **every point maps to a
   planned fix** in `plan.md` (or to a justified already-resolved / false-positive skip). Check this
   against the report's §2 "Blocking-review coverage" map *and* against the live review bodies — a
   blocking point that the report itself missed must still be caught here. Any blocking point without a
   planned fix is a **mandatory** Step 3b item to raise with the user; a blocking point may not be
   skipped on severity grounds.

b. **Sanity-check each skip.** For each skipped finding, verify the plan's skip rationale against the
   actual code (same verification as Step 2). Is "Low/Info, not worth it" actually true? Could it be
   cheaply folded into an already-approved fix that touches the same file?

c. **Present every skip to the user via `AskUserQuestion`** — one question per skipped finding (or a
   single multi-select if several are independent), offering at minimum **Keep skipped** vs.
   **Include** (and **Fold into Fix N** when it shares a file with an approved fix). Default/recommend
   the planner's call unless your Step 3b check contradicts it. Do **not** silently inherit a skip.

## Step 4 — User Confirmation

7. Present the review results as a table:

```
| # | Fix | Verdict | Notes |
|---|-----|---------|-------|
| 1 | {title} | ✅ Approve | Looks correct |
| 2 | {title} | ⚠️ Revise | {brief reason} |
| 3 | {title} | ❌ Reject | {brief reason} |
```

8. For any ⚠️ items, show the specific revision guidance:
   - What part of the fix needs to change
   - What the revised approach should be
   - Updated code snippet if applicable

9. Ask the user to confirm:
   - "Does this look right? Should I write the approved plan?"
   - Let the user override any verdict (approve a rejected fix, reject an approved one)
   - The user can also provide additional revision notes
   - Fold the Step 3b decisions in: any skipped finding the user chose to **Include** becomes a new
     approved fix (or an addition to the fix it was folded into); the rest stay in the Rejected/Skipped
     table with the user's ratification noted.

## Step 5 — Write Approved Plan

10. Write `.pr-fix/plan-approved.md` containing:

```markdown
# Approved Fix Plan — {PR_TITLE}

**PR**: {PR_URL}
**Reviewed**: {TIMESTAMP}
**Fixes approved**: {count_approved} of {count_total}

---

## Approved Fixes (apply in this order)

{For each ✅ approved fix: copy the fix block from plan.md as-is}

{For each ⚠️ revised fix: copy the fix block but with the REVISED proposed change, rationale, and test changes}

---

## Rejected Fixes

| # | Fix | Reason |
|---|-----|--------|
| {n} | {title} | {why it was rejected} |

---

## Review Notes

{Any overall observations from the cross-check. Patterns noticed. Concerns about the plan as a whole. Anything the implementer should be aware of.}
```

## Step 6 — Hand Off

11. Compose the `implement` handoff prompt per `Workflows/_handoff.md`. The planning chat remembers
    its own `plan.md`, so the brief's main job is to say what this review changed. Then tell the user:
    > Approved plan written to `.pr-fix/plan-approved.md`.
    >
    > **Next step**: Go back to your **Phase 2 chat** (the one where you ran `/pr-fix plan`) and
    > paste the prompt below. That chat already has context about the codebase from planning.

    ```
    /pr-fix implement {BRIEF}
    ```

## Error Handling

- If the plan has no fixes (e.g., "No actionable findings"): inform the user and skip to handoff — write a plan-approved.md that says "No fixes to apply"
- If a file referenced in the plan no longer exists or has changed: flag it as a discrepancy, suggest the user re-run Phase 1
- If the PR has been updated since Phase 1 ran: warn that the report may be stale (check head SHA against context.json)
