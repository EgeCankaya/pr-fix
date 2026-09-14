# Staleness Check (shared)

**Every phase runs this before it does anything else** (after resolving `_github.md`, before
reading artifacts). The pipeline spans several sessions and can span hours; a reviewer pushing a
commit mid-run invalidates whatever came before, and the failure is silent — a plan whose line
numbers moved, or a fix applied to code that no longer exists.

Previously only Phase 3 checked this. It belongs in all of them.

## The check

```bash
LOCAL_HEAD=$(git rev-parse HEAD)
LOCAL_BRANCH=$(git rev-parse --abbrev-ref HEAD)
```

Fetch the live head SHA per `_github.md` § Operation map (*Live head SHA*), and compare three
values:

| Value | Where from | Question it answers |
|---|---|---|
| `state.json` → `context.head_sha` | pinned by Phase 1 | what the pipeline was planned against |
| live head SHA | GitHub, now | has the PR moved? |
| `LOCAL_HEAD` | working tree | am I looking at the right code? |

## Outcomes

**All three match** → proceed. Record the observed SHA in
`state.json` → `phases.<phase>.head_sha_seen`.

**Live ≠ pinned** → the PR advanced after Phase 1. Everything downstream may be stale.

- Phases 2, 3 (planning, still cheap to redo): warn and **recommend re-running `/pr-fix report`**.
  Proceed only if the user says to.
- Phase 4 (about to edit files): **stop.** Do not apply fixes against a plan written for different
  code. Tell the user what moved and offer `/pr-fix report` to refresh.
- Phase 5 (auditing): proceed — you are reviewing the tree as it is — but record the drift in
  `verdict.md` under Remaining Concerns, because the fixes were designed against an older head.

Report the drift concretely, not as a generic warning:

> ⚠️ PR #{N} has advanced since the report: pinned `{PINNED:0:8}`, now `{LIVE:0:8}`.
> {M} commits landed, touching {FILES}. {Findings/fixes} in those files may no longer apply.

Get the commit list with `git log --oneline {PINNED}..{LIVE}` when the objects are local, or from
the PR's commit list per `_github.md` otherwise.

**`LOCAL_HEAD` ≠ pinned** → wrong code checked out. Phase 1 warns only (its report is built from
remote data and is valid either way). Phases 4 and 5 **stop**: they edit and diff the local tree.
Offer the checkout command from `_github.md` § Checking out the PR.

**Phase 4 iteration rounds are the documented exception.** On an iteration round, `LOCAL_HEAD`
still matches the pinned head but the tree is *dirty* with round-1 fixes — that is expected, not
drift. The handoff brief says so. Compare against `state.json` → `baseline_sha` rather than
treating the dirty tree as a failure.

## Offline / degraded

If the live SHA can't be fetched (rate limit, `git-only` access path), say the check was skipped
and why, record `head_sha_seen: null`, and continue. A skipped check is a caveat in the output —
never an assumed pass.
