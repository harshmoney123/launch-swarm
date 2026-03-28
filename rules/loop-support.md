# Support Loops -- Notion Sync, Task Planner, Digest, Rules Hygiene

## Notion Sync (`/loop 5m`)

1. Open PRs -> tasks "In Review". Merged to `YOUR_DEV_BRANCH` -> "Done" with date.
2. "In Progress" with no recent commits -> flag stale.
3. Dedup check before creating. Dedup sweep every 5th tick.
4. Every 3rd tick: pull meeting action items per Rule 15.
5. Screenshot sync: untracked `.png` files -> match to PR, upload to S3, delete local.

## Task Planner (`/loop 15m`)

Senior architect -- reads code, never writes it.

`To-do -> [Planner adds plan] -> To-do (planned) -> [Sprint Worker claims] -> In progress -> Done`

1. Query **My Current Sprint view** (`YOUR_CURRENT_SPRINT_VIEW_ID`). Filter `Status === "To-do"`, sort by priority order. This view already filters by `Sprint = "Current Sprint"`.
2. Sort by priority order (see `loop-modes.md`).
3. **Owner filter (MANDATORY)**: Only plan tasks owned by you or unassigned. **Skip tasks owned by your CTO or anyone else** -- do not comment on or modify their tasks.
4. Skip `Planned === true`. Skip Epics (13+ SP).
5. Deep research: read task, find files, check architecture, identify dependencies.
6. Generate 3-4 options with files, lines, pros, cons, risk.
7. Recommend best option. Set Risk Level + `Planned = true` on Notion task.
8. Post plan as Notion comment:

```markdown
## Implementation Plan
### Risk: Low / Medium / High
### Options
**A: [Name]** -- files, lines, pros, cons
**B: [Name]** -- files, lines, pros, cons
### Recommended: [X]
**Why**: [2-3 sentences]
### Checklist
1. [ ] Failing tests: [cases]
2. [ ] Files to modify: [list]
3. [ ] Verify: [what to test]
### Dependencies
Backend PR: [yes/no] | Blocked by: [list]
```

9. Break down >3 SP or >300 lines into sub-tasks.
10. Risk Level Backfill + Plan Coverage Sweep every tick (use Master Table view). **Same owner filter applies** -- only touch your own or unassigned tasks.

## Swarm Digest (cron `0 8 * * *`)

Daily briefing: TL;DR -> What shipped -> Mega PR status -> High-risk items -> Tomorrow's priorities. Deliver to Slack `YOUR_DIGEST_CHANNEL`.

## Rules Hygiene (`/loop 12h`)

Check Claude Code version, best practices from GitHub/X/changelog. Diff against harness. Apply updates, propose experiments, report health score.
