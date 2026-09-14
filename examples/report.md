# PR Issue Report — Add retry backoff to the ingest client

**PR**: https://github.com/acme/ingest-service/pull/42
**Branch**: `feature/retry-backoff` → `main`
**Stats**: 4 files, +118/−23
**Generated**: 2026-09-14 10:02 UTC
**Analysis depth**: full read of all changed files
**Data completeness**: complete

---

## §1 — PR Overview

Adds a retry wrapper around the ingest client's HTTP calls. `retry_with_backoff` retries a failed
request up to `max_attempts` times, doubling the wait each attempt; `_should_retry` decides whether
a given response warrants another attempt. Two callers in `src/ingest/pipeline.py` are switched
over, and `tests/test_client.py` gains three cases.

The shape is right and the caller migration is clean. The concerns below are concentrated in the
retry policy itself — what gets retried, and for how long.

### Changed Files

| File | Changes | Read as | Summary |
|------|---------|---------|---------|
| `src/ingest/client.py` | +81/−9 | full | The retry wrapper and its predicate |
| `src/ingest/pipeline.py` | +6/−12 | full | Two call sites switched to the wrapper |
| `tests/test_client.py` | +28/−0 | full | Three new cases for the happy path |
| `docs/operations.md` | +3/−2 | full | Notes the new retry behavior |

---

## §2 — Historical Context

### Resolved Issues (Past Feedback)

| # | Reviewer | What was raised | How it was resolved |
|---|----------|----------------|-------------------|
| H-01 | @dana | Retry logic was inline in both callers | Extracted into `retry_with_backoff` in `client.py` (commit `9f3c1aa`) |
| H-02 | @sam | `time.sleep` blocks the event loop | Switched to `await asyncio.sleep` (commit `c4d8e02`) |

### Open Review Comments

| # | Reviewer | File:Line | Comment | Status |
|---|----------|-----------|---------|--------|
| H-03 | @sam | `src/ingest/client.py:49` | "where does 30 come from?" | 🟡 Open |

### Pending Change Requests

| Reviewer | Date | Summary |
|----------|------|---------|
| @dana | 2026-09-13 | Retry ceiling is unbounded; non-retryable statuses are retried |
| codescene[bot] | 2026-09-13 | Complexity regression in `_should_retry` |

### Blocking-review coverage

Every distinct point of each current blocking (`CHANGES_REQUESTED`, not superseded) review, mapped
to the finding ID(s) covering it. Mirrors `state.json` → `blocking_coverage[]`. Phase 2 plans a fix
for each; Phase 3 re-derives this map from the live reviews rather than trusting it.

| Review (id, reviewer) | Point | Finding ID(s) | Disposition |
|-----------------------|-------|---------------|-------------|
| 2481003, @dana | The retry ceiling is unbounded — this will overflow on a long outage | F-01 | fix |
| 2481003, @dana | A 400 gets retried the same as a 503; only 5xx and 429 should retry | F-02 | fix |
| 2481188, codescene[bot] | Complexity regression in `_should_retry` (nesting depth 4, threshold 3) | F-04 | fix |

> @dana's review is a single paragraph raising **two** separate asks. They are tracked as two
> points with two finding IDs. Collapsed into one, the second would very likely be planned around
> and the `CHANGES_REQUESTED` would not clear.

### Recurring Themes

- Both blocking human points are about the retry *policy* — what to retry and for how long — not
  about the wrapper's structure, which reviewers have already accepted.
- @sam has now twice asked about unexplained constants (`H-02`'s timeout, `H-03`'s `30`).

---

## §3 — New Findings

The following are observations from analyzing the current state of the PR. These are advisory —
consider each on its merits.

### F-01: Backoff exponent is unbounded

- **File**: `src/ingest/client.py` L42–L55
- **Severity**: High
- **Category**: Bug
- **What I noticed**: `delay = 2 ** attempt` has no ceiling. `max_attempts` defaults to 5, but the
  parameter is public and `pipeline.py:88` passes a value from config. At 40 attempts the delay is
  ~35 years; well before that, the `asyncio.sleep` argument exceeds what the event loop will
  schedule. This is @dana's first point.
