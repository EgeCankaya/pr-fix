# Phase 2 — Fix Plan

Read the issue report from Phase 1 and create a concrete, actionable fix plan. This phase converts
advisory suggestions into specific code changes.

Steps below are named, not numbered.

## Execution context

This phase runs **in the session that will later implement** — a chat the user keeps, or the
orchestrator's own session under `/pr-fix run`. That is deliberate: the planner implements its own
plan in Phase 4, and the reasoning behind each fix survives in context.

## Prerequisites

- `.pr-fix/state.json` and `.pr-fix/report.md` must exist (written by Phase 1)

If either is missing:
> Missing prerequisite files. Run `/pr-fix report #PR` first. `/pr-fix status` shows where the
> pipeline currently stands.

## Step: Load context

If text followed `plan` in `$ARGUMENTS`, that's the handoff brief from Phase 1 (or from Phase 5 on
a re-plan) — read it first, per `_handoff.md` § Receiving a brief.

Resolve the GitHub access path per `_github.md`, then run the staleness check per `_staleness.md`.
If the PR has advanced since Phase 1 pinned it, planning against the old report wastes the work —
warn and recommend `/pr-fix report` before continuing.

Read, in order:

1. `.pr-fix/state.json` — PR metadata, config, findings, blocking coverage
2. `.pr-fix/report.md` — the full report, including the reasoning behind each finding
3. `CLAUDE.md` (plus `AGENTS.md` / `CONTRIBUTING.md`) — conventions, coding standards, test patterns

## Step: Resolve and persist the verification suite

Resolve the project's verification suite per `_verify.md` § Resolve the suite, **now, before
designing any fix**. Write it to `.pr-fix/verify-resolved.md` and set `state.json` →
`gates.resolved_from`.

Phase 4 reuses this file rather than re-deriving the suite in a different session; Phase 5
deliberately re-resolves from scratch and diffs against it. Resolving here also means you design
fixes against the gates they must pass, rather than discovering them at implementation time.

## Step: Triage findings

Select every finding rated at or above `config.min_severity` (default: Medium). Add every **open
historical issue** from the report's §2 that hasn't been addressed. Drop anything whose category is
in `config.skip_categories`. If `config.include_own_findings` is `false`, plan only reviewer-raised
findings and blocking points.

### Cover every blocking-review point (reviewer-agnostic)

Walk `state.json` → `blocking_coverage[]`. **Every point of a current blocking review must get a
planned fix** — regardless of which reviewer raised it (human or bot) and regardless of its
severity label.

A blocking point may **not** be dropped to "Skipped Findings" on severity grounds. Leaving it
unaddressed is exactly what keeps the PR's `CHANGES_REQUESTED` from clearing, which is the thing
this pipeline exists to do. The only legitimate non-fix dispositions are:

- **`already-resolved`** — genuinely fixed on the current head. Cite where.
- **`false-positive`** — justify it.

Either way it appears explicitly in the plan's "Skipped Findings" table with that reason — never
omitted. If the coverage map is missing, or a blocking point has no finding, treat that as a report
gap: surface it and add a fix for the point yourself rather than planning around the omission.

For each selected finding, read the actual source file to understand the current code in context.

## Step: Design fixes

> **Design constraint — don't trade one CI failure for another.** Design against the suite you just
> resolved, including complexity thresholds (Step B) and PR-only code-health gates (Step C). Prefer
> early returns and small extracted helpers over deeper nesting, and reuse existing
> fixtures/constants over copied blocks. A fix that adds complexity or duplication will bounce in
> CI even when it is logically correct.

Structure each fix as:

````markdown
### FIX-{NN} — {Brief title} (addresses {FINDING_ID})

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

**Rationale**: {Why this addresses the finding. Reference the report's suggestion, but make a
concrete decision here.}

**Test changes**:
- {New test to add, or existing test to modify — file and description}

**Risk assessment**:
- **Could break**: {what downstream code or behavior might be affected}
- **Mitigation**: {how to verify — specific test, manual check}

**Dependencies**: {Does this depend on another fix landing first? e.g. "After FIX-02"}
````

Mirror each fix into `state.json` → `fixes[]` with its `id`, `addresses`, `file`, `severity`, and
`depends_on`. Use `FIX-NN` IDs consistently in both places — Phases 3, 4 and 5 key off them, and
the per-fix patch files in Phase 4 are named from them.

## Step: Assert coverage, then write the plan

Before writing, walk `blocking_coverage[]` once more and confirm **every** point resolves to either
a planned fix or a justified row in "Skipped Findings" (`already-resolved` or `false-positive`
only). If you cannot satisfy this for a point, stop and flag it rather than writing a plan that
under-covers the blocking review.

**Group fixes by file** so Phase 4 can apply them efficiently. **Order by dependencies first, then
severity** — fixes others depend on come first; within independent fixes, Critical before High
before Medium.

Write `.pr-fix/plan.md`:

````markdown
# Fix Plan — {PR_TITLE}

**PR**: {PR_URL}
**Based on report**: `.pr-fix/report.md`
**Generated**: {TIMESTAMP}
**Total fixes proposed**: {count}
**Blocking points covered**: {n} fixed, {m} justified skips, 0 unmapped

---

## Execution Order

| # | Fix | File | Severity | Depends on |
|---|-----|------|----------|------------|
| FIX-01 | {title} | `{file}` | High | — |
| FIX-02 | {title} | `{file}` | Medium | FIX-01 |

---

## Fixes by File

### `{path/to/file1.py}`

{FIX-01 block}
{FIX-03 block — if also in this file}

---

## Skipped Findings

| Finding | Blocking? | Reason skipped |
|---------|-----------|---------------|
| F-05 | no | Low severity, cosmetic |
| F-09 | **yes** | already-resolved — fixed on head in `src/x.py:41` |

---

## Test Strategy

Summary of all test changes across the fixes:
- {test change 1}

Verification after implementation: see `.pr-fix/verify-resolved.md` (resolved from
{override / CI + project instructions + tool config}). Phases 4 and 5 run that same suite.
````

Set `state.json` → `phases.plan` to complete.

## Step: Present and hand off

Summarize: fixes proposed, breakdown by severity, files to be modified, findings intentionally
skipped and why (call out any blocking point among them explicitly).

Compose the `review-plan` brief per `_handoff.md` — it feeds a fresh-eyes review, so follow that
file's attention-not-conclusions rule. In manual mode:

> Plan written to `.pr-fix/plan.md`.
>
> **Next step**: Open a **fresh Claude Code chat** and paste the prompt below. A fresh session will
> cross-check the plan against the actual code.

```
/pr-fix review-plan {BRIEF}
```

> **After the review**, come back to **this chat** and paste the `/pr-fix implement` prompt the
> review hands you. This chat retains the reasoning behind each fix.

Under `/pr-fix run`, the orchestrator spawns the review as a subagent and returns here for Phase 4
automatically.

## Error handling

- **`report.md` empty or malformed** — tell the user to re-run Phase 1.
- **A referenced file doesn't exist in the working tree** — note it as a discrepancy; the PR may
  not be checked out locally.
- **No findings at or above `min_severity`, and no blocking points** — say so and write a `plan.md`
  reading "No actionable findings". Do not invent work to justify the run.
