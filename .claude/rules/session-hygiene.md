# Rule 11: Context & Session Hygiene

- Keep each CLAUDE.md under 200 lines; use `.claude/rules/` directory for overflow
- Use `/compact` at 50% context capacity -- don't wait until forced
- Use `/clear` when switching between unrelated tasks
- Commit at least hourly; commit immediately when a task completes
- Use screenshots and Playwright MCP for autonomous browser debugging
- Use permission wildcards (e.g., `Bash(npm run *)`) over dangerously-skip-permissions
- Use `/rename` to label important sessions for later `/resume`
- Use `Esc Esc` or `/rewind` to undo mistakes mid-conversation
- Review `lessons.md` at session start for patterns relevant to current project
- One task per subagent -- keep main context window clean for orchestration
- **Worktree pruning**: After a mega PR or at the end of any multi-PR session, prune stale worktrees:
  1. `git worktree list` -- identify worktrees whose branches are already merged
  2. Remove deepest-nested first: `git worktree remove --force <path>` (children before parents)
  3. `git worktree prune` -- clean up stale metadata
  4. Stale worktrees cause: slow test runs (vitest picks up their test files), disk bloat, branch lock conflicts
  5. The PR Watchdog should prune merged-branch worktrees every tick
