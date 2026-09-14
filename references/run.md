# Run — Drive the whole pipeline from one session

Run all five phases from a single session, using **subagents** for the phases that need fresh eyes
and keeping the planning/implementing phases in this session's context.

Steps below are named, not numbered.

## Why this exists

The manual pipeline asks the user to open four chats and hand-carry four briefs between them. The
design insight behind that — fresh context for the reviewing phases, retained context for the
planning and implementing phases — is sound. The delivery was the problem: every paste is a place
the pipeline stalls, and in practice runs get abandoned at the second or third handoff.

A subagent starts cold. That is exactly the isolation the fresh-chat rule was buying, and it costs
the user nothing. So `run` keeps the phase boundaries and the human checkpoints, and drops the
copy-paste.

```
            ┌──────────── this session (holds context) ─────────────┐
  Phase 1        Phase 2        Phase 3        Phase 4      Phase 5
  subagent   →   here      →    subagent   →   here    →    subagent
  (report)       (plan)         (review)       (apply)      (audit)
     ▲              ▲              ▲              ▲            ▲
     └── checkpoint ┘              └─ checkpoint ─┘            └─ checkpoint
```

Phases 2 and 4 stay in this session for the same reason they shared a chat before: the planner
implements its own plan, and the reasoning behind each fix is worth more than a written-down
summary of it.

## Input

```
/pr-fix run #9
/pr-fix run https://github.com/owner/repo/pull/9 --min-severity=high
/pr-fix run #9 --auto-checkpoints=false
```

Flags are parsed per `_config.md`. `--auto-checkpoints=false` runs unattended, taking the
recommended default at every decision — appropriate for a routine run on a PR you own, not for one
where fixes touch code you haven't read.

## Step: Set up

Resolve the access path per `_github.md` and the config per `_config.md`.

Determine the **absolute path of this skill directory** — the one holding `references/`. Every
subagent needs it, because a subagent starts cold and cannot find `references/report.md` from a
relative path it was never given. Record it and the working directory; both go in every subagent
prompt.

Tell the user what's about to happen, once:

> Running the full pipeline on PR #{N}. Phases 1, 3 and 5 run as subagents with fresh context;
> planning and implementation stay in this session. I'll check in with you {at three points:
> after the report, before any file is edited, and before anything is reverted | only if something
> needs a decision — `--auto-checkpoints=false`}.

## Step: Phase 1 — report (subagent)

Spawn a `general-purpose` subagent. It needs write access: it creates `.pr-fix/` and writes
`state.json` and `report.md`.

Prompt template — the three parts are from `_handoff.md` § Subagent delivery:

```
Read {SKILL_DIR}/references/report.md and follow it exactly.

Task: Phase 1 (report) for PR #{N} in {CWD}.
Config overrides: {any --flags}

You are running as a subagent with fresh context. No human is reachable:
never call AskUserQuestion, and never delete anything. If .pr-fix/ holds
stale artifacts, stop and report that instead of wiping it.

Record decisions needing a human in state.json -> pending_decisions[] and
lead your final report with them. Do not summarize what you wrote to disk —
I can read the artifacts. Report: decisions needing a human, data gaps,
and the handoff brief for Phase 2.
```

Run it in the foreground: nothing else can usefully proceed until the report exists.

When it returns, **read `.pr-fix/report.md` and `state.json` yourself.** The subagent's report is
not shown to the user and is not authoritative — the artifacts are. Relay to the user what matters:
finding counts by severity, the top findings, blocking coverage, and any data gap.

### Checkpoint 1 — scope

Unless `auto_checkpoints` is false, ask the user (one `AskUserQuestion`):

- **Proceed to planning** (recommended)
- **Adjust scope first** — e.g. raise `min_severity`, skip a category, plan only reviewer points
- **Stop here** — the report alone was the goal

This checkpoint is cheap and catches the expensive mistake: planning twenty fixes when the user
wanted the three the reviewer asked for.

## Step: Phase 2 — plan (this session)

Follow `references/plan.md` here, in this session. You have the brief from Phase 1 in context
already — there is nothing to paste.

Do not skip reading `report.md` just because a subagent you spawned wrote it. You did not see its
reasoning; you saw its report.

## Step: Phase 3 — review-plan (subagent)

Spawn a fresh `general-purpose` subagent with the `review-plan` brief you composed per
`_handoff.md`. The attention-not-conclusions rule is the whole point here: you wrote the plan, so
anything you tell this subagent about the plan's correctness corrupts the review you're asking for.

