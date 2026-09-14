# pr-fix

A [Claude Code](https://claude.com/claude-code) skill that takes a GitHub pull request from review
feedback to verified fixes — with a human checkpoint at each step and fresh context wherever an
independent look matters.

```bash
/pr-fix run #123
```

One session drives the whole pipeline, spawning subagents for the phases that need fresh eyes:

```
Phase 1 (report) → Phase 2 (plan) → Phase 3 (review-plan) → Phase 4 (implement) → Phase 5 (review-impl) → Phase 6 (respond)
    subagent         this session      subagent                this session          subagent               this session
        └─── checkpoint ───┘                └──── checkpoint ────┘                       └─ checkpoint ─┘
```

| Phase | Command | What it does | Writes |
|---|---|---|---|
| 1 | `/pr-fix report #123` | Reads the PR's review history and diff, writes an advisory issue report. | `state.json`, `report.md` |
| 2 | `/pr-fix plan` | Turns findings at or above the severity floor — and every blocking-review point — into concrete fixes. | `plan.md`, `verify-resolved.md` |
| 3 | `/pr-fix review-plan` | Checks each fix against the real code with fresh context; you ratify every skip. | `plan-approved.md` |
| 4 | `/pr-fix implement` | Applies the approved fixes, then runs the project's checks. | `changes.md`, `patches/` |
| 5 | `/pr-fix review-impl` | Audits the diff with fresh context, re-runs the checks, reverts rejected fixes. | `verdict.md` |
| 6 | `/pr-fix respond` | Drafts the commit message and the reviewer reply. Optional; posts nothing without a yes. | `response.md` |
| — | `/pr-fix status` | Where the pipeline stands and what to run next. Read-only. | — |
| — | `/pr-fix clean` | Wipes `.pr-fix/` to start over. | — |

Each phase also works standalone, if you'd rather read and edit the artifacts between steps. Run
`/pr-fix report #123` and follow the handoff prompt each phase prints.

## How it works

- **Files are the message bus.** Phases talk through `.pr-fix/` in the repo root, so each can run
  in its own session — or its own subagent. Phase 1 adds `.pr-fix/` to `.git/info/exclude`, so it
  never lands in a commit.
- **Fresh context for review.** Phases 1, 3 and 5 start clean so they aren't anchored by the
  reasoning they're checking. Phases 2 and 4 share a session so the planner implements its own plan.
- **Handoff briefs.** Each phase hands the next a ~10-sentence brief — decisions you made, where
  it's unsure, what's coupled. Under `run` the orchestrator passes it to the subagent; in manual
  mode you paste it. Briefs into the review phases only point at risk; they never vouch for a fix.
- **Blocking reviews are covered in full.** Every point of every open `CHANGES_REQUESTED` review —
  human or bot — maps to a finding and gets either a fix or an explicit, justified skip. Phase 3
  re-derives that map from the live reviews rather than trusting Phase 1's.
- **Pre-existing failures are measured, not guessed.** Phase 4 runs the project's gates *before*
  applying anything, so "that was already broken" is a set difference both it and Phase 5 compute
  from the same baseline.
- **One fix, one patch.** Phase 4 writes an incremental patch per fix, so Phase 5 can reverse
  exactly one with `git apply -R` instead of hand-editing a file.
- **Guarded against drift.** Every phase re-checks the PR's head SHA before acting; Phase 4 records
  a baseline commit so the reviewed diff is exactly the fix set.
- **Nothing is committed or pushed for you.** Ever, in any mode.

## Requirements

- Claude Code, and a git repository
- A way to reach GitHub — **any one of**:
  - the [GitHub CLI](https://cli.github.com/), authenticated (`gh auth login`)
  - the GitHub MCP server (`mcp__github__*` tools), which is what Claude Code on the web and most
    remote sessions provide
  - plain `git` alone, which gets you the diff but **not** the review history — a degraded,
    code-review-only run that the pipeline labels as such

  The skill resolves this itself; you don't configure it. See `references/_github.md`.
- The PR checked out locally before Phase 4 (`gh pr checkout 123`, or
  `git fetch origin pull/123/head:branch && git switch branch`)

## Install

pr-fix is free to use, but only with permission — see [License](#license). Once your request is
approved, install it for every project:

```bash
git clone https://github.com/EgeCankaya/pr-fix.git ~/.claude/skills/pr-fix
```

For one project, clone into that repo's `.claude/skills/pr-fix` instead. Update with `git pull`.

## Configuration

Both override files live in the **target** repo (the one holding the PR), under `.claude/pr-fix/`.
Neither is required — the skill discovers sensible defaults for both.

### `verify.md` — the project's checks

Phases 2, 4 and 5 run the target project's own checks. `references/_verify.md` finds them: this
override file if present; otherwise CI decides *what* must pass, `CLAUDE.md` / `AGENTS.md` /
`CONTRIBUTING.md` decide *how* to run it locally, and tool config (`package.json`, `Makefile`,
`pyproject.toml`, …) fills the gaps.

Discovery covers most repos. Add the override when the checks need special setup, or when CI
enforces a hosted gate with no local CLI:

````markdown
# pr-fix verification

## Step A — auto-fixers
```bash
npm run format
npm run lint -- --fix
```

## Step B — gates
```bash
npm test
npm run lint
npm run typecheck
```

## Step C — PR-only gates
CodeScene's delta check fails any changed file scoring below 10.0. Watch for duplicated test
blocks and conditionals nested more than two deep.
````

### `config.md` — scope

````markdown
# pr-fix config

```yaml
min_severity: high          # only plan fixes this severe or worse
include_own_findings: true  # false = only reviewer-raised points
skip_categories: [performance]
full_read_file_budget: 40   # above this many changed files, triage instead of reading all
```

## Notes

This repo vendors `third_party/`; never plan fixes there.
````

Anything can also be set per run: `/pr-fix run #123 --min-severity=high`.

One rule overrides all of this: **a blocking-review point is always planned**, whatever its
severity. Leaving it unaddressed is what keeps the `CHANGES_REQUESTED` from clearing.

## Examples

`examples/` holds a complete worked run against a small Python PR — the real `report.md`,
`plan.md`, `verdict.md` and `state.json` a run produces. Useful for seeing the output shape before
you install, and as a reference for what "good" looks like.

## License

pr-fix is source-available, not open source. You can read the code here, but using, copying,
modifying, or redistributing it requires permission first. Permission is free: open an issue at
[github.com/EgeCankaya/pr-fix/issues](https://github.com/EgeCankaya/pr-fix/issues) saying who you
are and how you plan to use it. The full terms are in [LICENSE](LICENSE).
