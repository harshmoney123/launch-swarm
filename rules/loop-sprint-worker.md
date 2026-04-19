# Sprint Worker -- "sprint worker" or "go ham"

`/loop 30m`. Max 3 concurrent. Pulls tasks from Notion, writes code + tests, opens PRs.

## Risk Classification

| Level | Criteria | Extra requirements |
|-------|----------|--------------------|
| **Low** | UI tweaks, CSS, copy. No backend/auth. | Unit tests |
| **Medium** | New pages/components, frontend logic. | Unit tests |
| **High** | External systems, Stripe, DB mutations. | Robust test plan in PR: unit + integration + manual QA steps + rollback plan. |

**Gating rules by domain:**
- **Stripe / Billing** -> Team lead owns unilaterally. Swarm ships it, robust test plan required.
- **Auth (login, tokens, sessions, password reset)** -> Check with CTO before shipping. PR stays in review until CTO approves.
- **Agent infra (core AI, agent orchestration, MCP)** -> Check with CTO before shipping. PR stays in review until CTO approves.
- **Everything else** -> Ship it.

Set Risk Level on Notion task. >500 lines -> break into sub-tasks first.

## Each tick

1. Query Notion via **My Current Sprint view** (`YOUR_CURRENT_SPRINT_VIEW_ID`). Filter `Status === "To-do"`, sort by priority order. This view already filters by `Sprint = "Current Sprint"`.
2. Skip tasks not owned by you or unassigned. Skip blocked tasks. **Skip any task tagged `Requires: Me`** -- those belong in `prep session`, not AFK work. See `loop-afk-vs-live.md`.
3. **All tasks get worked** -- no skipping by risk level. For High Risk: include robust test plan (unit + integration + manual QA steps + rollback). For Auth/Agent infra: open PR but hold for CTO's approval before merge.
4. Prefer tasks with `Planned === true` -- follow the plan comment directly.
5. Update Notion -> "In progress". Classify risk if not set.
6. Write tests -> implement -> `npm test` must pass.
7. Open PR targeting `YOUR_DEV_BRANCH`: `gh pr create --base YOUR_DEV_BRANCH`. Use Reviewer-First format.
8. Move Notion -> "In Review". Pick next task.

**Playwright is QA's job, not yours.** Do NOT open the MCP browser in Sprint Worker cycles. The MCP chrome instance is a single shared resource across the swarm, and the QA loop (`~/.claude/rules/loop-qa.md`) is its exclusive owner. If Sprint Worker also grabs it, concurrent ticks collide and both fail.

What Sprint Worker is responsible for on UX tasks:
1. Implement + `npm run test:run` must pass
2. Write concrete, numbered "How to test" steps in the PR body -- the QA loop reads these and executes them against the test env
3. Call out any specific data state or account needed ("log in as admin", "have at least one `trialing` contact", etc.)
4. List edge cases you did NOT cover -- QA expands the adversarial pass from that list
5. Mark Notion Status='In Review'

That's it. No rsync, no `test-env.sh sync`, no screenshots, no browser navigation. If a layout regression slips through, the QA loop catches it before auto-merge and files a `REQUEST_CHANGES` with repro steps. That is the whole point of QA existing.

Edge cases where Sprint Worker DOES need the test env:
- Investigating a failing test that only reproduces against real data -- read-only inspection, no browser.
- Debugging a reported bug that needs server logs -- use `loop-log-watcher` output or SSH into the test env directly.
In both cases, do not open the Playwright MCP browser.

**Mid-work escalation**: If you start a task and discover it needs credentials, taste calls, or a live account you don't have (e.g., "I need an API key", "there are 3 valid design choices"), STOP. Re-tag the task `Requires: Me` + the right session subtag (see `loop-afk-vs-live.md`), add a comment explaining what's blocking, and move on. Do not open a half-done PR.

## Investigate Tasks

Tasks prefixed with "Investigate:" follow a different flow -- **research only, no code changes**.

1. Update Notion -> "In progress".
2. Research: read code, grep, check logs, test env, Playwright, whatever's needed. Do NOT write code or open PRs.
3. Append a structured `## Audit Findings` section to the **Notion task page content** with:
   - **Summary**: 1-2 sentence finding
   - **Evidence**: What was checked (files, logs, endpoints, accounts tested)
   - **Root Cause** (if applicable): What's actually happening
   - **Recommendation**: One of: `Convert to implementation task`, `Close -- not a real issue`, `Needs more investigation -- [what's missing]`
   - **Proposed Next Steps**: If converting, outline the fix approach + estimated effort
4. Post a **Notion comment**: "Investigation complete -- [one-line recommendation]. See Audit Findings above."
5. Move Notion -> "In Review". You review and decide next action.

**Never auto-convert an investigate task into code.** The whole point is to pause and let you decide before committing to a fix.

## Branching

Always branch from `YOUR_DEV_BRANCH`: `git fetch origin && git checkout YOUR_DEV_BRANCH && git merge origin/main`.

## Never Block on YOUR_DEV_BRANCH

`YOUR_DEV_BRANCH` is the working production state. **If a dependency has been merged to `YOUR_DEV_BRANCH`, it is available -- do not treat it as blocked.** The mega PR pattern means `YOUR_DEV_BRANCH` accumulates all work before shipping to `main`, so anything in `YOUR_DEV_BRANCH` is as good as done for dependency purposes.

- A task that says "depends on backend PR" -- if that backend PR is merged to `YOUR_DEV_BRANCH`, proceed.
- A task that says "blocked by X" -- if X is in `YOUR_DEV_BRANCH`, it's unblocked. Start working.
- Only truly block on things NOT yet in `YOUR_DEV_BRANCH` (e.g., pending PRs, unstarted work).
- When in doubt, check if the dependency code exists on `YOUR_DEV_BRANCH` and use it.

## Idle

No To-do tasks in Current Sprint -> stop. Don't self-assign work. Don't pull from the broader backlog -- only you curate what goes into Current Sprint.
