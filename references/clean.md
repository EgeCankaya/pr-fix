# Clean — Wipe pipeline state

Remove all `.pr-fix/` artifacts to start a fresh run. Use this between different PRs, or to reset a
corrupted or stale state.

To *inspect* state without deleting it, use `/pr-fix status` — that's the read-only command.

## Step: Check what's there

```bash
ls -la .pr-fix/ 2>/dev/null
```

If `.pr-fix/` doesn't exist or is empty:
> ℹ️ Nothing to clean — `.pr-fix/` is already empty or doesn't exist.

Stop there.

Otherwise read `.pr-fix/state.json` to identify which PR this run belongs to.

## Step: Show what will be deleted

```
Current .pr-fix/ state (PR #{N} — {TITLE}):

✓ state.json          — pipeline state (all phases)
✓ report.md           — issue report (Phase 1)
✓ plan.md             — fix plan (Phase 2)
✓ verify-resolved.md  — resolved verification suite (Phase 2)
✓ plan-approved.md    — approved fixes (Phase 3)
✗ changes.md          — (not yet created — Phase 4 not run)
✗ patches/            — (not yet created — Phase 4 not run)
✗ verdict.md          — (not yet created — Phase 5 not run)
✗ response.md         — (not yet created — Phase 6 not run)
```

Use ✓ for files that exist, ✗ for those that don't.

**Warn about uncommitted work.** If Phase 4 ran, the fixes are in the working tree, and the patches
about to be deleted are the only record of which change belonged to which fix:

```bash
git status --porcelain | grep -v '^?? \.pr-fix/' || true
```

If that's non-empty:

> ⚠️ Your working tree still holds the applied fixes, and `.pr-fix/patches/` is the only record of
> which change came from which fix. Cleaning won't touch your code, but afterwards you won't be
> able to revert a single fix — only edit by hand. Commit first if you want that history.

This is the one genuinely lossy thing `clean` does, so say it before asking.

## Step: Confirm and delete

> **Delete all `.pr-fix/` artifacts?** This cannot be undone. Your source changes are not touched.
> You'll need to re-run from `/pr-fix report` afterwards.

Only on an explicit yes:

```bash
rm -rf .pr-fix
```

Then:

> ✅ `.pr-fix/` removed. Your working tree is unchanged. Start a fresh run with:
> ```
> /pr-fix report #PR      (step by step)
> /pr-fix run #PR         (driven from one session)
> ```

The `.pr-fix/` entry added to `.git/info/exclude` by Phase 1 is left in place — it's harmless, and
the next run reuses it.
