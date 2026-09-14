# Phase 1 — Issue Report

Generate a thorough, advisory report on a pull request by analyzing its full history (past reviews,
resolved threads, open comments) and the current diff. All findings are framed as **suggestions** —
never prescriptive mandates.

Steps below are named, not numbered. Cross-references use the step's name, so inserting a step
never renumbers a reference.

## Execution context

This phase runs with fresh context — a fresh chat in manual mode, or a subagent under `/pr-fix run`.

If you are a **subagent**, no human is reachable: never call `AskUserQuestion`. Where a step below
asks the user something, record the question in `state.json` → `pending_decisions[]` and lead your
final report with it. The one exception is the stale-artifact wipe in *Step: Gather PR context* — a
subagent must **not** delete anything; it stops and reports instead.

## Input

`$ARGUMENTS` contains a PR reference after the word `report`:

- `report #9` or `report 9`
- `report https://github.com/owner/repo/pull/9`
- optionally followed by `--key=value` overrides per `_config.md`

If no PR reference is provided, ask the user (or, as a subagent, stop and report).

## Step: Resolve access and config

Before touching GitHub, resolve the access path per `_github.md` § Resolve the access path. Record
it in `state.json` → `github`. Every GitHub operation below refers to `_github.md` § Operation map
rather than naming a CLI — this phase must work under `gh`, MCP tools, or degraded `git-only`.

Then resolve run configuration per `_config.md`: defaults, overridden by the target repo's
`.claude/pr-fix/config.md`, overridden by `--key=value` flags in `$ARGUMENTS`.

If the access path is `git-only`, the review history is unreachable. Say so now, prominently — a
run with no review history cannot build the blocking-review coverage map, which is this pipeline's
main guarantee. Continue only if the user accepts a code-review-only run.

## Step: Gather PR context

Parse the PR reference from `$ARGUMENTS` (strip the leading `report` word). If the argument is a
URL, extract owner/repo from it; otherwise get the repo slug per `_github.md`.

Fetch PR metadata per `_github.md` (*PR metadata*), including the head commit SHA — it is the
anchor every later phase uses to detect a PR that moved out from under the report.

**Stale state check.** Before writing anything, check whether `.pr-fix/` already holds artifacts:

```bash
ls .pr-fix/ 2>/dev/null
```

If it exists and contains files, read the PR number from `state.json` (or, for a pre-schema run,
`context.json`):

- **Different PR** — the old artifacts would corrupt this run.
  > ⚠️ **Stale artifacts detected.** `.pr-fix/` holds artifacts from PR #{OLD}, but you're
  > analyzing PR #{NEW}. These will be **deleted**. Back them up now if you need them.
- **Same PR** — this is a re-run.
  > ℹ️ **Re-running Phase 1** for PR #{N}. The report and every downstream artifact (plan, approved
  > plan, changes, verdict) will be overwritten, and those phases will need re-running.

Either way, confirm with the user, then `rm -rf .pr-fix`. **As a subagent, do not delete** — stop
and report that the orchestrator must resolve the stale state first.

Create the directory and keep it out of git without touching any tracked file (the exclude entry is
added once; re-runs are no-ops):

```bash
mkdir -p .pr-fix/patches
git check-ignore -q .pr-fix || echo '.pr-fix/' >> "$(git rev-parse --git-path info/exclude)"
```

Write `.pr-fix/state.json` per `_schema.md`, filling `context`, `github`, `config`, and
`phases.report.status = "running"`. `context.head_sha` is required.

**Anchor the local working tree.** Phases 4 and 5 edit and diff the *local* tree, so it must hold
the PR's code:

```bash
git rev-parse --abbrev-ref HEAD
git rev-parse HEAD
```

If the branch isn't the PR's head branch, or the SHA differs from `context.head_sha`:

> ⚠️ Your working tree is on `{CURRENT_BRANCH}` ({CURRENT_SHA:0:8}), but this PR is
> `{HEAD_BRANCH}` ({HEAD_SHA:0:8}). Phases 4–5 edit and review the local tree, so they need the
> PR's code checked out. Check out per `_github.md` § Checking out the PR (commit or stash local
> work first), or proceed read-only for the report and check out before Phase 4.

The report itself is built from remote data and is valid either way — **warn, never force a
checkout**. The user may have uncommitted work.

## Step: Collect PR history

