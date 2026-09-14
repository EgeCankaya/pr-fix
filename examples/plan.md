# Fix Plan — Add retry backoff to the ingest client

**PR**: https://github.com/acme/ingest-service/pull/42
**Based on report**: `.pr-fix/report.md`
**Generated**: 2026-09-14 10:19 UTC
**Total fixes proposed**: 6
**Blocking points covered**: 3 fixed, 0 justified skips, 0 unmapped

---

## Execution Order

| # | Fix | File | Severity | Depends on |
|---|-----|------|----------|------------|
| FIX-01 | Clamp the backoff exponent to `MAX_BACKOFF_SECONDS` | `src/ingest/client.py` | High | — |
| FIX-02 | Retry only on 5xx and 429 | `src/ingest/client.py` | High | — |
| FIX-03 | Add full jitter to the backoff interval | `src/ingest/client.py` | Medium | FIX-01 |
| FIX-04 | Flatten `_should_retry` with early returns | `src/ingest/client.py` | Medium | FIX-02 |
| FIX-05 | Parametrized test for the ceiling and the 4xx path | `tests/test_client.py` | Medium | FIX-01, FIX-02 |
| FIX-06 | Add a docstring to `retry_with_backoff` | `src/ingest/client.py` | Info | — |

FIX-04 comes after FIX-02 deliberately: FIX-02 changes the same predicate, and flattening first
would mean rewriting the flattened version. FIX-05 comes last so it tests the final behavior rather
than an intermediate state.

---

## Fixes by File

### `src/ingest/client.py`

### FIX-01 — Clamp the backoff exponent to `MAX_BACKOFF_SECONDS` (addresses F-01, F-06)

**Severity**: High
**File**: `src/ingest/client.py` L42–L55

**Current code**:
```python
async def retry_with_backoff(fn, max_attempts=5):
    for attempt in range(max_attempts):
        try:
            return await fn()
        except HTTPError:
            if attempt == max_attempts - 1:
                raise
            delay = 2 ** attempt
            await asyncio.sleep(delay)
```

**Proposed change**:
```python
MAX_BACKOFF_SECONDS = 30


async def retry_with_backoff(fn, max_attempts=5):
    for attempt in range(max_attempts):
        try:
            return await fn()
        except HTTPError:
            if attempt == max_attempts - 1:
                raise
            delay = min(2 ** attempt, MAX_BACKOFF_SECONDS)
            await asyncio.sleep(delay)
```

**Rationale**: Bounds the delay regardless of `max_attempts`, which `pipeline.py:88` reads from
config and can therefore set arbitrarily high. Naming the constant also answers @sam's open comment
at L49, folding F-06 in at no extra cost.

**Test changes**:
- Covered by FIX-05's parametrized case at `max_attempts=40`.

**Risk assessment**:
- **Could break**: nothing at the call sites — the signature is unchanged.
- **Mitigation**: FIX-05 asserts the delay never exceeds the cap.

**Dependencies**: none.

---

### FIX-02 — Retry only on 5xx and 429 (addresses F-02)

**Severity**: High
**File**: `src/ingest/client.py` L58–L71

**Current code**:
```python
        except HTTPError:
            if attempt == max_attempts - 1:
                raise
```

**Proposed change**:
```python
        except HTTPError as exc:
            if not _should_retry(exc.response, attempt, max_attempts):
                raise
```

**Rationale**: Moves the retryability decision into the predicate that already exists for it, so a
400 or 422 surfaces immediately instead of after the full backoff sequence. This is @dana's second
blocking point.

**Test changes**:
- FIX-05 adds a case asserting a 400 raises after exactly one call.

**Risk assessment**:
- **Could break**: any caller relying on a 4xx being retried. Grep found none — both call sites in
  `pipeline.py` treat a raised `HTTPError` as fatal either way.
- **Mitigation**: the new 400 case in FIX-05.

**Dependencies**: none, but FIX-04 must land after this.

---

### FIX-04 — Flatten `_should_retry` with early returns (addresses F-04)

**Severity**: Medium
**File**: `src/ingest/client.py` L74–L96

**Rationale**: `codescene[bot]` is blocking on nesting depth 4 against a threshold of 3. Early
returns for the disqualifying cases leave the retry decision at the top level and drop the depth to
2. Note this is planned **because a blocking review raised it**, not because of its severity — a
Medium code-quality finding would normally sit below the fix threshold, but leaving it is what
keeps the bot's `CHANGES_REQUESTED` in place.

**Test changes**: none — behavior is unchanged. FIX-05's cases cover the predicate's outcomes.

**Risk assessment**:
- **Could break**: the predicate's behavior, if a branch is dropped in the rewrite.
- **Mitigation**: FIX-05 exercises all four outcome paths.

**Dependencies**: FIX-02 (which rewrites the same predicate's contract).

*(FIX-03 and FIX-06 blocks elided in this example.)*

---

### `tests/test_client.py`

### FIX-05 — Parametrized test for the ceiling and the 4xx path (addresses F-05)

**Severity**: Medium
**File**: `tests/test_client.py`

**Rationale**: One parametrized case covers the ceiling, the 4xx short-circuit, and the retryable
5xx path without repeating the client-construction block three times. The existing three cases
already repeat that block; extending the pattern would likely trip the duplication gate in Step C.

**Test changes**: adds `test_retry_policy` with four parameter sets; seeds `random.seed(0)` so
FIX-03's jitter doesn't make the ceiling assertion flaky.

**Risk assessment**:
- **Could break**: nothing in `src/`.
- **Mitigation**: n/a.

**Dependencies**: FIX-01, FIX-02.

---

## Skipped Findings

| Finding | Blocking? | Reason skipped |
|---------|-----------|---------------|
| F-07 | no | Info-level; a docstring is proposed as FIX-06 rather than skipped, since it also closes a `CLAUDE.md` convention point |

No blocking-review point appears in this table. Every point in the report's §2 coverage map
resolves to a planned fix above.

---

## Test Strategy

- `FIX-05` adds `test_retry_policy`, parametrized over four cases: success, retryable 5xx, 429, and
  non-retryable 400.
- The ceiling assertion pins `max_attempts=40` and asserts no delay exceeds `MAX_BACKOFF_SECONDS`.
- `random.seed(0)` is set in the fixture so FIX-03's jitter is deterministic under test.

Verification after implementation: see `.pr-fix/verify-resolved.md` (resolved from CI +
`CONTRIBUTING.md` + `pyproject.toml`). Phases 4 and 5 run that same suite:

| Step | Command |
|---|---|
| A | `ruff format src tests`, `ruff check --fix src tests` |
| B | `make test`, `ruff check src tests`, `mypy src` |
| C | CodeScene delta — watch nesting depth in `client.py` and duplication in `test_client.py` |
