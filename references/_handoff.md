# Handoff Brief (shared)

Every phase ends by handing the next one a **brief**: a short paragraph priming it with what this
session learned. This file is the single source of truth for writing and reading briefs — the phase
workflows reference it instead of restating the rules, the same way they reference `_verify.md`.

The `.pr-fix/` files stay authoritative. The brief carries what they don't: what the user said in
this chat, where this session is unsure, what surprised it, and where the next phase should spend
its attention.

A brief is delivered one of two ways, and the content is identical either way:

- **Manual mode** — printed as a ready-to-paste `/pr-fix <command> <brief>` prompt for the user to
  carry into the next chat.
- **`run` mode** — passed by the orchestrator as the `prompt` of a subagent, or held in the
  orchestrator's own context when the next phase runs in-session. See § Subagent delivery.

## Format

In manual mode, print the prompt as one fenced block **at the top level of the message** (not
inside a blockquote or list), so it copies cleanly:

```
/pr-fix <command> <brief>
```

- The brief goes **on the same line as the command, as one paragraph with no line breaks**, so it
  arrives intact as the text after the subcommand.
- **About 10 sentences** (8–12, under ~250 words). If it runs long, cut the least useful sentence.
- Open with the anchor: PR number, short head SHA, and the `.pr-fix/` file or item to start from.
- Be concrete: finding/fix IDs, `file:line`, function names. Never "some issues in the tests".
- Address the next session and name who did what ("the user chose…", "Phase 2 assumed…"). The
  user pastes the prompt as their own message, so "I" would be ambiguous.
- Say *where* to paste it ("fresh chat" / "your Phase 2 chat") in the surrounding message, not in the
  brief.

## What goes in (highest value first)

1. **What the user decided or said in this chat** — priorities, overrides, scope cuts, "don't touch
   X". The next chat can't see this conversation, so nothing else in the brief is worth more.
2. **Uncertainty** — the findings or fixes this session is least sure of, and the assumption each
   rests on, so the next phase checks the assumption instead of inheriting it.
3. **Couplings** — items that share a file or root cause, or must land in a particular order.
4. **Environment caveats** — tree not on the PR head, a dirty baseline the user accepted, a fetch
   that failed (`state.json` → `github.fetch_gaps`), a gate already red at baseline, a degraded
   `git-only` access path with no review history.
5. **CI traps** — changed files likely to trip the gates in `_verify.md` (near the complexity
   ceiling, duplication-prone tests).

Leave out whatever the next phase will read from `.pr-fix/` anyway: counts, tables, full finding
text. "Check F-03's confidence" beats restating F-03.

## Briefing a fresh-eyes review (`review-plan`, `review-impl`)

Phases 3 and 5 run with fresh context so the author's reasoning can't anchor them. A brief into
those phases directs **attention**, never **conclusions**: state doubts, assumptions, and late
changes; never vouch for correctness or tell the reviewer what it can skip.

- ✅ "Fix 3 assumes `parse_row` never gets an empty list — check the callers."
- ✅ "Fix 5's test was rewritten during retry 2; it's the least-reviewed code in the diff."
- ❌ "Fixes 1–4 are straightforward and correct."
- ❌ "The mypy error is pre-existing, ignore it."

The last one is worth dwelling on: since Phase 4 records a gate baseline, "pre-existing" is a fact
in `state.json` → `gates.baseline`, not a claim to pass along in prose. Point the reviewer at the
baseline; let it draw its own conclusion.

## Per-target checklist

Find the row for the command you're handing off to and cover whichever items apply to this run.