Fetch all three streams per `_github.md` § Operation map, paginating fully:

- **Line-level review comments** (*Line-level review comments*) — including resolved threads
- **Reviews** (*Reviews*) — approvals, change requests, comment reviews
- **Conversation comments** (*Conversation comments*)

Pagination is not optional. A PR with a long history truncates to the first page by default, and an
under-fetched review list silently shrinks the coverage map below. If any fetch is partial, record
it in `state.json` → `github.fetch_gaps[]` and surface it in the report.

Categorize each piece of feedback:

- 🟢 **Resolved** — raised earlier and since addressed. Note *what* was raised and *how* it was
  resolved; this shows the PR's evolution and what the author already handled.
- 🟡 **Open** — unresolved review comments or threads.
- 🔴 **Requested changes** — from `CHANGES_REQUESTED` reviews not dismissed or superseded.

Look for **patterns**: are reviewers repeatedly flagging the same kind of issue, the same file, the
same approach?

### Itemize every blocking review (reviewer-agnostic)

From the reviews you fetched, find the **most recent `CHANGES_REQUESTED` review per reviewer that
has not been dismissed or superseded by a later approval from that same reviewer**. Those are the
*blocking* reviews, whoever submitted them — a human, or a bot such as a code-health or security
scanner. Do **not** hardcode or privilege any reviewer name.

For each blocking review, **decompose the body into the distinct points it raises** — a single
review often bundles several. Give **each point its own finding ID** (or map it to an existing
finding), so nothing in a multi-point review collapses into one entry and gets lost.

Record each point as an entry in `state.json` → `blocking_coverage[]` with its `review_id`,
`reviewer`, `point` text, and the `finding_ids` covering it. A point you judge already-resolved on
the current head still gets an entry (`disposition: "already-resolved"`, citing where) — never
silently dropped.