- **Suggestion**: Consider clamping — `min(2 ** attempt, MAX_BACKOFF_SECONDS)`. The named constant
  would also answer @sam's open question at L49 about where `30` comes from.
- **Confidence**: High

### F-02: Retry loop swallows non-retryable 4xx responses

- **File**: `src/ingest/client.py` L58–L71
- **Severity**: High
- **Category**: Bug
- **What I noticed**: The `except` catches every `HTTPError` and retries. A 400 or 422 is a client
  error that will fail identically every time, so the caller waits through the full backoff
  sequence before seeing an error it could have had immediately — and the ingest endpoint sees five
  copies of a malformed request. This is @dana's second point.
- **Suggestion**: Consider retrying only on 5xx and 429, and re-raising everything else
  immediately. `_should_retry` is the natural home for that check.
- **Confidence**: High

### F-03: No jitter on the backoff interval

- **File**: `src/ingest/client.py` L42–L55
- **Severity**: Medium
- **Category**: Performance
- **What I noticed**: Every worker that fails at the same moment — which is what happens when the
  upstream goes down — retries at the same moment. `docs/operations.md` says the pipeline runs 24
  workers, so the upstream gets a synchronized burst of 24 on each retry.
- **Suggestion**: You might add full jitter (`random.uniform(0, delay)`), which is the usual
  remedy. Worth confirming the test in `tests/test_client.py` seeds the RNG if you do.
- **Confidence**: Medium

### F-04: `_should_retry` nests four levels deep

- **File**: `src/ingest/client.py` L74–L96
- **Severity**: Medium
- **Category**: Code quality
- **What I noticed**: `if response → if status → if attempt → if header` puts the actual decision
  four levels in. `codescene[bot]` is blocking the PR on this; the repo's threshold is 3.
- **Suggestion**: Consider early returns for the disqualifying cases, leaving the retry decision at
  the top level. That would likely fold together with F-02, which changes this same predicate.
- **Confidence**: High

### F-05: No test covers the retry ceiling

- **File**: `tests/test_client.py`
- **Severity**: Medium
- **Category**: Test gap
- **What I noticed**: The three new cases cover one success, one retry-then-success, and one
  exhaustion. Nothing exercises a large `max_attempts`, which is exactly where F-01 bites.
- **Suggestion**: A parametrized case over attempt counts would cover the ceiling and the 4xx path
  together without repeating the setup block.
- **Confidence**: High

### F-06: Magic number 30 for the cap

- **File**: `src/ingest/client.py` L49
- **Severity**: Low
- **Category**: Code quality
- **What I noticed**: `30` appears unexplained; @sam asked about it at H-03 and it is still open.
- **Suggestion**: A module-level `MAX_BACKOFF_SECONDS = 30` would document it and be reused by F-01.
- **Confidence**: High

### F-07: `retry_with_backoff` has no docstring

- **File**: `src/ingest/client.py` L40
- **Severity**: Info
- **Category**: Convention
- **What I noticed**: `CLAUDE.md` asks for docstrings on public functions. This one is arguably
  module-private despite the name.
- **Suggestion**: Add one, or confirm the module docstring is considered sufficient here.
- **Confidence**: High

---

## §4 — Summary Dashboard

| Severity | Count |
|----------|-------|
| Critical | 0 |
| High | 2 |
| Medium | 3 |
| Low | 1 |
| Info | 1 |

**Open historical issues**: 1
**Blocking points covered**: 3 of 3
**Files with most findings**: `src/ingest/client.py` (6), `tests/test_client.py` (1)

### Overall Impression

The refactor itself is the right call and addresses both points from the earlier review round —
the logic is out of the call sites and the blocking sleep is gone. Reviewers have accepted the
structure; nothing here suggests reworking it.

What's left is the retry *policy*. Two of the three blocking points (F-01, F-02) are about
correctness under failure, and they share a fix site: the predicate that decides whether to retry.
F-04, the bot's complexity flag, lives in that same function, so all three are likely to resolve
together in one careful pass over `_should_retry` and the delay calculation. F-03 is the one
genuinely new suggestion — no reviewer raised jitter, and with 24 workers it is worth considering
before this ships.
