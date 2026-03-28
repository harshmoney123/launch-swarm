# PR Operations -- Watchdog, Senior Reviewer, Digest

## PR Watchdog -- "watch prs" (`/loop 10m`)

Two jobs: (1) fix review comments, (2) auto-merge when both gates pass.

### Each tick

1. `gh pr list --state open` across repos. Include mega PRs (`YOUR_DEV_BRANCH` -> `main`) in all checks -- review comments, CI, conflicts -- but **never auto-merge them**.
2. **Sync `YOUR_DEV_BRANCH`**: `git fetch origin && git merge origin/main` into `YOUR_DEV_BRANCH`, push.
3. **Conflict resolution**: For PRs with merge conflicts, checkout in worktree, merge `origin/YOUR_DEV_BRANCH`, resolve, run tests, push.
4. **Fix CI failures**: Diagnose and push fix.
5. **Fix review comments (#1 job)**:
   - Fetch comments, filter unresolved Bug and Nit.
   - Fix order: priority x smallest diff. Bug before Nit.
   - Checkout branch, make minimal fix, run tests, commit, push, reply "Fixed."
   - Skip informational comments. Skip fixes needing >3 files. Revert if tests fail.
6. **Auto-merge (#2 job)**:
   - Check: Senior Reviewer "Approve" + Docs "Documented" + CI green + no unresolved bugs.
   - All met -> `gh pr merge <num> --merge --delete-branch`. Update Notion -> "Done".
   - **After merge -> redeploy test env**: First rsync standalone repos to parent: `rsync -a --delete --exclude='node_modules' --exclude='.git' --exclude='.claude' YOUR_FRONTEND_REPO_PATH/ YOUR_MONOREPO_PATH/customer-portal/`. Then: `cd YOUR_MONOREPO_PATH && ./test-env/test-env.sh sync`. Verify with `./test-env/test-env.sh status`.
   - Only Senior Approve, no Docs -> skip (Docs will pick it up).
   - **Never auto-merge PRs targeting `main`.**
7. **Mega PR feedback (#3 job -- CTO's comments)**:
   a. Check mega PR (`YOUR_DEV_BRANCH` -> `main`) for new comments from your CTO: `gh api repos/{owner}/{repo}/issues/{num}/comments`.
   b. Filter to unresolved comments (no "Fixed" reply from your bot account).
   c. For each bug/suggestion CTO reports:
      - Investigate the root cause (read code, reproduce if possible)
      - Branch from `YOUR_DEV_BRANCH`, fix, write tests, open PR targeting `YOUR_DEV_BRANCH`
      - Reply on the mega PR: "Fixed in PR #NNN -- [description]"
   d. After all CTO comments addressed: update the mega PR description with new sub-PR count and re-run test suite.
   e. **Priority**: CTO's mega PR feedback is HIGH priority -- it blocks the merge to `main`. Process before regular watchdog tasks.
8. **Feature Launch (#4 job)**: When a mega PR is detected as merged to `main`, trigger the Docs agent's Feature Launch flow (see `loop-docs.md`). Only fire once per mega PR -- track announced PRs to avoid duplicates.
9. **Notion auto-close on mega PR merge (#5 job)**:
   When a mega PR is detected as merged to `main`:
   a. Parse the mega PR description for the sub-PR table. Extract all PR numbers listed.
   b. For each sub-PR, fetch its description (`gh pr view <num>`). Extract any Notion task URLs (pattern: `notion.so/<id>`).
   c. Also cross-reference: `git log main..<previous-main-head>` commit messages for Notion links.
   d. For each Notion task found:
      - **Owner filter**: Only update your own or unassigned tasks. Skip CTO's.
      - Set `Status: "Done"` and `Completed Date` to today.
      - Add comment: "Shipped to main via mega PR #NNN on {date}."
   e. If a sub-PR has no Notion link, search the sprint board by task name keywords. If a confident match is found (exact or near-exact title), close it. If ambiguous, skip.
   f. Report summary: "{X} tasks auto-closed, {Y} skipped (no Notion link or ambiguous match)."
   g. Only fire once per mega PR -- same tracking as Feature Launch.
10. **Mega PR health**: Flag if `YOUR_DEV_BRANCH` has >20 commits or >7 days since last merge to `main`.
11. **Cross-repo linking**: Match PRs across backend <-> frontend repos, comment links.
12. **Worktree pruning**: Remove worktrees for merged branches. `git worktree prune`.
13. Flag stale PRs (>7 days). Report digest.

## Senior Reviewer -- "review prs" (`/loop 20m`)

Deep code review. Reads actual code, not screenshots.

### Each tick

1. `gh pr list --state open` -- filter to your PRs targeting `YOUR_DEV_BRANCH` AND mega PRs (`YOUR_DEV_BRANCH` -> `main`).
2. Priority order per `loop-modes.md`. Skip unchanged PRs. **Mega PRs get priority review** -- they block the merge to `main`.
3. **Deep read** (min 5 minutes):
   - Full diff + entire changed files + imports 2 levels deep + usage sites + test files.
4. **Evaluate**: Architecture, Reliability, Elegance, Security, Compatibility.
5. **Post comments** with severity: Bug, Nit, Pre-existing, Suggestion.
6. **Verdict**: Approve / Needs changes / Rethink approach.
7. Approve -> Docs agent picks up the PR next. Watchdog merges after both gates pass.
8. Auto-resolve old threads when fixes are verified.

## Daily Digest -- "pr digest" (cron `3 9 * * *`)

Pipeline status for team:
1. Tasks merged to `YOUR_DEV_BRANCH` since last mega PR
2. Mega PR readiness (commits, lines, high-risk items)
3. Pending PRs with review status

Deliver to Slack `YOUR_DIGEST_CHANNEL`.