```
Read {SKILL_DIR}/references/review-plan.md and follow it exactly.

Task: Phase 3 (review-plan) for PR #{N} in {CWD}.

You are running as a subagent with fresh context. No human is reachable:
never call AskUserQuestion. For each decision the workflow puts to the user,
record it in state.json -> pending_decisions[], apply the recommended
default so the pipeline can continue, and list it in your final report.

Brief from the planning session: {BRIEF}
```

When it returns, read `plan-approved.md` and `state.json` yourself.

### Checkpoint 2 — approve the fix set

**This checkpoint is not optional, even under `--auto-checkpoints=false`.** It is the last point
before any source file is edited, and the first point where the pipeline can destroy work rather
than just waste time.

Present the verdict table, then one `AskUserQuestion` covering:

- The ⚠️ revisions and ❌ rejections, for override
- Every item in `pending_decisions[]` the subagent defaulted — especially any **blocking point
  without a fix**, which is never auto-ratifiable
- Whether to apply the approved set now

Update `state.json` with the user's decisions before continuing, and rewrite `plan-approved.md` if
any verdict changed. The implementing phase reads the file, not this conversation.

## Step: Phase 4 — implement (this session)

Follow `references/implement.md` here. Everything it says applies unchanged: anchor the tree, run
the baseline gates, apply fixes with per-fix patches, verify with the no-progress stop.

Two things to watch in orchestrated mode:

- **The dirty-tree check still applies.** Running under `run` is not consent to edit over
  uncommitted work. Ask, as the workflow says.
- **Report the retry loop as it goes.** The user is watching one session drive five phases; a
  silent four-minute gap during attempt 2 reads as a hang. One line per attempt is enough.

## Step: Phase 5 — review-impl (subagent)

Spawn a fresh `general-purpose` subagent:

```
Read {SKILL_DIR}/references/review-impl.md and follow it exactly.

Task: Phase 5 (review-impl) for PR #{N} in {CWD}.

You are running as a subagent with fresh context. No human is reachable:
never call AskUserQuestion, and NEVER revert anything — audit only. Write
verdict.md, record your recommended keep/revert decision per fix in
state.json, and return those recommendations. I will put them to the user
and apply any reverts myself.

Brief from the implementing session: {BRIEF}
```

The no-revert rule is load-bearing. A subagent reverting on its own judgment is the one action in
this pipeline that destroys work the user might have wanted, and it cannot ask before doing it.

When it returns, read `verdict.md` and `state.json` yourself.

### Checkpoint 3 — keep or revert

Present the per-fix verdict table and any regression against the baseline. One `AskUserQuestion`
covering every ⚠️ and ❌ as a multi-select.

Then apply the chosen reverts yourself, per `references/review-impl.md` § Applying a revert —
including the `git apply -R --check` dry run and the overlap handling. Re-run the Step B gates
afterwards.

Under `--auto-checkpoints=false`, **do not auto-revert**: keep everything, record the
recommendations in the verdict, and tell the user which fixes the reviewer wanted reverted. An
unattended run may waste time; it may not silently throw away code.

## Step: Close out

Report the outcome in a few lines: fixes applied and kept, gates green (with any pre-existing
failures named as such), blocking points covered, concerns left open.

Then, based on the verdict:

- **ACCEPT** → offer `/pr-fix respond` to draft the commit message and reviewer reply. Don't run it
  unprompted; it writes to someone else's PR.
- **NEEDS_ITERATION** → offer to run Phase 4 again in this session with the iteration brief. This
  session is already the Phase 2/4 session, so iteration needs no handoff at all.
- **REJECT** → offer to re-plan from Phase 2 with the re-plan brief.

Nothing is committed or pushed, in either mode, ever.

## Failure handling

- **A subagent fails or returns nothing usable** — do not guess at what it would have found. Check
  whether its artifact exists and is well-formed; if not, re-run that phase once, then fall back to
  running the phase in this session and tell the user the isolation was lost for that phase.
- **A subagent reports stale `.pr-fix/` artifacts** — subagents can't delete. Resolve it here:
  show the user what's there and confirm the wipe, then re-spawn.
- **The staleness check trips mid-run** — the PR moved while the pipeline was running. Stop at the
  current phase boundary and tell the user; don't plan or apply against a moved head.
- **Never fabricate a subagent's result.** If one is still running, say so. The notification is the
  only thing that says it finished.

## When not to use `run`

Manual mode still earns its place. Prefer the step-by-step commands when you want to read and edit
`plan.md` between phases, when the PR is large enough that you want to stop and think between
phases, or when you want a different model or a different machine for the reviewing phases. `run`
is the default; it isn't the only way.
