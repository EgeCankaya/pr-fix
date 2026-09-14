# Implementation Verdict — Add retry backoff to the ingest client

**PR**: https://github.com/acme/ingest-service/pull/42
**Reviewed**: 2026-09-14 11:31 UTC
**Phase 4 reported**: PASSED — all gates green on attempt 2 of 3

---

## Overall Verdict: NEEDS_ITERATION

Four of five applied fixes are correct and match the approved plan. FIX-02 works for every case the
tests cover but widens an exception clause further than the plan approved, in a way that will
swallow a cancellation. That is a one-line change, not a re-plan — hence NEEDS_ITERATION rather
than REJECT, and nothing has been reverted.

---

## Per-Fix Results

| # | Fix | File | Verdict | Action taken |
|---|-----|------|---------|-------------|
| FIX-01 | Clamp the backoff exponent | `src/ingest/client.py` | ✅ Good | Kept |
| FIX-02 | Retry only on 5xx and 429 | `src/ingest/client.py` | ⚠️ Needs work | Flagged for iteration (not reverted) |
| FIX-03 | Add full jitter | `src/ingest/client.py` | ✅ Good | Kept |
| FIX-04 | Flatten `_should_retry` | `src/ingest/client.py` | ✅ Good | Kept |
| FIX-05 | Parametrized retry-policy test | `tests/test_client.py` | ✅ Good | Kept |
| FIX-06 | Docstring | `src/ingest/client.py` | — | Not applied (rejected in Phase 3) |

### FIX-02 — what's wrong

`plan-approved.md` approved `except HTTPError as exc:`. The applied code at
`src/ingest/client.py:61` is `except (HTTPError, asyncio.TimeoutError, OSError) as exc:`, and
`_should_retry` then dereferences `exc.response`, which only `HTTPError` has.

Two consequences:

1. `asyncio.TimeoutError` is a subclass of `asyncio.CancelledError` on this runtime. A cancelled
   ingest task will now be caught and **retried** rather than propagating — the shutdown path in
   `pipeline.py:142` will hang until `max_attempts` is exhausted.
2. An `OSError` reaching `_should_retry` raises `AttributeError` on `exc.response`, masking the
   original error.

Neither is covered by the tests, which is why Phase 4's gates went green. The plan's narrower
clause was correct; the widening appears to have come from attempt 1's retry round, which the
implementer's brief flagged as the least-reviewed code in the diff — that flag was accurate.

**Suggested change**: restore the approved `except HTTPError as exc:`. If timeouts genuinely need
retrying, that belongs in a separate fix with its own predicate branch and its own test.

---

## Verification Results

| Gate | Command | At baseline | Now | Verdict |
|------|---------|-------------|-----|---------|
| Tests | `make test` | ✅ pass | ✅ 47 passed | — |
| Lint | `ruff check src tests` | ✅ pass | ✅ clean | — |
| Format | `ruff format --check src tests` | ✅ pass | ✅ clean | — |
| Types | `mypy src` | ❌ fail | ❌ fail (same signature) | pre-existing — not this run |
| Code-health (Step C) | — | — | ✅ `_should_retry` depth 4 → 2; no new duplication in tests | — |

**Suite resolution**: matches `verify-resolved.md`. Independently re-resolved from
`.github/workflows/ci.yml`, `CONTRIBUTING.md` and `pyproject.toml`, and arrived at the same four
gates.

The `mypy` failure is `src/legacy_shim.py:88 assignment`, identical to the signature recorded in
`gates.baseline` before any fix was applied. It is untouched by this PR and does not count against
the implementation. Worth noting it is also red on `main`.

---

## Blocking-Review Coverage

Whether the fixes actually clear the reviews that blocked the PR.

| Review point | Finding | Fix | Status |
|--------------|---------|-----|--------|
| Retry ceiling is unbounded (@dana) | F-01 | FIX-01 | ✅ addressed — capped at `MAX_BACKOFF_SECONDS`, covered by `test_retry_policy[ceiling]` |
| 400 retried like a 503 (@dana) | F-02 | FIX-02 | ⚠️ addressed, but see the FIX-02 concern above — the 400 path is correct; the over-broad clause is the problem |
| Complexity in `_should_retry` (codescene[bot]) | F-04 | FIX-04 | ✅ addressed — nesting depth 2, below the threshold of 3 |

All three blocking points are addressed in substance. Neither of @dana's points should be reported
as resolved until FIX-02's clause is narrowed, since the fix as applied changes behavior they
didn't ask for.

---

## Remaining Concerns

- **FIX-02's exception clause**, above. The only blocker.
- **FIX-03's jitter is untested for distribution.** `test_retry_policy` seeds the RNG and asserts
  the delay stays under the cap, which is right, but nothing asserts the jitter actually varies.
  A frozen `random.uniform` returning a constant would pass. Minor, and arguably out of scope.
- **`pipeline.py:88` still reads `max_attempts` from config unvalidated.** FIX-01 makes an absurd
  value harmless rather than catastrophic, so this is no longer urgent — but it was the root of
  F-01 and nothing in this PR bounds the input. Worth a follow-up issue, not a change here.

---

## Next Steps

**NEEDS_ITERATION** — one fix needs another pass. Go back to the Phase 2/4 chat and run
`/pr-fix implement` with the brief below; the tree already holds the round-1 fixes, so the
dirty-tree warning is expected. Only FIX-02 changes — leave FIX-01, FIX-03, FIX-04 and FIX-05 alone.

```
/pr-fix implement PR #42 at a1b2c3d4 — iteration round 1; start from verdict.md's FIX-02 section. The tree already holds all five round-1 fixes, so the dirty-baseline check will fire and that is expected — the baseline SHA in state.json is still correct. Only FIX-02 needs rework: client.py:61 widened the approved `except HTTPError as exc:` to also catch asyncio.TimeoutError and OSError, which makes a cancelled task retry through shutdown and makes _should_retry raise AttributeError on exc.response. Restore the narrow clause the plan approved; do not redesign the predicate. Nothing was reverted, so no patch needs re-applying. FIX-01, FIX-03, FIX-04 and FIX-05 were all verified good — leave them untouched. The tests pass today and will still pass after the narrowing, so add a case that asserts a cancelled awaitable propagates rather than retrying, or the same gap stays open. mypy is red on src/legacy_shim.py:88 and was red at baseline; ignore it. Phase 5 re-resolved the suite independently and got the same four gates as verify-resolved.md.
```
