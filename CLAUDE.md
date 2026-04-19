# Global Development Rules

**IMPORTANT: Read `.claude/.private/overrides.md` first — it maps all YOUR_* placeholders in these rules to real project values. Read `.claude/.private/reference-ids.md` for all real IDs.**

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
| **"prep sprint"** | Query board -> propose tasks totaling **>= 20 SP** (floor, not ceiling) -> you approve -> check `Move to Current Sprint` checkbox. **MUST exclude any task tagged `Requires: Me`.** Priority order: (1) full AFK-able epics, (2) prelim tasks inside epics that can ship without you, (3) High, (4) Medium, (5) Low. Within each tier, prefer **Low Risk** first (you can ship unverified). Use My Current Sprint view (`YOUR_CURRENT_SPRINT_VIEW_ID`) to check current state. See `rules/loop-afk-vs-live.md`. |
| **"prep session"** | Surfaces tasks tagged `Requires: Me`, grouped by session subtag (`Session: Creds` / `Taste` / `Money` / `CTO` / `Device`). Output is a batched to-do list for your next in-person work block, NOT a sprint. See `rules/loop-afk-vs-live.md`. |
| **"launch swarm"** | Requires Current Sprint curated. Fires local `/loop` commands via the `loop` skill for Sprint Worker (30m), Watchdog (10m), Reviewer (20m), Planner (15m), Docs (20m). Each loop needs its own terminal. Cloud triggers were tried and abandoned (2026-04-05) -- they 500'd on manual run and the 1h min interval was too coarse. Local loops require laptop awake but have full access to Playwright, AWS creds, test env SSH, Payload secret. See `rules/loop-modes.md`. |
| **"prep for deploy"** | Run tests -> open PR targeting `YOUR_DEV_BRANCH` -> report. |
| **"prep for review"** | Opens **draft** mega PR with structured review table. You comment `fix #N: reason` or `revert #N` on the PR. See `rules/loop-review.md`. |
| **"batch fix"** | Reads your mega PR comments, reverts `revert` items, ships fix PRs for `fix` items, updates mega PR. See `rules/loop-review.md`. |
| **"ship clean"** | Ships what's ready NOW -- reverts all `fix` + `revert` items, marks mega PR ready. Held items go to next sprint. |
| **"prep mega pr"** | (Legacy) Sync `YOUR_DEV_BRANCH` <- `main` -> run tests -> verify all sub-PR gates -> open mega PR per `loop-back-testing.md` -> notify your CTO via Slack. |

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

11: Session Hygiene | 12: Notion Tracking | 13: Loop Modes | 14: Auto User Guide | 15: Meeting Actions | 16: Sprint Review (`loop-review.md`) | IDs: `reference-ids.md` | Health: `prompt-health.md`
