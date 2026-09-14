---
name: pr-fix
description: Multi-phase PR review-to-fix pipeline with fresh-chat isolation. USE WHEN "pr-fix", "fix PR", "review and fix PR", "PR report", "PR plan", "implement fixes", "review implementation".
---

# PRFix

A five-phase pipeline that reviews a GitHub pull request, plans fixes, and implements them — with human checkpoints and fresh-chat isolation between phases. The file system (`.pr-fix/` directory) acts as the message bus between chat sessions. Requires `git` and an authenticated `gh` CLI.

## Pipeline Overview

```
Phase 1 (report)  →  Phase 2 (plan)  →  Phase 3 (review-plan)  →  Phase 4 (implement)  →  Phase 5 (review-impl)
   Fresh Chat A        Fresh Chat B          Fresh Chat C            Chat B (resumed)         Fresh Chat D
```

- **Phases 1, 3, 5** run in **fresh** Claude Code chats for independent analysis
- **Phases 2 and 4** share the **same** chat so the planner retains context when implementing
- Every phase ends with a **ready-to-paste handoff prompt** for the next one: the command plus a ~10-sentence brief carrying what that chat knows and the `.pr-fix/` files don't (user decisions, doubts, couplings). See `Workflows/_handoff.md`.

## Workflow Routing

Parse the first word of `$ARGUMENTS` and execute the matching workflow file. For every subcommand except `report`, any text after the first word is an optional **handoff brief** from the previous phase — the workflow reads it per `Workflows/_handoff.md` § Receiving a brief.

| Argument | Workflow | Phase | Description |
|----------|----------|-------|-------------|
| `report` | `Workflows/Report.md` | 1 | Analyze PR history + diff → thorough issue report with suggestions |
| `plan` | `Workflows/Plan.md` | 2 | Read report → concrete fix plan |
| `review-plan` | `Workflows/ReviewPlan.md` | 3 | Fresh-eyes cross-check of the plan |
| `implement` | `Workflows/Implement.md` | 4 | Apply approved fixes → run the project's checks |
| `review-impl` | `Workflows/ReviewImpl.md` | 5 | Fresh-eyes review of implementation |
| `clean` | `Workflows/Clean.md` | — | Wipe `.pr-fix/` to start fresh |

If `$ARGUMENTS` is empty or unrecognized, show this help:

```
/pr-fix — Multi-phase PR review-to-fix pipeline

Usage:
  /pr-fix report [#PR or URL]   — Phase 1: Generate issue report (fresh chat)
  /pr-fix plan [brief]          — Phase 2: Create fix plan (fresh chat)
  /pr-fix review-plan [brief]   — Phase 3: Review the plan (fresh chat)
  /pr-fix implement [brief]     — Phase 4: Implement fixes (same chat as Phase 2)
  /pr-fix review-impl [brief]   — Phase 5: Review implementation (fresh chat)
  /pr-fix clean                 — Wipe .pr-fix/ to start a fresh pipeline run

Handoff files are stored in .pr-fix/ (Phase 1 keeps it out of git).
Run phases in order. Phases 1, 3, 5 should each be a fresh chat.
Phases 2 and 4 should share the same chat session.
Each phase ends with a ready-to-paste prompt for the next one (command + brief).

Note: Phase 1 auto-detects stale artifacts and prompts before overwriting.
      Use /pr-fix clean to manually wipe state between pipeline runs.
```

## Handoff Files

All phases communicate through `.pr-fix/` in the repo root:

| File | Written by | Read by | Content |
|------|-----------|---------|---------|
| `context.json` | Phase 1 | All | PR number, repo, branches, **`head_sha`** (staleness anchor) |
| `report.md` | Phase 1 | Phase 2, 3 | Thorough issue report with suggestions |
| `plan.md` | Phase 2 | Phase 3 | Concrete fix plan |
| `plan-approved.md` | Phase 3 | Phase 4 | Approved/revised fixes |
| `baseline_sha.txt` | Phase 4 | Phase 4, 5 | Pre-implementation commit SHA — all fix diffs/reverts are computed against it |
| `changes.md` | Phase 4 | Phase 5 | Implementation summary |
| `verdict.md` | Phase 5 | User | Final verdict |

`Workflows/_verify.md` is the shared verification procedure that Phases 2, 4, and 5 reference
instead of duplicating commands. It finds the target project's own checks (a
`.claude/pr-fix/verify.md` override, or else CI, project instructions, and tool config) and sorts
them into auto-fixers, gates, and PR-only code-health gates.

`Workflows/_handoff.md` is the shared contract for the handoff prompt every phase ends with —
format, length, what goes in, and how the receiving phase reads it.

## Examples

**Example 1: Start the pipeline on a PR**
```
User: /pr-fix report #9
→ Fetches PR #9 history and diff
→ Writes .pr-fix/report.md with findings and suggestions
→ Prints a "/pr-fix plan <brief>" prompt to paste into a fresh chat
```

**Example 2: Generate the fix plan**
```
User: /pr-fix plan <brief>
→ Reads the brief, then .pr-fix/report.md
→ Proposes concrete fixes for Medium+ findings
→ Writes .pr-fix/plan.md
→ Prints a "/pr-fix review-plan <brief>" prompt for a fresh chat; come back here for implement
```

**Example 3: Review and implement**
```
User: /pr-fix review-plan <brief>    (in fresh Chat C)
→ Cross-checks plan against actual code
→ Writes .pr-fix/plan-approved.md
→ Prints a "/pr-fix implement <brief>" prompt to paste back in Chat B

User: /pr-fix implement <brief>      (back in Chat B)
→ Applies approved fixes, runs the project's checks
→ Writes .pr-fix/changes.md
→ Prints a "/pr-fix review-impl <brief>" prompt for a fresh chat

User: /pr-fix review-impl <brief>    (in fresh Chat D)
→ Independent review of implementation
→ Writes .pr-fix/verdict.md
```
