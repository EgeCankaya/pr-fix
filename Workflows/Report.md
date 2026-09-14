# Phase 1 — Issue Report

Generate a thorough, advisory report on a pull request by analyzing its full history (past reviews, resolved threads, open comments) and the current diff. All findings are framed as **suggestions** — never prescriptive mandates.

## Input

`$ARGUMENTS` should contain a PR reference after the word `report`:
- `report #9` or `report 9`
- `report https://github.com/owner/repo/pull/9`

If no PR reference is provided, ask the user.

## Step 1 — Gather PR Context

1. Parse the PR reference from `$ARGUMENTS` (strip the leading `report` word).

2. Determine the repo. If the argument is a URL, extract owner/repo. Otherwise, detect from the current git remote:
   ```bash
   OWNER_REPO=$(gh repo view --json nameWithOwner -q '.nameWithOwner')
   ```

3. Fetch PR metadata (including `headRefOid` — the PR's head commit SHA, used by every later phase to detect a PR that advanced after the report was written):
   ```bash
   gh pr view {PR_NUMBER} --json title,body,baseRefName,headRefName,headRefOid,state,number,url,reviewDecision,additions,deletions,changedFiles,files
   ```

4. **Stale state check** — before writing anything, check if `.pr-fix/` already has artifacts from a previous run:
   ```bash
   ls .pr-fix/ 2>/dev/null
   ```

   If `.pr-fix/` exists and contains files:
   - Check if `context.json` exists and read the PR number from it
   - If the previous PR number **differs** from the current one:
     > ⚠️ **Stale artifacts detected.** `.pr-fix/` contains artifacts from PR #{OLD_NUMBER}, but you're analyzing PR #{NEW_NUMBER}.
     >
     > These old files will be **deleted** to prevent state corruption.
     > If you need to keep them, abort now and back them up first.
     >
     > Proceeding will wipe `.pr-fix/` and start fresh.
     
     Ask the user to confirm, then wipe:
     ```bash
     rm -rf .pr-fix
     ```
   
   - If the previous PR number is the **same** (re-running Phase 1 on the same PR):
     > ℹ️ **Re-running Phase 1** for PR #{NUMBER}. Previous report and any downstream artifacts (plan, approved plan, etc.) will be overwritten.
     >
     > Downstream phases will need to be re-run after this.
     
     Ask the user to confirm, then wipe:
     ```bash
     rm -rf .pr-fix
     ```

5. Create the `.pr-fix/` directory and keep it out of git without touching any tracked file (the
   exclude entry is added once; re-runs are no-ops):
   ```bash
   mkdir -p .pr-fix
   git check-ignore -q .pr-fix || echo '.pr-fix/' >> "$(git rev-parse --git-path info/exclude)"
   ```

6. Write `.pr-fix/context.json` with the PR metadata. **`head_sha` is required** — it is the anchor every later phase uses to detect whether the PR moved out from under the report:
   ```json
   {
     "pr_number": 9,
     "repo": "owner/repo",
     "url": "https://github.com/owner/repo/pull/9",
     "title": "PR title",
     "base_branch": "main",
     "head_branch": "feature/retry-backoff",
     "head_sha": "<headRefOid from step 3>",
     "state": "OPEN",
     "review_decision": "REVIEW_REQUIRED"
   }
   ```

7. **Anchor the local working tree to the PR.** The later phases (4, 5) edit and diff the *local* tree, so it must actually contain the PR's code. Check what's checked out:
   ```bash
   git rev-parse --abbrev-ref HEAD   # current branch
   git rev-parse HEAD                # current head SHA
   ```
   - If the current branch is **not** `{HEAD_BRANCH}` or the SHA differs from `head_sha`, tell the user the working tree doesn't match the PR and offer to check it out:
     > ⚠️ Your working tree is on `{CURRENT_BRANCH}` ({CURRENT_SHA:0:8}), but this PR is `{HEAD_BRANCH}` ({HEAD_SHA:0:8}). Phases 4–5 edit and review the local tree, so they need the PR's code checked out.
     >
     > Run `gh pr checkout {PR_NUMBER}` (commit/stash any local work first), or proceed read-only for the report and check out before Phase 4.
   - The report itself works off `gh` (remote) data and is safe to generate either way — only warn; do not force a checkout.

## Step 2 — Collect PR History (Past + Current Issues)

6. Fetch **all review comments** (line-level comments, including resolved threads):
   ```bash
   gh api repos/{OWNER}/{REPO}/pulls/{PR_NUMBER}/comments --paginate
   ```

7. Fetch **all PR reviews** (approval/change-request/comment reviews):
   ```bash
   gh api repos/{OWNER}/{REPO}/pulls/{PR_NUMBER}/reviews --paginate
   ```

8. Fetch **top-level conversation comments**:
   ```bash
   gh pr view {PR_NUMBER} --json comments -q '.comments'
   ```

9. Categorize each piece of feedback:
   - 🟢 **Resolved** — issues that were raised and subsequently addressed. Note *what* was raised and *how* it was resolved — this shows the PR's evolution and what the author already handled.
   - 🟡 **Open** — unresolved review comments or threads that still need attention.
   - 🔴 **Requested Changes** — from reviews with state `CHANGES_REQUESTED` that haven't been dismissed or superseded by an approval.

10. Look for **patterns**: Are reviewers repeatedly flagging the same type of issue? Are there recurring concerns about a specific file or approach?

10b. **Identify the latest blocking review(s) and itemize every point (reviewer-agnostic).** From the
   reviews fetched in Step 7, find the **most recent `CHANGES_REQUESTED` review per reviewer that has
   not been dismissed or superseded by a later approval from the same reviewer** — these are the
   *blocking* reviews, whoever submitted them (a human, or a bot such as a code-health or security scanner).
   Do **not** hardcode or privilege any reviewer name. If a run-specific focus was supplied (e.g. a
   `report_focus` instruction captured into `context.json` scoping the run to one reviewer's latest
   review), honor it — but the default, absent such an instruction, is *every* current blocking review.

   For each blocking review, **decompose its body into the distinct points it raises** (a single
   review often bundles several). Give **each distinct point its own finding ID** in §3 (or map it to
   an existing finding) so nothing in a multi-point review is collapsed into one entry and lost. Build
   a small map you will record in the report — `{review_id → [finding IDs covering its points]}` — so
   Step 4 can prove the report covers the whole blocking surface. A point you judge already-resolved on
   the current head still gets listed (as a Resolved item with the finding it was checked against), not
   silently dropped.

## Step 3 — Analyze the Current Diff

11. Fetch the full PR diff:
    ```bash
    gh pr diff {PR_NUMBER}
    ```

12. Read **every changed file** in its entirety (not just the diff hunks) to understand surrounding context. Use the files list from Step 1.

13. Read the project's instructions — `CLAUDE.md` in the repo root, plus `AGENTS.md` / `CONTRIBUTING.md` if present — for conventions, coding standards, and working agreements.

14. Perform a thorough analysis of the changes. Check for:
    - **Bugs & Logic**: incorrect logic, edge cases, off-by-one errors, null/undefined handling, race conditions
    - **Security**: injection, data exposure, auth gaps, hardcoded secrets, SSRF, path traversal
    - **Convention violations**: `CLAUDE.md` rules, naming conventions, import ordering, line length, type annotations
    - **Test gaps**: new code paths without tests, untested error branches, missing edge-case assertions
    - **Performance**: blocking I/O, O(n²) patterns, unnecessary allocations, N+1 queries
    - **Code quality**: naming clarity, complexity (deep nesting, long functions), DRY violations, readability
    - **Compatibility**: breaking changes to public APIs, contract violations with consuming modules

## Step 4 — Write the Report

14b. **Coverage assertion before writing (blocking reviews).** Using the `{review_id → [finding IDs]}`
   map from Step 10b, verify that **every distinct point of every current blocking review maps to at
   least one finding ID** in the report (a §3 New Finding, or a §2 Resolved/Open row with its finding
   ID). If any point is unmapped, add a finding for it now — the report must not under-capture a
   multi-point blocking review, because Phase 2 plans only what the report lists. Record the map in §2
   (see the "Blocking-review coverage" block in the template) so Phase 2 and Phase 3 can re-check it.

15. Write `.pr-fix/report.md` with the following structure:

---

```markdown
# PR Issue Report — {PR_TITLE}

**PR**: {PR_URL}
**Branch**: `{HEAD_BRANCH}` → `{BASE_BRANCH}`
**Stats**: {CHANGED_FILES} files, +{ADDITIONS}/−{DELETIONS}
**Generated**: {TIMESTAMP}

---

## §1 — PR Overview

{Brief description of what the PR does, based on the PR body and diff analysis.}

### Changed Files

| File | Changes | Summary |
|------|---------|---------|
| `path/to/file.py` | +20/−5 | {one-line summary} |
| ... | ... | ... |

---

## §2 — Historical Context

### Resolved Issues (Past Feedback)

These issues were raised in earlier reviews and have been addressed:

| # | Reviewer | What was raised | How it was resolved |
|---|----------|----------------|-------------------|
| H-01 | @reviewer | {description} | {resolution} |

### Open Review Comments

These review comments are currently unresolved:

| # | Reviewer | File:Line | Comment | Status |
|---|----------|-----------|---------|--------|
| H-02 | @reviewer | `file.py:42` | {comment} | 🟡 Open |

### Pending Change Requests

| Reviewer | Date | Summary |
|----------|------|---------|
| @reviewer | {date} | {key concerns from the review} |

### Blocking-review coverage

Every distinct point of each current blocking (`CHANGES_REQUESTED`, not superseded) review, mapped to
the finding ID(s) that cover it. Phase 2 plans a fix for each, and Phase 3 re-checks this map.

| Review (id, reviewer) | Point | Finding ID(s) | Status |
|-----------------------|-------|---------------|--------|
| {review_id}, @reviewer | {point 1} | F-01 | 🔴 Open |
| {review_id}, @reviewer | {point 2} | F-02 | 🔴 Open |

### Recurring Themes

{Are there patterns? Repeated concerns? Summarize in 1–3 bullets.}

---

## §3 — New Findings

{Preamble: "The following are observations and suggestions from analyzing the current state of the PR. These are advisory — consider each on its merits."}

### F-01: {Brief title}

- **File**: `path/to/file.py` L42–L55
- **Severity**: Medium
- **Category**: Bug
- **What I noticed**: {Clear description of the concern, explaining *why* it matters.}
- **Suggestion**: Consider {approach}. One option would be {alternative}. You might also want to {another option}.
- **Confidence**: High

### F-02: {Brief title}

...

---

## §4 — Summary Dashboard

| Severity | Count |
|----------|-------|
| Critical | 0 |
| High | 1 |
| Medium | 3 |
| Low | 2 |
| Info | 1 |

**Open historical issues**: {count}
**Files with most findings**: `{file1}` ({n}), `{file2}` ({n})

### Overall Impression

{1–2 paragraphs giving a balanced overall assessment. Acknowledge what's done well. Note the key areas that would benefit from attention. Keep the tone constructive and collaborative.}
```

---

> **IMPORTANT — Tone guidance**: Every finding in §3 must use advisory language:
> - ✅ *"Consider adding a bounds check here..."*
> - ✅ *"You might want to handle the case where..."*
> - ✅ *"One option would be to extract this into..."*
> - ❌ *"You must change this to..."*
> - ❌ *"This is wrong, fix it by..."*
> - ❌ *"Change line 42 to..."*
>
> The report informs; Phase 2 (planning) is where suggestions become actionable items if the developer agrees.

## Step 5 — Present and Hand Off

16. After writing the report, present a brief summary to the user:
    - How many historical items found (resolved + open)
    - How many new findings by severity
    - Top 3 most important findings (brief one-liners)

17. Compose the `plan` handoff prompt per `Workflows/_handoff.md`, then tell the user:
    > Report written to `.pr-fix/report.md`.
    >
    > **Next step**: Open a **fresh Claude Code chat** and paste the prompt below. That chat will
    > read this report and create a concrete fix plan.

    ```
    /pr-fix plan {BRIEF}
    ```

## Error Handling

- If `gh` is not authenticated: tell user to run `gh auth login`
- If PR doesn't exist: report the error clearly
- If a `gh api` call fails (e.g., rate limit): note the failure, continue with what's available, and flag in the report what data was missing
