# pr-fix

A [Claude Code](https://claude.com/claude-code) skill that takes a GitHub pull request from review
feedback to verified fixes in five phases, with a human checkpoint at each step and a fresh chat
wherever an independent look matters.

```
Phase 1 (report)  →  Phase 2 (plan)  →  Phase 3 (review-plan)  →  Phase 4 (implement)  →  Phase 5 (review-impl)
   Fresh Chat A        Fresh Chat B          Fresh Chat C            Chat B (resumed)         Fresh Chat D
```

| Phase | Command | What it does | Writes |
|---|---|---|---|
| 1 | `/pr-fix report #123` | Reads the PR's review history and diff and writes an advisory issue report. | `context.json`, `report.md` |
| 2 | `/pr-fix plan` | Turns Medium+ findings and every blocking-review point into concrete fixes. | `plan.md` |
| 3 | `/pr-fix review-plan` | Checks each fix against the real code in a fresh chat; you ratify every skip. | `plan-approved.md` |
| 4 | `/pr-fix implement` | Applies the approved fixes in the planning chat, then runs the project's checks (at most 3 attempts). | `changes.md` |
| 5 | `/pr-fix review-impl` | Audits the diff in a fresh chat, re-runs the checks, and reverts rejected fixes without touching the rest. | `verdict.md` |
| — | `/pr-fix clean` | Wipes `.pr-fix/` to start over. | — |

## How it works

- **Files are the message bus.** Phases talk through `.pr-fix/` in the repo root, so each can run
  in its own chat. Phase 1 adds `.pr-fix/` to `.git/info/exclude`, so it never lands in a commit.
- **Fresh chats for review.** Phases 1, 3 and 5 start clean so they aren't anchored by the reasoning
  they check. Phases 2 and 4 share a chat so the planner implements its own plan.
- **Handoff prompts.** Each phase ends by printing the next command plus a ~10-sentence brief —
  decisions you made in that chat, where it's unsure, what's coupled. Paste it into the next chat
  as-is. Briefs into the review phases only point at risk; they never vouch for a fix.
- **Blocking reviews are covered in full.** Every point of every open `CHANGES_REQUESTED` review —
  human or bot — maps to a finding and gets either a fix or an explicit, justified skip.
- **Guarded against drift.** Phase 1 pins the PR's head SHA and later phases re-check it; Phase 4
  records a baseline SHA so the reviewed diff is exactly the fix set. Nothing is committed or
  pushed for you.

## Requirements

- Claude Code
- A GitHub repository, and the [GitHub CLI](https://cli.github.com/) authenticated (`gh auth login`)
- The PR checked out locally before Phase 4 (`gh pr checkout 123`)

## Install

pr-fix is free to use, but only with permission — see [License](#license). Once your request is
approved, install it for every project:

```bash
git clone https://github.com/EgeCankaya/pr-fix.git ~/.claude/skills/pr-fix
```

For one project, clone into that repo's `.claude/skills/pr-fix` instead. Update with `git pull`.

## Verification

Phases 2, 4 and 5 run the target project's own checks. `Workflows/_verify.md` finds them: an
override file if the repo has one; otherwise CI decides *what* must pass, `CLAUDE.md` /
`AGENTS.md` / `CONTRIBUTING.md` decide *how* to run it locally, and tool config (`package.json`,
`Makefile`, `pyproject.toml`, …) fills the gaps.

Discovery covers most repos. Add `.claude/pr-fix/verify.md` to the target repo when the checks need
special setup, or when CI enforces a hosted gate that has no local CLI:

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

## License

pr-fix is source-available, not open source. You can read the code here, but using, copying,
modifying, or redistributing it requires permission first. Permission is free: open an issue at
[github.com/EgeCankaya/pr-fix/issues](https://github.com/EgeCankaya/pr-fix/issues) saying who you
are and how you plan to use it. The full terms are in [LICENSE](LICENSE).
