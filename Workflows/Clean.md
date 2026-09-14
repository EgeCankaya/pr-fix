# Clean — Wipe Pipeline State

Remove all `.pr-fix/` artifacts to start a fresh pipeline run. Use this between reviewing different PRs or to reset a corrupted/stale pipeline state.

## Step 1 — Check Current State

1. Check if `.pr-fix/` exists and what's in it:
   ```bash
   ls -la .pr-fix/ 2>/dev/null
   ```

2. If `.pr-fix/` does not exist or is empty:
   > ℹ️ Nothing to clean — `.pr-fix/` is already empty or doesn't exist.
   
   Stop here.

3. If `.pr-fix/` has files, read `context.json` to identify which PR run this belongs to:
   ```bash
   cat .pr-fix/context.json 2>/dev/null
   ```

## Step 2 — Show What Will Be Deleted

4. Show the user what artifacts exist and which pipeline phases they came from:

   ```
   Current .pr-fix/ state (PR #{NUMBER}):
   
   ✓ context.json      — PR metadata + head_sha (Phase 1)
   ✓ report.md         — Issue report (Phase 1)
   ✓ plan.md           — Fix plan (Phase 2)
   ✗ plan-approved.md  — (not yet created — Phase 3 not run)
   ✗ baseline_sha.txt  — (not yet created — Phase 4 not run)
   ✗ changes.md        — (not yet created — Phase 4 not run)
   ✗ verdict.md        — (not yet created — Phase 5 not run)
   ```

   Use ✓ for files that exist, ✗ for files that don't.

## Step 3 — Confirm and Delete

5. Ask the user to confirm:
   > **Delete all `.pr-fix/` artifacts?** This cannot be undone.
   > 
   > You'll need to re-run the pipeline from `/pr-fix report` after cleaning.

6. If confirmed, wipe everything:
   ```bash
   rm -rf .pr-fix
   ```

7. Confirm deletion:
   > ✅ `.pr-fix/` has been removed. You can start a fresh pipeline run with:
   > ```
   > /pr-fix report #PR
   > ```
