# Global Development Rules

Project-level CLAUDE.md takes precedence on conflicts.

## Git Workflow

**At session start**, run `git remote -v`: No remote -> work on current branch. Has remote -> use `EnterWorktree`, never commit directly to main.

- All PRs target **`YOUR_DEV_BRANCH`**, not `main`. Branch from `YOUR_DEV_BRANCH`.
- Two gates before auto-merge: Senior Reviewer + Docs. Watchdog merges after both pass.
- **Mega PR** (`YOUR_DEV_BRANCH` -> `main`) for your CTO's manual review. Never auto-merge to `main`.
- **`YOUR_DEV_BRANCH` = working prod.** If a dependency is in `YOUR_DEV_BRANCH`, it's available -- don't block on it.
- If `YOUR_DEV_BRANCH` doesn't exist: `git checkout -b YOUR_DEV_BRANCH main && git push -u origin YOUR_DEV_BRANCH`
- Never stash another agent's changes while agents are active. SQLite not tracked in git.

## Core Rules

1. **Read project CLAUDE.md first**
2. **Design before code**: Clarify -> Plan with approval -> Code
3. **TDD**: Failing tests (min 3 fail + 1 pass) -> Implement -> All pass
4. **Task tracking**: Notion = engineering (`rules/notion-tracking.md`). Command Center = business. No task files in customer repos.
5. **Simplicity first**: Minimal changes. Root causes. Senior dev standards.
6. **Self-improvement**: After corrections -> update Command Center `tasks/lessons.md`

## Workflow

- **Plans**: Interactive -> plan mode for 3+ steps. Autonomous loops -> Notion comment, never plan mode UI.
- **Priority**: High+Low -> High+Med -> Med+Low -> High+High -> Med+Med -> Med+High -> Low+Low -> Low+Med -> Low+High
- **Quality gates**: Tests pass. "What was NOT tested" on every PR. Screenshots/blogs = Docs agent's job.

## Magic Keywords

| Keyword | What it does |
|---------|-------------|
| **"prep sprint"** | Query board -> propose 3-5 tasks by impact -> you approve -> check `Move to Current Sprint` checkbox. Never auto-set without approval. Use My Current Sprint view (`YOUR_CURRENT_SPRINT_VIEW_ID`) to check current state. |
| **"launch swarm"** | Requires Current Sprint curated. Launches: Sprint Worker + Watchdog + Planner + Reviewer + Docs + cron jobs. All agents query My Current Sprint view. |
| **"prep for deploy"** | Run tests -> open PR targeting `YOUR_DEV_BRANCH` -> report. |
| **"prep mega pr"** | Sync `YOUR_DEV_BRANCH` <- `main` -> run tests -> verify all sub-PR gates -> open mega PR per `loop-back-testing.md` -> notify your CTO via Slack. |

## Development Lifecycle

0. **Notion** -- find/create task, set "In progress"
1. **Risk** -- Low/Medium/High
2. **Size** -- >500 lines? Break into sub-tasks
3. **Plan** -- get approval
4. **Tests** -- write failing tests, then implement
5. **PR** -- target `YOUR_DEV_BRANCH`, Reviewer-First format
6. **Pipeline** -- Reviewer -> Docs -> Watchdog merges

Done = merged to `YOUR_DEV_BRANCH`. Mega PR is a separate batch.

## Extended Rules (`.claude/rules/`)

11: Session Hygiene | 12: Notion Tracking | 13: Loop Modes | 14: Auto User Guide | 15: Meeting Actions | IDs: `reference-ids.md` | Health: `prompt-health.md`
