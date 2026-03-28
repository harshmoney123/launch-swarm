# Rule 12: Notion Task Tracking

All work items MUST be tracked in Notion. Use the Notion MCP tools.

**This is Step 0 -- before planning, before code, before anything.**

## Rules

1. **Every task needs a Notion entry (FIRST STEP)**: Search the sprint board for an existing task. If found, read it and update status to "In progress". If not found, create one.
2. **Fibonacci Sprint Points (REQUIRED)**: See scale in `reference-ids.md`. Max 8 pts per task. 13+ = Epic, break into Phases (<=21 pts) -> Tasks (<=8 pts). No one works on Epics/Phases directly.
3. **Due Date (REQUIRED)**: Set realistic due date based on effort estimate.
4. **Default Assignment**:
   - **UX** tasks -> Team lead (always, no exceptions)
   - **Infra** and **Security** tasks -> CTO
   - **All other** -> Team lead unless specified
5. **Bidirectional PR <-> Notion linking**:
   - PR description includes Notion task link
   - Comment on Notion task with PR URL
6. **Status flow**: To-do -> In progress -> In Review (PR open) -> Done (PR merged, set Completed Date)
7. **Source property**: Set `Source: "Human"` on all tasks. (Agent-created tasks have been removed from the workflow.)

## Querying (IMPORTANT)

- **NEVER use `notion-search`** for finding tasks by status -- it's semantic and misses items.
- **Use `notion-query-database-view`** with the **Master Table view** (see `reference-ids.md`) for planners, sweeps, and any query that needs ALL tasks. The Master Table has NO filters.
- **DO NOT use the Backlog view** for comprehensive queries -- it filters by `Sprint = "Backlog"` and misses tasks in other sprints or with no sprint set.
- Parse the JSON results array and filter by `Status` field in code.