| Next command | Written at the end of | Cover |
|---|---|---|
| `plan` | Phase 1 (report) | Blocking-review points that must each get a fix (by ID); low-confidence findings to confirm in code before fixing; findings likely to collapse into one fix; data missing from the report (fetch gaps, degraded access path); whether the working tree matches the PR head. |
| `review-plan` | Phase 2 (plan) | Fixes resting on an unverified assumption (name it); close-call skips; fixes that interact or must land in order; "Current code" snippets that are abbreviated or line numbers that are approximate; test changes at risk in `_verify.md` Step C. |
| `implement` | Phase 3 (review-plan) | That `plan-approved.md` supersedes the chat's own `plan.md`; each ⚠️ revision in one clause (what changed from the original); rejected fixes not to apply from memory; skipped findings the user pulled in (the planning chat never designed these); user overrides; whether the PR head still matched. |
| `review-impl` | Phase 4 (implement) | Deviations from `plan-approved.md` and why; code changed during retries (the least-reviewed code); fixes skipped at apply time; gates that were already red at baseline (point at `gates.baseline`, don't characterize them); files that already had uncommitted changes, if the user accepted a dirty baseline; on FAILED, the failing gates and the suspected cause framed as a hypothesis to test. |
| `respond` | Phase 5 (ACCEPT) | Which blocking points the response must claim as addressed and which as justified-skip; any fix whose reviewer-facing description differs from its internal rationale; concerns the verdict left open that the PR comment should be honest about. |
| `implement` (iteration) | Phase 5 — NEEDS_ITERATION | That this is an iteration round: the tree already holds round-1 fixes, so the dirty-tree check is expected; which fixes to rework and the issue with each (point to `verdict.md`); which were reverted and by which patch; which to leave alone. |
| `plan` (re-plan) | Phase 5 — REJECT | Which approaches failed and why, so the redesign doesn't repeat them; which fixes were sound and can carry over; the root cause the verdict identified; whether the rejected changes were reverted or are still in the tree, and in which files. |

## Example

A `review-plan` handoff — it points at risk and never vouches for a fix:

```
/pr-fix review-plan PR #42 at a1b2c3d4 — start from plan.md's Execution Order. In the planning chat the user asked to keep the public `load_config()` signature unchanged, which is why Fix 2 adds a wrapper instead of a parameter; check that the wrapper covers every caller. Fix 3 assumes `parse_row` never receives an empty list, and Phase 2 did not confirm that against the fixtures in tests/unit/test_ingest.py. Fixes 1 and 4 both edit `ingest.py::_flush` and must land in that order, so check they still compose once both are applied. F-06 was skipped as Low, but it sits in the same function as Fix 4 and may be cheap to fold in. Fix 5's "Current code" snippet elides lines 88–120, so compare it with the file directly. Fix 3's new test repeats the same setup block three times, which is likely to be flagged as duplication. The one blocking review has two points, which Phase 2 mapped to Fixes 1 and 2; re-derive that mapping from the live review rather than trusting it.
```

## Subagent delivery

Under `/pr-fix run`, the orchestrator spawns Phases 1, 3 and 5 as subagents instead of sending the
user to a fresh chat. A subagent starts with no conversation history, which is the same isolation
the manual fresh-chat rule was buying — so the brief's job, and every rule above, is unchanged.

The orchestrator composes the subagent prompt as:

1. The phase instruction: *"Read `references/<phase>.md` in the pr-fix skill directory and follow
   it exactly for PR #{N} in {cwd}."*
2. The brief, verbatim, written to the same spec as a manual brief.
3. The execution-context line: *"You are running as a subagent. No human is reachable. Do not call
   `AskUserQuestion` — record decisions needing a human in `state.json` and list them in your final
   report."*

Two things a subagent brief must add that a manual one doesn't:

- **The absolute path of the skill directory**, since the subagent has to find `references/`.
- **The working directory**, since it starts cold.

A subagent's final report is not shown to the user. The orchestrator relays what matters, so the
subagent's report should lead with decisions needing a human and anything that contradicts the
brief — not a summary of what it wrote to disk, which the orchestrator can read itself.

## Receiving a brief

Text after the subcommand (`/pr-fix plan <brief>`) is the handoff brief. It is optional — every
phase must run correctly without one.

- Read it first, and use it to decide where to look hardest.
- **The files and the code win.** If the brief contradicts a `.pr-fix/` file or the source, trust
  the file/source and tell the user about the mismatch.
- If it names extra `.pr-fix/` files to read (e.g. `verdict.md` on an iteration round), read them.
- In `review-plan` / `review-impl` it narrows **attention, not scope**: still verify every fix in
  full, and treat its claims as hypotheses to check.
