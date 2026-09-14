# Phase 2 — Fix Plan

Read the issue report from Phase 1 and create a concrete, actionable fix plan. This phase converts advisory suggestions into specific code changes.

## Prerequisites

- `.pr-fix/context.json` must exist (written by Phase 1)
- `.pr-fix/report.md` must exist (written by Phase 1)

If either is missing, tell the user:
> Missing prerequisite files. Run `/pr-fix report #PR` in a fresh chat first.

## Step 1 — Load Context

If text followed `plan` in `$ARGUMENTS`, that's the handoff brief from Phase 1 (or from Phase 5 on a
re-plan) — read it first, per `Workflows/_handoff.md` § Receiving a brief.

1. Read `.pr-fix/context.json` to get PR metadata (number, repo, branches).
2. Read `.pr-fix/report.md` — the full issue report from Phase 1.
3. Read `CLAUDE.md` for project conventions, coding standards, test patterns.

## Step 2 — Triage Findings

4. From the report's §3 (New Findings), select all findings rated **Medium or above**.
5. Also include any **open historical issues** from §2 that haven't been addressed.
5b. **Cover every blocking-review point (reviewer-agnostic).** Read the report's §2
   "Blocking-review coverage" map. **Every point of a current blocking (`CHANGES_REQUESTED`, not
   superseded) review must get a planned fix** — regardless of which reviewer (human or bot) raised it
   and regardless of the point's severity label. A blocking point may **not** be left to "Skipped
   Findings" on severity grounds alone, because leaving it unaddressed is what keeps the PR's
   `CHANGES_REQUESTED` from clearing. The only legitimate non-fix dispositions for a blocking point are:
   (a) it is genuinely already resolved on the current head (cite where), or (b) it is a false positive
   (justify it) — and either way it must appear explicitly in the plan's "Skipped Findings" table with
   that reason, never omitted. If the report's coverage map is missing or a blocking point has no
   finding, treat that as a report gap: surface it and add a fix for the point yourself rather than
   planning around the omission.
6. For each selected finding, read the actual source file to understand the current code in context.

## Step 3 — Design Fixes

> **Design constraint — don't trade one CI failure for another.** Resolve the project's
> verification suite per `Workflows/_verify.md` before designing, and design fixes that pass it —
> including any complexity thresholds (Step B) and PR-only code-health gates (Step C). Prefer early
> returns and small extracted helpers over deeper nesting, and reuse existing fixtures/constants
> over copied blocks. A fix that adds complexity or duplication will bounce in CI even if it is
> logically correct.

7. For each selected finding, propose a concrete fix. Structure each fix as:

```markdown
### Fix {FIX_NUMBER} — {Brief title} (addresses {FINDING_ID})

**Severity**: {severity from report}
**File**: `{path/to/file.py}` L{start}–L{end}

**Current code**:
```{language}
{exact current code snippet}
```

**Proposed change**:
```{language}
{proposed replacement code}
```

**Rationale**: {Why this change addresses the finding. Reference the suggestion from the report but make a concrete decision here.}

**Test changes**:
- {New test to add, or existing test to modify}
- {File and description}

**Risk assessment**:
- **Could break**: {what downstream code or behavior might be affected}
- **Mitigation**: {how to verify it doesn't break — specific test, manual check}

**Dependencies**: {Does this fix depend on another fix being applied first? e.g., "Apply after Fix 2"}
```

## Step 4 — Organize and Write the Plan

7b. **Blocking-review coverage assertion.** Before writing, walk the report's §2 "Blocking-review
   coverage" map and confirm **every** point resolves to either a planned fix above or a justified row
   in "Skipped Findings" (already-resolved or false-positive only — not "low severity"). No blocking
   point may be silently absent. If you cannot satisfy this for a point, stop and flag it to the user
   rather than writing a plan that under-covers the blocking review.

8. **Group fixes by file** — if multiple fixes touch the same file, group them together so they can be applied efficiently in Phase 4.

9. **Order by dependencies first, then severity** — fixes that other fixes depend on come first; within independent fixes, Critical before High before Medium.

10. Write the plan to `.pr-fix/plan.md`:

```markdown
# Fix Plan — {PR_TITLE}

**PR**: {PR_URL}
**Based on report**: `.pr-fix/report.md`
**Generated**: {TIMESTAMP}
**Total fixes proposed**: {count}

---

## Execution Order

| # | Fix | File | Severity | Depends on |
|---|-----|------|----------|------------|
| 1 | {title} | `{file}` | High | — |
| 2 | {title} | `{file}` | Medium | Fix 1 |
| ... | ... | ... | ... | ... |

---

## Fixes by File

### `{path/to/file1.py}`

{Fix 1 block}

{Fix 3 block — if also in this file}

---

### `{path/to/file2.py}`

{Fix 2 block}

---

## Skipped Findings

These findings from the report were rated Low/Info or determined to be non-actionable:

| Finding | Reason skipped |
|---------|---------------|
| F-05 | Low severity, cosmetic |
| F-07 | Info-level observation, no code change needed |

---

## Test Strategy

Summary of all test changes across the fixes:
- {test change 1}
- {test change 2}

Verification after implementation: {the suite resolved per `Workflows/_verify.md` — list each
command and its source (override / CI / project instructions / tool config), so Phases 4 and 5
run the same checks}.
```

## Step 5 — Present and Hand Off

11. Present a summary to the user:
    - Number of fixes proposed
    - Breakdown by severity
    - Files that will be modified
    - Any findings that were intentionally skipped and why

12. Compose the `review-plan` handoff prompt per `Workflows/_handoff.md` — it feeds a fresh-eyes
    review, so follow that file's attention-not-conclusions rule. Then tell the user:
    > Plan written to `.pr-fix/plan.md`.
    >
    > **Next step**: Open a **fresh Claude Code chat** and paste the prompt below. A fresh session
    > will cross-check the plan against the actual code.

    ```
    /pr-fix review-plan {BRIEF}
    ```

    > **After the review**, come back to **this chat** and paste the `/pr-fix implement` prompt the
    > review hands you. This chat retains context from planning, which helps during implementation.

## Error Handling

- If report.md is empty or malformed: tell user to re-run Phase 1
- If a referenced file doesn't exist in the working tree: note it as a discrepancy (PR may need to be checked out locally)
- If the report has no Medium+ findings: inform user that no fixes are needed, write a plan.md that says "No actionable findings"
