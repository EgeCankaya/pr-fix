---
name: pr-fix
description: Multi-phase PR review-to-fix pipeline with fresh-context isolation between phases. USE WHEN "pr-fix", "fix PR", "review and fix PR", "PR report", "PR plan", "implement fixes", "review implementation", "address review comments", "clear a changes-requested review".
---

# PRFix

A pipeline that reviews a GitHub pull request, plans fixes, implements them, and reports back to
the reviewers — with human checkpoints and fresh-context isolation between phases.
`.pr-fix/` acts as the message bus. Requires `git` and a way to reach GitHub (see § Requirements).

## Two ways to run it

**Orchestrated** — one session drives everything, spawning subagents where fresh eyes matter:

```
/pr-fix run #9
```

**Step by step** — each phase is its own command, so you can read and edit the artifacts between
phases:

```
Phase 1 (report) → Phase 2 (plan) → Phase 3 (review-plan) → Phase 4 (implement) → Phase 5 (review-impl) → Phase 6 (respond)
  fresh context     your chat         fresh context           your chat             fresh context          your chat
```

- **Phases 1, 3, 5** run with **fresh context** — a subagent under `run`, or a fresh chat in manual
  mode. A reviewer carrying the author's reasoning can't independently catch the author's mistakes.
- **Phases 2, 4, 6** run in **one retained session**, so the planner implements its own plan.
- In manual mode every phase ends with a **ready-to-paste handoff prompt**: the next command plus a
  ~10-sentence brief carrying what that session knows and the `.pr-fix/` files don't — user
  decisions, doubts, couplings. See `references/_handoff.md`.

## Workflow routing

Parse the first word of `$ARGUMENTS` and execute the matching workflow file. Trailing
`--key=value` flags are config overrides (`references/_config.md`). For every subcommand except
`report` and `run`, remaining text is an optional **handoff brief** — read it per
`references/_handoff.md` § Receiving a brief.

| Argument | Workflow | Phase | Description |
|----------|----------|-------|-------------|
| `run` | `references/run.md` | 1–5 | Drive the whole pipeline from this session, with subagents for 1/3/5 |
| `report` | `references/report.md` | 1 | Analyze PR history + diff → advisory issue report |
| `plan` | `references/plan.md` | 2 | Read report → concrete fix plan |
| `review-plan` | `references/review-plan.md` | 3 | Fresh-eyes cross-check of the plan |
| `implement` | `references/implement.md` | 4 | Apply approved fixes → run the project's checks |
| `review-impl` | `references/review-impl.md` | 5 | Fresh-eyes review of the implementation |
| `respond` | `references/respond.md` | 6 | Draft the commit message + reviewer reply (optional) |
| `status` | `references/status.md` | — | Where the pipeline stands and what to run next (read-only) |
| `clean` | `references/clean.md` | — | Wipe `.pr-fix/` to start fresh |

If `$ARGUMENTS` is empty or unrecognized, show this help:

```
/pr-fix — Multi-phase PR review-to-fix pipeline

Usage:
  /pr-fix run #PR          — Drive all five phases from this session (recommended)

  /pr-fix report #PR       — Phase 1: issue report            (fresh chat)
  /pr-fix plan [brief]     — Phase 2: fix plan                (keep this chat)
  /pr-fix review-plan …    — Phase 3: review the plan         (fresh chat)
  /pr-fix implement …      — Phase 4: apply fixes + checks    (Phase 2 chat)
  /pr-fix review-impl …    — Phase 5: review the diff         (fresh chat)
  /pr-fix respond [brief]  — Phase 6: commit msg + PR reply   (optional)

  /pr-fix status           — Where am I, and what runs next?
  /pr-fix clean            — Wipe .pr-fix/ and start over

Flags (any subcommand): --min-severity=high --skip-categories=performance
                        --include-own-findings=false --auto-checkpoints=false

Nothing is ever committed or pushed for you.
```

## Handoff files

