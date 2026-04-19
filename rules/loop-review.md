# Sprint Review Flow -- "prep for review", "batch fix", "ship clean"

The human review layer between swarm output and production. Uses the mega PR as the review surface.

## Pipeline (updated)

```
"launch swarm" -> Swarm ships PRs -> Sprint complete
  -> "prep for review" -> Draft mega PR with review table + testing plan
  -> You comment on mega PR (fix/revert notes)
  -> "batch fix" -> Swarm processes your notes
  -> "ship clean" or merge manually -> Production
```

## "prep for review"

Opens a **draft** mega PR (`YOUR_DEV_BRANCH` -> `main`) formatted for quick human review with step-by-step testing instructions.

### Steps

1. Sync `YOUR_DEV_BRANCH` with `main`: `git fetch origin && git merge origin/main`
2. Run full test suite. If failures, fix before proceeding.
3. Collect sub-PRs: `git log main..YOUR_DEV_BRANCH --merges --oneline`
4. **Filter out other people's PRs**: Only include PRs opened by you or the swarm (your-github-account account). Skip PRs from the CTO or anyone else that were merged directly to `main` and picked up via sync. Use `gh pr view <num> --json author` to check.
5. **Pull testing plans from sub-PRs**: Each sub-PR should already have a "How to test" section (written by the Sprint Worker). Use `gh pr view <num> --json body` to extract it. If a PR is missing testing steps, generate them by reading the diff.
6. **Order logically**: If PRs depend on each other (e.g., onboarding changes before features that assume onboarding), order those first. Group related PRs (e.g., all Agent Mode changes together).
7. Open **draft** mega PR: `gh pr create --base main --head YOUR_DEV_BRANCH --draft`

### Mega PR description format

```markdown
## Sprint Review: {date} -- {count} PRs

**How to review**: Go through the testing plan below. Comment on this PR with fix/revert notes.
Anything you don't mention = approved to ship.

**Comment format**:
- `fix #NNN: what's wrong` -- swarm will fix it
- `revert #NNN` -- will be removed before shipping

---

### PR #NNN: {title}
**Risk**: Low | **Type**: feat | **Files**: 3

**What changed**: {2-3 sentence description of what this does and why}

**How to test**:
1. Go to {page/route}
2. {Do this specific action}
3. **Expected**: {What you should see}
4. {Next step}
5. **Expected**: {What you should see}

**Edge case to check**: {One thing the automated tests didn't cover}

---

### PR #NNN: {title}
...

---

### High-Risk Items -- Test these first
- **#NNN -- {title}**: {why risky}. Test plan above covers it.
  Rollback: `git revert <sha>` on YOUR_DEV_BRANCH.

### Test Status
- Full suite: {X} passed, 0 failed
- All sub-PRs passed Senior Reviewer + Docs gates

### What was NOT tested (aggregate)
{Honest list across all PRs}
```

**Key rules for the testing plan**:
- Each PR gets its OWN section with step-by-step instructions
- Steps must be concrete: specific URLs, buttons to click, text to look for
- Include what the user should see at each step ("Expected: ...")
- For admin-only features: specify which account to use
- For UI changes: describe what the visual change looks like
- For behavioral changes: describe the before vs. after behavior
- Keep it scannable -- if a PR is trivial (CSS-only, copy change), the testing plan can be 2 lines

7. Post the mega PR URL. Tell the user to review and comment when ready.
8. **Stop and wait.** Do not proceed until the user says `batch fix` or `ship clean`.

## "batch fix"

Reads the user's comments on the mega PR and processes all fix/revert requests.

### Steps

1. Fetch all comments on the mega PR: `gh api repos/{owner}/{repo}/issues/{num}/comments`
2. Parse each comment for directives:
   - `fix #NNN: description` -> collect as fix task
   - `revert #NNN` or `revert #NNN: reason` -> collect as revert task
   - Ignore comments from bots or that don't match the pattern
3. **Process reverts first** (order matters -- reverts change the branch state):
   - For each revert: find the merge commit for that PR on `YOUR_DEV_BRANCH`
   - `git revert <merge-commit> --no-edit` on `YOUR_DEV_BRANCH`
   - Push. Comment on mega PR: "Reverted #NNN from YOUR_DEV_BRANCH."
4. **Process fixes in parallel** (use worktrees):
   - For each fix: create a worktree, branch from `YOUR_DEV_BRANCH`
   - Read the original PR diff to understand what was built
   - Apply the fix based on the user's description
   - Write/update tests if the fix changes behavior
   - `npm test` must pass
   - Open fix PR targeting `YOUR_DEV_BRANCH` with format:

   ```
   ## Fix for #NNN: {original title}

   **User feedback**: {their comment}

   **What changed**: {description of fix}

   **Test evidence**: X passed, 0 failed
   ```

   - Fast-track: since the user already reviewed and requested the fix, auto-merge after tests pass. Skip Senior Reviewer + Docs gates.
5. **Update mega PR description** with verdicts:

   Add a verdict column to each PR section header:
   ```
   ### PR #NNN: {title} — ✅ Ship
   ### PR #NNN: {title} — 🔧 Fixed (#441)
   ### PR #NNN: {title} — ⏪ Reverted
   ```

6. Report summary: "X fixes shipped, Y reverted. Mega PR ready to merge."
7. Convert draft to ready: `gh pr ready {num}`

## "ship clean"

For when you want to ship what's ready NOW without waiting for fixes.

### Steps

1. Read mega PR comments (same parsing as `batch fix`).
2. Revert all `revert` AND `fix` items from `YOUR_DEV_BRANCH` (fixes haven't been made yet).
3. Update mega PR description marking reverted/held items.
4. Run tests. Must pass.
5. Convert draft to ready: `gh pr ready {num}`
6. Report: "Mega PR ready with {X} of {Y} PRs. {Z} items held for next sprint."
7. The held items stay on a `held/{date}` branch for the swarm to fix in the next sprint.

## Rules

- **Only YOUR PRs**: The review table must only include PRs authored by you (your-github-account) or the swarm. Filter out the CTO's or anyone else's direct-to-main merges that show up in the diff.
- **Draft = safe**: Draft PRs can't be accidentally merged. Always start as draft.
- **Silent approval**: No comment on a PR = approved. Only flag problems.
- **One review surface**: All feedback goes on the mega PR. Not on individual sub-PRs.
- **Fast-track fixes**: Fix PRs requested by the user skip the full Senior Reviewer + Docs pipeline. User already reviewed. Just run tests and merge.
- **Revert before fix**: Always process reverts before fixes to avoid conflicts.
- **Never auto-merge to main**: Even after `batch fix`, the user clicks merge manually.
- **Concrete testing steps**: Every PR section must have a "How to test" with numbered steps and expected results. No vague instructions like "verify it works."
