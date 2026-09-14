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

Record the exact commands and where they came from in the phase's output (`plan.md`, `changes.md`,
`verdict.md`), so the phases can compare like with like.

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
instructions, and the PR's check runs (`gh pr checks {PR_NUMBER}`). There's no local CLI, so review
each changed file against the rules they enforce. The usual offenders:

- **Duplication** — near-identical blocks, especially in tests. Fix with parametrization or shared
  fixtures/builders/constants, not copy-paste.
- **Complex or deeply nested functions** — flatten with early returns and extract helpers.
- **Long parameter lists** — group related arguments or split the function.

A change that introduces one of these should be reworked even if Step B is green — it will bounce
in CI. If the project has no such gate, Step C is a no-op.

## Interpreting results

- **All of Step B green AND no Step C concern** → verification PASSED.
- **Any Step B command non-zero** → verification FAILED for that check; see the calling phase for
  how to handle it (Phase 4 enters its bounded retry loop; Phase 5 records it as a finding).