If a run-specific focus was supplied (e.g. a flag scoping the run to one reviewer's latest review),
honor it. Absent that, the default is *every* current blocking review.

## Step: Analyze the current diff

Fetch the full PR diff per `_github.md` (*PR diff*).

Read the project's instructions — `CLAUDE.md` in the repo root, plus `AGENTS.md` /
`CONTRIBUTING.md` if present — for conventions, coding standards, and working agreements.

### Reading changed files, within a budget

Reading every changed file in full gives the best analysis and is the default. It is also how this
phase fails on a large PR: context runs out mid-analysis and findings already made are silently
lost from the report.

Count the changed files. If the count is at or under `config.full_read_file_budget` (default 25)
**and** the diff is under roughly 2,000 lines, read every changed file in its entirety.

Above either threshold, triage instead:

1. **Read in full**: files carrying an open or blocking review comment; files whose diff touches
   control flow, error handling, concurrency, auth, or I/O; files under 200 lines.
2. **Read the diff hunks plus surrounding context** (roughly 40 lines each side) for the rest.
3. **Skim**: generated files, lockfiles, fixtures, and pure-formatting changes. Say in the report
   that you skimmed them and why.

Either way, **write findings out as you go** rather than holding them all to the end. Append each
finding to `state.json` → `findings[]` as you confirm it. If context runs short, the report is then
built from a complete list rather than from whatever survived in your head. Record the strategy you
used in the report's §1 so the reader knows the analysis depth.

### What to look for

- **Bugs & logic**: incorrect logic, edge cases, off-by-one errors, null/undefined handling, race conditions
- **Security**: injection, data exposure, auth gaps, hardcoded secrets, SSRF, path traversal
- **Convention violations**: `CLAUDE.md` rules, naming, import ordering, line length, type annotations
- **Test gaps**: new code paths without tests, untested error branches, missing edge-case assertions
- **Performance**: blocking I/O, O(n²) patterns, unnecessary allocations, N+1 queries
- **Code quality**: naming clarity, complexity (deep nesting, long functions), DRY violations, readability
- **Compatibility**: breaking changes to public APIs, contract violations with consuming modules

Skip any category listed in `config.skip_categories`.

## Step: Assert blocking coverage

Before writing the report, walk `state.json` → `blocking_coverage[]` and verify **every point of
every current blocking review maps to at least one finding ID**. If a point is unmapped, add a
finding for it now.

This matters because Phase 2 plans only what the report lists: a point missing here is a point that
never gets fixed, and the `CHANGES_REQUESTED` never clears. The check is a loop over a list of
objects, not a re-reading of prose — that is why `blocking_coverage[]` exists.

## Step: Write the report

Write `.pr-fix/report.md`:

````markdown
# PR Issue Report — {PR_TITLE}

**PR**: {PR_URL}
**Branch**: `{HEAD_BRANCH}` → `{BASE_BRANCH}`
**Stats**: {CHANGED_FILES} files, +{ADDITIONS}/−{DELETIONS}
**Generated**: {TIMESTAMP}
**Analysis depth**: {full read of all changed files | triaged — N read in full, M by hunk, K skimmed}
**Data completeness**: {complete | degraded — see caveats}

---

## §1 — PR Overview

{Brief description of what the PR does, based on the PR body and diff analysis.}

{If `github.fetch_gaps` is non-empty or the access path was `git-only`, state plainly here what
data is missing and what that means for the findings below.}

### Changed Files

| File | Changes | Read as | Summary |
|------|---------|---------|---------|
| `path/to/file.py` | +20/−5 | full | {one-line summary} |

---

## §2 — Historical Context

### Resolved Issues (Past Feedback)

| # | Reviewer | What was raised | How it was resolved |
|---|----------|----------------|-------------------|
| H-01 | @reviewer | {description} | {resolution} |

### Open Review Comments

| # | Reviewer | File:Line | Comment | Status |
|---|----------|-----------|---------|--------|
| H-02 | @reviewer | `file.py:42` | {comment} | 🟡 Open |

### Pending Change Requests

| Reviewer | Date | Summary |
|----------|------|---------|
| @reviewer | {date} | {key concerns} |

### Blocking-review coverage

Every distinct point of each current blocking (`CHANGES_REQUESTED`, not superseded) review, mapped
to the finding ID(s) covering it. Mirrors `state.json` → `blocking_coverage[]`. Phase 2 plans a fix
for each; Phase 3 re-derives this map from the live reviews rather than trusting it.

| Review (id, reviewer) | Point | Finding ID(s) | Disposition |
|-----------------------|-------|---------------|-------------|
| {review_id}, @reviewer | {point 1} | F-01 | fix |
| {review_id}, @reviewer | {point 2} | F-02 | already-resolved — fixed in {sha} |

### Recurring Themes

{Patterns? Repeated concerns? 1–3 bullets.}

---

## §3 — New Findings

The following are observations from analyzing the current state of the PR. These are advisory —
consider each on its merits.

### F-01: {Brief title}

- **File**: `path/to/file.py` L42–L55
- **Severity**: Medium
- **Category**: Bug
- **What I noticed**: {The concern, and *why* it matters.}
- **Suggestion**: Consider {approach}. One option would be {alternative}.
- **Confidence**: High

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
**Blocking points covered**: {n} of {n}
**Files with most findings**: `{file1}` ({n}), `{file2}` ({n})

### Overall Impression

{1–2 balanced paragraphs. Acknowledge what's done well. Note the key areas worth attention. Keep
the tone constructive.}
````

> **Tone guidance.** Every finding in §3 uses advisory language:
> - ✅ *"Consider adding a bounds check here…"* / *"You might want to handle the case where…"*
> - ❌ *"You must change this to…"* / *"This is wrong, fix it by…"*
>
> The report informs; Phase 2 is where suggestions become actionable items if the developer agrees.
> The one exception is a Critical or High **security** finding: state the risk plainly. Hedging a
> vulnerability into "you might consider" misrepresents its severity.

Then set `state.json` → `phases.report` to complete, with the head SHA you observed.

## Step: Present and hand off

Summarize for the user: historical items found (resolved + open), new findings by severity, the top
3 findings as one-liners, and any data gaps.

Compose the `plan` brief per `_handoff.md`. In manual mode:

> Report written to `.pr-fix/report.md`.
>
> **Next step**: Open a **fresh Claude Code chat** and paste the prompt below. That chat will read
> this report and create a concrete fix plan.

```
/pr-fix plan {BRIEF}
```

As a subagent, return the brief in your final report instead — the orchestrator carries it forward.

## Error handling

- **Access path unresolvable** — say which of `gh` / MCP / git was tried and what failed. Don't guess.
- **PR doesn't exist** — report it clearly.
- **A fetch fails partway (rate limit)** — record in `github.fetch_gaps[]`, continue with what you
  have, and flag the gap in §1. Never treat missing data as evidence of absence.
