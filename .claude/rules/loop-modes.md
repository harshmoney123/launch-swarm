# Rule 13: Autonomous Loop Modes

## Pipeline

```
"prep sprint" -> You approve -> "launch swarm" -> Sprint Worker -> PR -> Senior Reviewer -> Docs -> Watchdog merges -> YOUR_DEV_BRANCH -> [Sprint Complete] -> Manual QA -> Mega PR -> CTO merges to main
```

## Current Sprint Curation

**Swarm only works on tasks in My Current Sprint view** (`YOUR_CURRENT_SPRINT_VIEW_ID`). No backlog grinding.

### "prep sprint" flow:
1. Query full board -> filter out junk/epics/CTO-owned
2. **Only propose tasks the swarm can complete autonomously** -- no human handoffs, no external config steps, no waiting on someone to send something. If it needs a human in the loop, it's not a sprint task.
3. Propose 3-5 tasks ranked by impact with one-line rationale (consider meetings, PRs, blockers)
4. You approve/reject/swap -> approved tasks get `Move to Current Sprint` checkbox checked
5. **Never set Current Sprint without explicit approval** -- Claude suggests, you decide
6. Between sessions: clear Done tasks from Current Sprint before curating new ones

## Agents

| Keyword | Interval | File | Role |
|---------|----------|------|------|
| `sprint worker` | 30m | `loop-sprint-worker.md` | Code + tests + PR (Current Sprint only) |
| `plan tasks` | 15m | `loop-support.md` | Implementation plans (Current Sprint only) |
| `review prs` | 20m | `loop-pr-ops.md` | Deep code review -> Approve |
| `watch prs` | 10m | `loop-pr-ops.md` | Fix comments, auto-merge after both gates |
| `docs` | 20m | `loop-docs.md` | Screenshots + blog -> Documented |
| `watch logs` | 1m | `loop-log-watcher.md` | Tail test env logs, report errors |

## Combos

| Keyword | Agents |
|---------|--------|
| `work horses` | Sprint Worker + PR Watchdog |
| `senior leadership` | Task Planner + Senior Reviewer |
| `ship it` | Docs agent |
| `launch swarm` | All (requires Current Sprint to be curated first) |

## Gates

Two gates before auto-merge to `YOUR_DEV_BRANCH`:
1. **Senior Reviewer Approve** -- code is correct
2. **Docs Documented** -- screenshots + blog draft (for `feat:`)

## Sprint Complete -- Manual QA Handoff

**Trigger**: When the swarm enters idle state -- Sprint Worker has no To-do tasks, all agents report idle on consecutive ticks. Do NOT wait for you to ask. Present the QA handoff **immediately** on the first idle cycle after work was done.

**Steps** (execute automatically, no human prompt needed):

1. **Ensure test env is running**: `cd YOUR_PROJECT_ROOT && ./test-env/test-env.sh status`. If not running, deploy it. Verify latest `YOUR_DEV_BRANCH` is deployed.
2. **Open test env in browser**: Use Playwright to navigate to `http://<test-env-IP>:YOUR_PORT`
3. **Present a structured testing plan** covering EVERY task shipped this sprint. Format:

```markdown
## Manual QA Testing Plan -- Sprint [date]

### Summary
[X tasks shipped, Y PRs merged. Brief description of what changed.]

---

### Test 1: [Task name]
**Why this was done**: [1-2 sentences on the problem/motivation]
**What changed**: [Brief description]

**As admin** (`admin@example.com` / `adminpassword`):
1. [Step-by-step instructions]
2. [What you should see]

**As basic user** (`user@example.com` / `userpassword`):
1. [Step-by-step instructions]
2. [What you should see / should NOT see]

**Pass criteria**: [What confirms it works]

---
[Repeat for each task]

### Edge Cases to Spot-Check
- [List anything the automated tests couldn't cover]
```

4. **Wait for your feedback** -- fix any issues reported, then proceed to "prep mega pr" when approved.

**Do NOT auto-proceed to mega PR** -- manual QA approval is required first.
**Do NOT keep looping idle agents** -- once QA handoff is presented, stop all cron ticks until you respond.

## Priority Order

High+Low risk -> High+Med -> Med+Low -> High+High -> Med+Med -> Med+High -> Low+Low -> Low+Med -> Low+High. Tiebreaker: CI green first, fewer unresolved comments first.

## Coordination

- Worktree isolation for all code-writing loops
- Notion "In progress" = claimed, others skip
- Auto-merge to `YOUR_DEV_BRANCH` only -- never to `main`
- Watchdog keeps `YOUR_DEV_BRANCH` synced with `main` every tick
- No duplicate Notion tasks -- search before creating
- **High Risk tasks get robust test plans** -- unit + integration + manual QA steps + rollback plan. Auth/Agent infra PRs held for CTO's approval. Everything else ships.