All phases communicate through `.pr-fix/` in the repo root (Phase 1 keeps it out of git):

| File | Written by | Read by | Content |
|------|-----------|---------|---------|
| `state.json` | All | All | The machine-readable state: context, findings, blocking coverage, fixes, gates. See `references/_schema.md` |
| `report.md` | Phase 1 | 2, 3 | Advisory issue report |
| `plan.md` | Phase 2 | 3 | Concrete fix plan |
| `verify-resolved.md` | Phase 2 | 4, 5 | The resolved verification suite |
| `plan-approved.md` | Phase 3 | 4, 5 | Approved/revised fixes |
| `changes.md` | Phase 4 | 5 | Implementation summary, carrying PASSED/FAILED |
| `patches/fix-NN.patch` | Phase 4 | 5 | One incremental patch per fix, so a single fix can be reverted |
| `verdict.md` | Phase 5 | 6, user | Final verdict |
| `response.md` | Phase 6 | user | Draft commit message + PR comment |

## Shared references

Five files hold the cross-cutting contracts, so the phase workflows reference them instead of
restating (and drifting from) the rules:

| File | What it governs |
|------|-----------------|
| `references/_github.md` | How to reach GitHub: `gh`, MCP tools, or degraded `git-only`. Every phase resolves this first — **never hardcode `gh`** |
| `references/_verify.md` | How the project's own checks are found, persisted, and run; failure signatures; the pre-implementation baseline |
| `references/_staleness.md` | The PR-moved check every phase runs before doing anything |
| `references/_schema.md` | The `state.json` schema and how to update it safely |
| `references/_handoff.md` | The handoff brief: format, content, and how it's delivered to a chat or a subagent |
| `references/_config.md` | Run configuration: severity floor, category skips, budgets, and the repo override file |

## Guarantees

- **Every blocking-review point is covered.** Phase 1 decomposes each open `CHANGES_REQUESTED`
  review — human or bot, no privileged reviewer names — into its distinct points and maps each to a
  finding. Phase 2 plans one for each; Phase 3 re-derives the map from the live reviews rather than
  trusting it. A point may only be skipped as *already-resolved* or *false-positive*, never on
  severity grounds — that's the thing keeping the review from clearing.
- **Pre-existing failures are a fact, not a judgment.** Phase 4 runs the gates once before touching
  anything, so "that was already broken" is a set difference both it and Phase 5 can compute.
- **A single fix can be reverted.** Each fix gets its own patch, so Phase 5 reverses one with
  `git apply -R` instead of hand-editing a file.
- **Nothing is committed or pushed.** Phase 6 drafts; the user decides.

## Examples

**Orchestrated, the common case**
```
User: /pr-fix run #9
→ Subagent writes .pr-fix/report.md; relays findings and blocking coverage
→ Checkpoint: confirm scope
→ Plans fixes in this session
→ Subagent cross-checks the plan against the real code
→ Checkpoint: approve the fix set (never skipped — last stop before editing)
→ Applies fixes here, runs the project's checks, stops on no-progress
→ Subagent audits the diff and re-runs the gates independently
→ Checkpoint: keep or revert
→ Offers /pr-fix respond
```

**Step by step**
```
User: /pr-fix report #9            → writes report.md, prints a plan prompt
User: /pr-fix plan <brief>         → writes plan.md, prints a review-plan prompt
User: /pr-fix review-plan <brief>  (fresh chat) → writes plan-approved.md
User: /pr-fix implement <brief>    (back in the planning chat) → writes changes.md
User: /pr-fix review-impl <brief>  (fresh chat) → writes verdict.md
```

**Coming back to a half-finished run**
```
User: /pr-fix status
→ PR #9, phases 1–3 complete, tree clean and on the PR head, PR unchanged
→ ▶ Next: /pr-fix implement — run it in your Phase 2 chat
```

A complete worked run — real report, plan, verdict and `state.json` — is in `examples/`.
