# Status — Where the pipeline stands

Report the current position of the pipeline and what to run next. Read-only: this command never
writes, never deletes, and never asks for confirmation.

It exists because the pipeline can span several sessions, and "where am I, and which chat should I
be in?" is the most common question a returning user has. Before this command, the only way to see
the artifact list was to start `/pr-fix clean` and back out of it.

## Step: Read the state

```bash
ls -la .pr-fix/ 2>/dev/null
```

If `.pr-fix/` doesn't exist or is empty:

> ℹ️ No pipeline in progress. Start one with `/pr-fix report #PR`, or `/pr-fix run #PR` to drive
> the whole thing from this session.

Stop there.

Otherwise read `.pr-fix/state.json`. If it's missing but `context.json` is present, this is a run
from before the current schema — say so and note that the next phase will migrate it.

If `state.json` exists but won't parse, say so plainly and show what artifacts are on disk. Do not
reconstruct state by guessing from the markdown files; a wrong guess about which phase completed is
worse than an honest "this is corrupt, here's what's on disk."

## Step: Check for drift

Run the staleness check per `_staleness.md` (resolving access per `_github.md` first). Report the
result, but never block on it — this command is informational.

If the live SHA can't be fetched, say the check was skipped and why.

## Step: Report

```
pr-fix — PR #{N}: {TITLE}
{URL}
{HEAD_BRANCH} → {BASE_BRANCH} @ {HEAD_SHA:0:8}

Phase                 Status        When              Artifact
1  report          ✅ complete    2026-09-14 10:02   report.md
2  plan            ✅ complete    2026-09-14 10:19   plan.md
3  review-plan     ✅ complete    2026-09-14 10:41   plan-approved.md
4  implement       ⏳ pending      —                 —
5  review-impl     ⏳ pending      —                 —
6  respond         ⏳ optional     —                 —

Findings: 7 (2 High, 3 Medium, 2 Low)
Fixes:    5 approved, 1 revised, 1 rejected
Blocking: 3 points — 3 covered, 0 unmapped
Tree:     on feature/retry-backoff @ a1b2c3d4 (matches the PR head), clean
PR:       unchanged since the report

▶ Next: /pr-fix implement
  Run it in your Phase 2 chat — the session where you ran /pr-fix plan.
```

Rules for the report:

- **Name the next command and where to run it.** In a pipeline whose whole design is about which
  session holds which context, "run this in your Phase 2 chat" is the most useful line on the
  screen. Phases 1, 3, 5 want fresh context; 2 and 4 want the planning session.
- **Surface anything that will block the next phase** before the user runs it: a dirty tree ahead
  of Phase 4, a PR that advanced, a fetch gap recorded in `github.fetch_gaps`, a gate red at
  baseline.
- **Mark a failed phase clearly**, with the reason from its artifact — e.g.
  `4 implement ❌ failed — no progress after attempt 2`.
- **Show blocking coverage as a number.** `0 unmapped` is the pipeline's core guarantee; if it's
  ever non-zero, say so loudly and name the points.
- **Report unresolved `pending_decisions[]`.** Each is a question a subagent had to default on
  because no human was reachable. An entry still unresolved is a decision nobody actually made —
  name it and offer to put it to the user now, rather than letting the default stand by silence.

## Step: Offer the next step

End with the single most useful action, and no more than two alternatives:

- Phase incomplete → the next command, and where to run it
- Everything complete, verdict ACCEPT → `/pr-fix respond`, or commit directly
- Verdict NEEDS_ITERATION → `/pr-fix implement` in the Phase 2/4 chat
- Verdict REJECT → `/pr-fix plan` in a fresh chat
- Stale artifacts for a different PR → `/pr-fix clean`

Don't list the whole command surface. The user asked where they are, not what exists.
