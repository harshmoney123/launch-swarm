# Rule 13: Autonomous Loop Modes

## Pipeline

```
"prep sprint" -> You approve -> "launch swarm" -> Sprint Worker -> PR -> Senior Reviewer -> Docs -> Watchdog merges -> YOUR_DEV_BRANCH -> [Sprint Complete] -> "prep for review" -> Draft mega PR -> You comment fix/revert -> "batch fix" -> Swarm fixes -> You merge to main
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

## Launch mode: local loops

The swarm runs as **local `/loop` commands** in separate Claude Code terminals. Cloud triggers were evaluated and rejected on 2026-04-05 (API 500'd on manual run; disabled trigger `trig_01JvF6rHqYyuHbukMM3NXpvy` left in place for reference).

**Trade-off accepted**: laptop must stay awake for the loops to run. In exchange you get Playwright, AWS/S3 creds, test env SSH, Payload secret, local file access, and 10-30m cadence instead of 1h.

## Agents

| Keyword | Interval | File | Role |
|---------|----------|------|------|
| `sprint worker` | 30m | `loop-sprint-worker.md` | Code + tests + PR (Current Sprint only) |
| `plan tasks` | 15m | `loop-support.md` | Implementation plans (Current Sprint only) |
| `review prs` | 20m | `loop-pr-ops.md` | Deep code review -> Approve |
| `watch prs` | 10m | `loop-pr-ops.md` | Fix comments, auto-merge after both gates |
| `qa tests` | 20m | `loop-qa.md` | Real smoke test on test env -> QA Pass (the gate) |
| `audit` | nightly | `loop-auditor.md` | Codebase audit + external CC-repo scan -> Notion report + Top 5 tasks |
| `watch logs` | 1m | `loop-log-watcher.md` | Tail test env logs, report errors |
| `swarm metrics` | on-demand | `swarm-metrics.md` | Diagnose throughput / quality / loop health from `~/.swarm/swarm-log.jsonl` |
| `docs` (dormant) | — | `loop-docs.md` | Blog drafts + screenshots. Not in `launch swarm`. Fire manually with `docs` keyword if needed. |

## How to actually launch

Each loop needs its own Claude Code session. Open a new terminal per agent and run:

```
cd YOUR_PROJECT_ROOT
claude
# inside the session:
/loop 30m sprint worker
```

Then repeat in separate terminals for `watch prs`, `review prs`, `plan tasks`, `docs` at the intervals above.

**Minimal swarm** (if you only want 2 terminals open): `sprint worker` + `watch prs`. These 2 are the core throughput pipeline. Add `review prs` as a 3rd if you want the quality gate running.

## Combos

| Keyword | Agents |
|---------|--------|
| `work horses` | Sprint Worker + PR Watchdog |
| `senior leadership` | Task Planner + Senior Reviewer |
| `ship it` | QA agent |
| `launch swarm` | Sprint Worker + PR Watchdog + Senior Reviewer + Task Planner + QA (Docs is dormant, fire separately if ever needed) |

## Gates

Two gates before auto-merge to `YOUR_DEV_BRANCH`:
1. **Senior Reviewer Approve** -- code is correct
2. **QA Pass** -- real smoke test on the test env verified end-state behavior (see `loop-qa.md`)

Docs (blog drafts + screenshots) is dormant. Fire manually with `docs` keyword only when you want a blog draft; it is NOT a merge gate.

## Sprint Complete -- Review Handoff

**Trigger**: When the swarm enters idle state -- Sprint Worker has no To-do tasks, all agents report idle on consecutive ticks. Do NOT wait for you to ask. Present the review handoff **immediately** on the first idle cycle after work was done.

**Steps** (execute automatically, no human prompt needed):

1. Stop all idle cron jobs.
2. Present sprint summary: tasks shipped, PRs merged, test count.
3. Prompt: "Say **prep for review** to open the draft mega PR for your review."

**Do NOT auto-proceed to mega PR** -- human review is required first.
**Do NOT keep looping idle agents** -- once review handoff is presented, stop all cron ticks until you respond.

The full review flow is in `rules/loop-review.md`:
- **"prep for review"** -> draft mega PR with review table
- **"batch fix"** -> process your fix/revert comments
- **"ship clean"** -> ship what's ready, hold the rest

## No background polling in subagents

Subagents MUST NOT use `Bash(run_in_background=true)` followed by `until ... do sleep; done` polling loops. These become zombie processes when the subagent exits — nobody kills them. Instead:
- Run `test-env.sh sync`, `rsync`, `npm install` **synchronously** with `timeout 300` prefix
- If the operation is truly too slow, use the `Monitor` tool (auto-cleans up) instead of hand-rolled polling

This rule was added 2026-04-17 after finding 50+ zombie polling loops from subagents that spawned background bash tasks for test-env sync monitoring.

## Metric log contract

Every loop MUST append one JSONL line to `~/.swarm/swarm-log.jsonl` on every tick. Contract + canonical loop names + action verbs live in `swarm-metrics.md`. This is how `swarm metrics` diagnoses the swarm. A loop that does not log is invisible to the metric, so silent loops get blamed for problems they did not cause. No exceptions.

## Priority Order

High+Low risk -> High+Med -> Med+Low -> High+High -> Med+Med -> Med+High -> Low+Low -> Low+Med -> Low+High. Tiebreaker: CI green first, fewer unresolved comments first.

## Coordination

- Worktree isolation for all code-writing loops
- Notion "In progress" = claimed, others skip
- Auto-merge to `YOUR_DEV_BRANCH` only -- never to `main`
- Watchdog keeps `YOUR_DEV_BRANCH` synced with `main` every tick
- No duplicate Notion tasks -- search before creating
- **High Risk tasks get robust test plans** -- unit + integration + manual QA steps + rollback plan. Auth/Agent infra PRs held for CTO's approval. Everything else ships.
