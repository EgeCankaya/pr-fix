# Phase 6 — Respond (optional)

Close the loop with the PR. Draft the commit message and the reviewer-facing comment from what the
pipeline already computed, so the human doesn't have to re-explain five phases of reasoning by
hand.

Steps below are named, not numbered.

## Why this phase exists

Phases 1–5 end with verified changes sitting in the working tree and a "commit when ready." But
the audience for a *PR* pipeline is the reviewers, and everything they need to see is already
computed: `blocking_coverage[]` says which of their points were addressed, `fixes[]` says how, and
`skips[]` says which were justified and why. Re-deriving that by hand is the most tedious part of
the whole flow, and the part most likely to be done badly under time pressure.

## What it will and won't do

- ✅ Draft a commit message mapping fixes to findings
- ✅ Draft one PR comment covering every blocking point
- ✅ Post the comment and resolve threads — **only on an explicit yes, each time**
- ❌ Never commit, never push, never merge, never approve
- ❌ Never post anything without showing the exact text first

Posting to a PR is public and reaches other people. Treat every write as requiring its own
confirmation, and never infer standing permission from an earlier yes in the session.

## Prerequisites

- `.pr-fix/verdict.md` with an overall verdict of **ACCEPT**

If the verdict is NEEDS_ITERATION or REJECT:
> The verdict is {VERDICT}, so there's nothing to report to reviewers yet. Finish the iteration
> first — `/pr-fix status` shows what's outstanding.

Run this only when the user asks. It is not an automatic continuation of Phase 5.

## Step: Load context

Read the brief per `_handoff.md` if one followed `respond`. Resolve the access path per
`_github.md`, then run the staleness check per `_staleness.md`.

Staleness matters more here than anywhere else: posting "I addressed your three points" to a PR
that has since moved is worse than posting nothing. If the PR advanced since Phase 1, **stop** and
say so — the reviewer's points may have changed.

Read `state.json` (blocking coverage, fixes, skips, gates), `verdict.md`, and `changes.md`.

## Step: Draft the commit message

Use the repo's convention — check recent history (`git log --oneline -20`) and `CONTRIBUTING.md`
for a format (Conventional Commits, a ticket prefix, a trailer). Match it rather than imposing one.

```
{type}({scope}): address review feedback on {short subject}

{One paragraph: what changed and why, in the author's voice, not the pipeline's.}

- {FIX-01}: {what changed} ({F-01} — {reviewer}'s point about {topic})
- {FIX-02}: {what changed} ({F-03})

Verified: {gates that ran and passed}.
{If a gate was red at baseline and still is: "Pre-existing: {gate} ({signature}), unrelated to this change."}
```

Do not mention the pipeline, its phases, or the tooling in the commit message. The commit describes
the code change; how it was produced isn't part of the repository's history.

Follow whatever attribution convention the user's environment specifies for commits they ask you to
author — add nothing beyond that.

## Step: Draft the PR comment

One comment, covering every blocking point. Build it directly from `blocking_coverage[]`.

```markdown
Thanks for the review — here's what changed.

**Addressed**

| Your point | What changed |
|---|---|
| {point} | {one sentence, naming the file} — `{file}:{line}` |

**Not changed, and why**

| Your point | Why |
|---|---|
| {point} | Already fixed on the current head in `{file}:{line}` ({sha}). |
| {point} | I don't think this one applies — {concrete reason}. Happy to change it if you disagree. |

**Verification**: {gates that ran and passed}.
{If a gate was red before these changes too: "{Gate} was already failing on this branch before
these changes ({signature}); I've left it alone as out of scope."}

{If the verdict left concerns open: one honest sentence naming them.}
```

Three rules for this text:

1. **Every blocking point appears**, in one table or the other. A point silently missing from the
   comment is exactly the failure the coverage map exists to prevent.
2. **A disputed point is phrased as a position, not a verdict.** "I don't think this applies
   because X — happy to change it" invites a reply. "This is a false positive" ends the
   conversation badly and is sometimes wrong.
3. **Don't oversell.** If Phase 5 left a concern open, say so. A reviewer who finds a problem you
   knew about and didn't mention trusts the next comment less.

Write both drafts to `.pr-fix/response.md` so they survive the session.

## Step: Confirm, then act

Show the user both drafts in full. Then ask what to do, offering:

- **Post the comment** — per `_github.md` (*Post a PR comment*)
- **Post and resolve the addressed threads** — per `_github.md` (*Resolve a review thread*), only
  for threads whose point is in the Addressed table; never resolve a thread you didn't act on
- **Copy only** — leave everything in `.pr-fix/response.md` for the user to post themselves
- **Commit with the drafted message** — `git commit`, staging only files in the fix set; still no push

Default to **copy only**. Each action needs its own yes; do not batch them behind one confirmation.

If the access path is `git-only`, posting isn't available — say so and hand over the drafts.

Set `state.json` → `phases.respond` to complete, recording which actions the user approved.

## Step: Close out

> ✅ Drafts written to `.pr-fix/response.md`. {What was posted, or "nothing was posted."}
>
> The pipeline is complete. `/pr-fix clean` wipes `.pr-fix/` when you're done with it — the patches
> in `.pr-fix/patches/` go with it, so keep them until the changes are committed.

That last point matters: the per-fix patches are the only record of which change belonged to which
fix, and they're gone after a clean.

## Error handling

- **Comment post fails** — report the error and leave `response.md` in place. Never retry silently
  or post a second copy; a duplicate comment on someone's PR is noise they have to clean up.
- **A thread won't resolve** — some are resolvable only by the review author. Note it and move on.
- **The user asks to push** — that's outside this phase. Point them at their own `git push`; the
  pipeline never pushes.
