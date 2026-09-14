# Run Configuration (shared)

The pipeline's scope knobs. Defaults live here; the target repo overrides them in
`.claude/pr-fix/config.md`, next to the existing `.claude/pr-fix/verify.md`. Phase 1 resolves the
config once and records the result in `state.json` → `config`, so every later phase reads the same
settings without re-deriving them.

A per-run override on the command line beats the file, and the file beats these defaults.

## Settings

| Key | Default | Meaning |
|---|---|---|
| `min_severity` | `medium` | Lowest severity Phase 2 turns into a fix. One of `critical`, `high`, `medium`, `low`, `info`. Blocking-review points ignore this entirely — they are always planned. |
| `include_own_findings` | `true` | When `false`, Phase 1 still reports its own analysis but Phase 2 plans **only** blocking-review points and open review comments. Use this when you want to clear a review, not improve the PR. |
| `skip_categories` | `[]` | Finding categories Phase 2 won't plan, e.g. `["performance", "code-quality"]`. Valid: `bug`, `security`, `convention`, `test-gap`, `performance`, `code-quality`, `compatibility`. |
| `max_retry_attempts` | `3` | Ceiling on Phase 4 verification attempts. The no-progress rule usually stops earlier. |
| `full_read_file_budget` | `25` | Above this many changed files, Phases 1 and 5 switch from reading every changed file in full to the triage strategy below. |
| `auto_checkpoints` | `true` | Under `/pr-fix run`, pause for human confirmation at the three checkpoints. `false` runs unattended and defers every decision to the recommended default. |

## Per-run overrides

Any subcommand accepts trailing `--key=value` flags, parsed before the handoff brief:

```
/pr-fix report #9 --min-severity=high --skip-categories=performance,code-quality
/pr-fix run #9 --include-own-findings=false
/pr-fix run #9 --auto-checkpoints=false
```

Record overrides in `state.json` → `config` with the rest, so a later phase in another session
honors them without being told again.

## Override file format

`.claude/pr-fix/config.md` in the target repo:

````markdown
# pr-fix config

```yaml
min_severity: high
include_own_findings: true
skip_categories:
  - performance
full_read_file_budget: 40
```

## Notes

Anything outside the YAML block is guidance for the run, read by Phases 1 and 2 —
e.g. "this repo vendors `third_party/`; never plan fixes there."
````

The prose after the YAML block is honored as scope guidance. Keep it short and factual; it is not
a place to redirect the pipeline's behavior.

## Severity and the blocking rule

`min_severity` filters **findings the pipeline discovered on its own**. It never filters a point
raised by a current blocking review. If a reviewer flagged something and labelled it trivial, it
still gets planned, because the `CHANGES_REQUESTED` doesn't clear until it is addressed or
explicitly justified. This is the one place the pipeline deliberately ignores its own severity
model — see `_schema.md` § `blocking_coverage[].disposition`.
