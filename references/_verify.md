# Verification Suite (shared)

The checks Phases 2, 4, and 5 run against the working tree. This file is the single source of
truth for *how* the suite is found and run — the phase workflows reference it instead of restating
commands. The commands themselves come from the target project, because every repo's toolchain
differs.

## Resolve the suite

**Project override.** If `.claude/pr-fix/verify.md` exists in the target repo, use its commands
verbatim and skip the rest of this section.

Otherwise, build the suite from these sources:

- **CI** — the jobs that run on pull requests (`.github/workflows/*.yml` and similar). CI decides
  *what* must pass: a gate CI enforces that you skip locally is a bounce waiting to happen.
- **Project instructions** — `CLAUDE.md`, `AGENTS.md`, `CONTRIBUTING.md`. These decide *how* to run
  each check locally (wrappers, env vars, flags); where they name a command for a gate, use theirs.
- **Tool config** — `package.json` scripts, `Makefile` / `justfile` targets, `pyproject.toml`,
  `tox.ini` / `noxfile.py`, `Cargo.toml`, `go.mod`, and so on. It fills whatever the first two
  leave open, and is the whole source when the repo has no CI.

Sort what you find into Steps A–C below. Prefer the project's own wrappers (`make test`,
`npm run lint`, `uv run …`) over bare tool calls so the right environment and config apply. If a
gate can't run locally (it needs a service, credentials, or hardware), mark it **not run** with the
reason — never report it as passing. If nothing defines a gate at all, say so; don't invent a
toolchain the project doesn't use.

## Persisting the resolution

**Phase 2 resolves the suite and writes `.pr-fix/verify-resolved.md`**, listing each command, its
step (A/B/C), and its source. It also sets `state.json` → `gates.resolved_from`.

- **Phase 4 reuses that file.** It runs in a different session from Phase 2 and would otherwise
  re-derive the suite from scratch, risking a different answer than the one the plan was designed
  against. If the file is missing, resolve it and write it.
- **Phase 5 re-resolves independently and does not read the file first.** That independence is the
  point of a fresh-eyes audit: a reviewer that inherits the implementer's idea of "the tests" can't
  catch a gate the implementer never ran.
- **Then Phase 5 diffs its resolution against `verify-resolved.md`.** A mismatch is itself a
  finding — usually "Phase 4 never ran a gate CI enforces" — and belongs in the verdict.

The format is the same as the override file, so a good resolution can be promoted into
`.claude/pr-fix/verify.md` by copying it.

## Step A — Safe auto-fixers (run first; never counts against a retry budget)

The deterministic formatters and `--fix` linters the project already uses — e.g. `ruff format` +
`ruff check --fix`, `prettier --write`, `eslint --fix`, `gofmt -w`, `cargo fmt`. Never introduce a
tool the project doesn't use. Phase 5 skips this step: it audits the tree and must not mutate it.

## Step B — Gates (a failure here is a real failure)

Everything that fails the PR in CI and can run locally, typically:

- Tests — the tier CI runs on pull requests if that's practical locally; otherwise the fast/unit
  tier, noting what was left out
- Lint, and the formatter in check mode (`--check`)
- Type checking
- Complexity or coverage thresholds, if the project enforces them

## Step C — PR-only gates (manual self-check)

Some gates exist only on the PR — hosted code-health and quality services (CodeScene, SonarCloud,
Codacy, Code Climate), coverage-delta bots, and similar. Find them in the CI config, the project
instructions, and the PR's check runs (per `_github.md` § Operation map, *PR check runs*). There's
no local CLI, so review each changed file against the rules they enforce. The usual offenders:

- **Duplication** — near-identical blocks, especially in tests. Fix with parametrization or shared
  fixtures/builders/constants, not copy-paste.
- **Complex or deeply nested functions** — flatten with early returns and extract helpers.
- **Long parameter lists** — group related arguments or split the function.

A change that introduces one of these should be reworked even if Step B is green — it will bounce
in CI. If the project has no such gate, Step C is a no-op.

## Failure signatures

Every Step B failure gets a **signature**: a short, stable fingerprint that survives re-runs.

- Test → `{test id} {exception type}`, e.g. `test_backoff.py::test_ceiling AssertionError`
- Lint → `{rule} {file}:{line}`, e.g. `E501 src/client.py:88`
- Type → `{file}:{line} {error code}`
- Build → first error line, path included

Signatures exist so two attempts can be compared mechanically. Phase 4's stop condition is "the
same gates failing with the same signatures as last attempt" — that needs a comparable string, not
a paragraph of prose. Exclude timestamps, durations, absolute paths, and run IDs: anything that
changes between two identical runs makes every attempt look like progress.

Record signatures in `state.json` → `gates.*`.

## The baseline run

**Phase 4 runs the Step B gates once before applying any fix** and records the result in
`state.json` → `gates.baseline`.

This is what separates "my fix broke it" from "it was already broken." Without it, a red gate in
untouched code is a judgment call — and a wrong call either burns the retry budget on someone
else's bug or hides a real regression behind "probably pre-existing."

With it, both Phase 4 and Phase 5 compute:

- **failing now** − **failing at baseline** = regressions introduced by the fixes. These are yours.
- **failing at both** (same signature) = pre-existing. Note it, don't fix it, don't count it
  against the retry budget, and don't let it block a PASSED verdict.
- **failing at baseline but not now** = your fixes incidentally repaired something. Worth a line in
  `changes.md`.

Phase 5 compares against the same baseline, so implementer and reviewer agree on what "green"
meant. The baseline costs one suite run; it is the cheapest correctness win in the pipeline.

## Interpreting results

- **No regressions vs. baseline AND no Step C concern** → verification PASSED.
- **Any Step B gate failing that was green at baseline** → verification FAILED for that check; see
  the calling phase (Phase 4 enters its retry loop; Phase 5 records it as a finding).
- **A gate red at baseline and still red** → reported as pre-existing, never as a pass. It does not
  fail the phase, but it appears in `changes.md` and `verdict.md` so nobody mistakes the tree for
  clean.
