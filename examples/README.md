# A worked run

A complete pipeline run against a small Python PR, kept here so you can see the shape of the output
before installing — and so the phase workflows have a concrete reference for what "good" looks like.

**The PR**: `acme/ingest-service#42`, "Add retry backoff to the ingest client". Four files,
+118/−23. Two reviewers had it blocked:

- **@dana** left a `CHANGES_REQUESTED` review raising two distinct points in one body — the case
  the pipeline's coverage map exists for, since a naive reading collapses it into one item.
- **codescene[bot]** left a `CHANGES_REQUESTED` on a complexity regression. The pipeline treats bot
  and human reviews identically; neither gets a privileged name.

**What happened**: Phase 1 found 7 findings and mapped all 3 blocking points. Phase 2 planned 6
fixes — including one a severity filter would have dropped, because a bot was blocking on it.
Phase 3 approved 4, revised 1, and rejected 1 as a false positive. Phase 4 applied the 5 surviving
fixes and went green on attempt 2, after jitter made a ceiling assertion flaky. Phase 5 found an
over-broad exception clause that the tests didn't catch, reverted nothing, and returned
NEEDS_ITERATION on that single fix.

## Files

| File | Phase | Notice |
|---|---|---|
| [`state.json`](state.json) | all | `blocking_coverage[]` — three points, three dispositions, nothing unmapped. This is what makes the guarantee checkable instead of hopeful. |
| [`report.md`](report.md) | 1 | §2's coverage table decomposes @dana's single review body into two separately-tracked points. |
| [`plan.md`](plan.md) | 2 | FIX-04 exists only because a *bot* flagged it. Severity would have dropped it; the blocking rule keeps it. |
| [`verdict.md`](verdict.md) | 5 | The gate table's `At baseline` column — the pre-existing mypy failure never counts against the run. |

`plan-approved.md`, `changes.md` and `response.md` from this run are elided for brevity; their
structure is in the phase workflows.

## The three things worth stealing

1. **Decompose review bodies into points, not reviews.** @dana wrote one paragraph containing two
   asks. Tracked as one item, the second gets lost, and the review never clears.
2. **Run the gates before you change anything.** `gates.baseline` shows mypy already failing on
   `src/legacy_shim.py`. Without that record, Phase 4 burns retries on it and Phase 5 reports a
   regression that isn't one.
3. **Severity filters your findings, never the reviewer's.** F-04 is a Medium code-quality nit by
   the pipeline's own reckoning. It's also the reason a bot is blocking the PR, so it gets fixed.
