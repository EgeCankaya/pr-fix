# Pipeline State (shared)

`.pr-fix/state.json` is the pipeline's single machine-readable file. Every phase reads it at the
start and updates it at the end.

It replaces the older `context.json` and `baseline_sha.txt`. The prose artifacts (`report.md`,
`plan.md`, `plan-approved.md`, `changes.md`, `verdict.md`) are still written — they are what the
human reads, and they hold the reasoning. `state.json` holds the *facts the next phase must not
get wrong*: IDs, severities, verdicts, coverage, gate results.

Why it exists: the pipeline's central guarantee is that every point of every blocking review ends
up either fixed or explicitly justified. That guarantee was previously asserted three times over by
re-reading markdown tables in three different sessions. Asserting it over a list of objects makes
it checkable instead of hopeful.

## Artifact layout

```
.pr-fix/
  state.json            # this file's schema — all phases read/write
  report.md             # Phase 1, human-readable
  plan.md               # Phase 2
  plan-approved.md      # Phase 3
  changes.md            # Phase 4  (carries verification_status; there is no changes-failed.md)
  verdict.md            # Phase 5
  response.md           # Phase 6, draft commit message + PR comment
  verify-resolved.md    # Phase 2, the resolved verification suite
  patches/fix-NN.patch  # Phase 4, one incremental patch per applied fix
```

## Schema

```json
{
  "schema_version": 1,
  "context": {
    "pr_number": 9,
    "repo": "owner/repo",
    "url": "https://github.com/owner/repo/pull/9",
    "title": "Add retry backoff to the ingest client",
    "base_branch": "main",
    "head_branch": "feature/retry-backoff",
    "head_sha": "a1b2c3d4...",
    "state": "OPEN",
    "review_decision": "CHANGES_REQUESTED"
  },
  "github": {
    "access_path": "gh",
    "review_history_available": true,
    "fetch_gaps": []
  },
  "config": {
    "min_severity": "medium",
    "include_own_findings": true,
    "skip_categories": [],
    "max_retry_attempts": 3,
    "full_read_file_budget": 25
  },
  "phases": {
    "report": { "status": "complete", "at": "2026-09-14T10:02:00Z", "head_sha_seen": "a1b2c3d4", "mode": "subagent" },
    "plan": { "status": "complete", "at": "...", "head_sha_seen": "a1b2c3d4", "mode": "interactive" },
    "review-plan": { "status": "pending" },
    "implement": { "status": "pending" },
    "review-impl": { "status": "pending" },
    "respond": { "status": "pending" }
  },
  "baseline_sha": null,
  "findings": [
    {
      "id": "F-01",
      "title": "Backoff overflows on the 31st retry",
      "file": "src/ingest/client.py",
      "lines": "42-55",
      "severity": "High",
      "category": "Bug",
      "confidence": "High",
      "source": "analysis",
      "status": "open"
    }
  ],
  "blocking_coverage": [
    {
      "review_id": 2481003,
      "reviewer": "@octocat",
      "point": "The retry ceiling is unbounded",
      "finding_ids": ["F-01"],
      "disposition": "fix"
    }
  ],
  "fixes": [
    {
      "id": "FIX-01",
      "addresses": ["F-01"],
      "title": "Clamp the backoff exponent",
      "file": "src/ingest/client.py",
      "severity": "High",
      "depends_on": [],
      "review_verdict": "approve",
      "review_note": "",
      "applied": "applied",
      "patch": "patches/fix-01.patch",
      "impl_verdict": "good",
      "action_taken": "kept"
    }
  ],
  "skips": [
    {
      "finding_id": "F-05",
      "reason": "Low severity, cosmetic",
      "decision": "keep-skipped",
      "ratified_by_user": true
    }
  ],
  "pending_decisions": [
    {
      "id": "PD-01",
      "phase": "review-plan",
      "question": "F-09 is a blocking point with no planned fix — include it?",
      "options": ["Include as a new fix", "Keep skipped as false-positive"],
      "defaulted_to": "Include as a new fix",
      "resolved_by_user": false
    }
  ],
  "gates": {
    "resolved_from": "ci+instructions+config",
    "baseline": [
      { "gate": "tests", "command": "make test", "result": "pass", "signature": null },
      { "gate": "types", "command": "mypy src", "result": "fail", "signature": "src/legacy.py:88 assignment" }
    ],
    "implement": {
      "attempts": [
        { "n": 1, "failing": ["tests"], "signature": "test_backoff.py::test_ceiling AssertionError", "action": "clamped exponent" },
        { "n": 2, "failing": [], "signature": null, "action": null }
      ],
      "final": "PASSED"
    },
    "review": [
      { "gate": "tests", "command": "make test", "result": "pass", "signature": null }
    ]
  }
}
```

## Field notes

**`config`** — defaults, overridden by `.claude/pr-fix/config.md` in the target repo. See
`_config.md`.

**`phases.<name>.status`** — `pending` | `complete` | `failed`. `mode` is `interactive` (a human is
in this session) or `subagent` (running under `/pr-fix run`, no human reachable). Phases branch on
`mode` where they would otherwise call `AskUserQuestion`.

**`findings[].source`** — `analysis` (the report found it) or `review` (a reviewer raised it).
`status` is `open` | `resolved` (already fixed on the current head) | `false-positive`.

**`blocking_coverage[].disposition`** — `fix` | `already-resolved` | `false-positive`. These are
the **only** three legal values. "Low severity" is not a disposition: a blocking point may not be
skipped on severity grounds, because leaving it unaddressed is exactly what keeps the PR's
`CHANGES_REQUESTED` from clearing. Every entry must carry at least one `finding_ids` member.

**`fixes[].review_verdict`** — `approve` | `revise` | `reject`, set by Phase 3. `applied` —
`applied` | `skipped` | `failed`, set by Phase 4. `impl_verdict` — `good` | `needs-work` |
`revert`, set by Phase 5. Phase 3 updates fixes **in place**; there is no separate approved-plan
JSON.

**`pending_decisions[]`** — questions a phase would have put to the user but could not, because
it ran as a subagent with no human reachable. The phase records the question, the options, and the
default it applied so the pipeline could continue, then leads its final report with the list. The
orchestrator surfaces them at the next checkpoint and sets `resolved_by_user`. An entry left
unresolved when the pipeline finishes is a decision nobody made — `/pr-fix status` reports those.

**`gates.baseline`** — the Step B gates run by Phase 4 *before* any fix is applied. This is what
makes "pre-existing failure" a set difference rather than a judgment call. `signature` is a short
stable fingerprint of the failure (test id + assertion, or rule + file:line) so two attempts can be
compared for progress.

**`gates.implement.attempts`** — drives the no-progress stop condition. See `implement.md`.

## Updating it safely

Read, modify, write whole — never append or hand-patch JSON. Preferred idiom:

```bash
python3 - <<'PY'
import json, pathlib
p = pathlib.Path(".pr-fix/state.json")
s = json.loads(p.read_text())
s["phases"]["plan"] = {"status": "complete", "at": "...", "head_sha_seen": "...", "mode": "interactive"}
p.write_text(json.dumps(s, indent=2) + "\n")
PY
```

If `state.json` is missing or unparseable, say so and point the user at `/pr-fix status` — do not
reconstruct it by guessing from the markdown artifacts. The one exception is a run started before
this schema existed: if `context.json` is present and `state.json` is not, migrate the old fields
into a fresh `state.json` and tell the user you did.
