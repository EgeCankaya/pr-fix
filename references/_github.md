# GitHub Access (shared)

Every phase that talks to GitHub resolves its access path **through this file**, once, at the start
of the phase. Do not hardcode `gh` in a workflow — the CLI is absent in many of the environments
where this skill is most useful (Claude Code on the web, remote sessions, CI runners), and a phase
that assumes it will fail on the first call.

Record which path you resolved in `state.json` (`github.access_path`) so later phases and the
verdict can say where the data came from.

## Resolve the access path (in this order)

### 1. `gh` CLI — preferred when present and authenticated

```bash
command -v gh >/dev/null 2>&1 && gh auth status >/dev/null 2>&1 && echo "gh ok"
```

Both must succeed. `command -v gh` alone is not enough: an unauthenticated `gh` fails every call
with an opaque error partway through a phase.

### 2. GitHub MCP tools — when `gh` is missing or unauthenticated

If tools named `mcp__github__*` are available in the session, use them. They cover everything the
pipeline needs. If they are listed as deferred, load them with `ToolSearch` before the first call.

### 3. Local git — last resort, degraded

With neither `gh` nor MCP, the diff is still reachable:

```bash
git fetch origin "pull/{PR_NUMBER}/head:pr-{PR_NUMBER}"
git diff "origin/{BASE_BRANCH}...pr-{PR_NUMBER}"
```

**Review history is not reachable this way.** Without it Phase 1 cannot build the blocking-review
coverage map, which is the pipeline's main guarantee. On this path:

- Set `github.access_path` to `"git-only"` and `github.review_history_available` to `false`.
- Say so at the top of `report.md`, in the handoff brief, and in the final verdict.
- Do **not** silently produce a report that looks complete. A run with no review history is a
  code-review run, not a PR-fix run.

If none of the three work, stop and tell the user what to install or authenticate. Never fabricate
PR metadata.

## Operation map

Resolve the row you need, then use the column for your access path.

| Operation | `gh` | MCP |
|---|---|---|
| PR metadata | `gh pr view {N} --json title,body,baseRefName,headRefName,headRefOid,state,number,url,reviewDecision,additions,deletions,changedFiles,files` | `mcp__github__pull_request_read` (method `get`) |
| PR diff | `gh pr diff {N}` | `mcp__github__pull_request_read` (method `get_diff`) |
| Changed files | included in metadata `files` | `mcp__github__pull_request_read` (method `get_files`) |
| Line-level review comments | `gh api repos/{OWNER}/{REPO}/pulls/{N}/comments --paginate` | `mcp__github__pull_request_read` (method `get_review_comments`) |
| Reviews (approve / changes-requested) | `gh api repos/{OWNER}/{REPO}/pulls/{N}/reviews --paginate` | `mcp__github__pull_request_read` (method `get_reviews`) |
| Conversation comments | `gh pr view {N} --json comments -q '.comments'` | `mcp__github__issue_read` (method `get_comments`) on the PR number |
| Live head SHA (staleness) | `gh pr view {N} --json headRefOid -q '.headRefOid'` | `mcp__github__pull_request_read` (method `get`) → `head.sha` |
| PR check runs | `gh pr checks {N}` | `mcp__github__pull_request_read` (method `get_status`) |
| CI job logs | `gh run view --log-failed` | `mcp__github__get_job_logs` (`failed_only: true`) |
| Repo slug | `gh repo view --json nameWithOwner -q '.nameWithOwner'` | parse `git remote get-url origin` |
| Post a PR comment (Phase 6 only) | `gh pr comment {N} --body-file {FILE}` | `mcp__github__add_issue_comment` |
| Resolve a review thread (Phase 6 only) | — (GraphQL; skip) | `mcp__github__resolve_review_thread` |

Pagination matters on both paths. A PR with a long review history will truncate to the first page
unless you paginate — `--paginate` on `gh`, or follow the page parameter on MCP until the result is
short. An under-fetched review list silently shrinks the blocking-review coverage map, which is the
worst failure this pipeline has.

## Checking out the PR

The pipeline's later phases edit the local tree, so it must hold the PR's code.

- With `gh`: `gh pr checkout {N}`
- Without: `git fetch origin pull/{N}/head:{HEAD_BRANCH} && git switch {HEAD_BRANCH}`

Never check out on the user's behalf without asking — they may have uncommitted work.

## Error handling

- **Rate limited** — note it, continue with what you have, and record the gap in
  `state.json` (`github.fetch_gaps[]`). Every phase that reads a partial fetch must surface the gap
  rather than treating absence as evidence.
- **PR not found** — report it plainly; do not guess a different PR number.
- **403 on a private repo** — the token lacks access. Say which path you tried so the user knows
  whether to fix `gh auth` or the MCP connection.
