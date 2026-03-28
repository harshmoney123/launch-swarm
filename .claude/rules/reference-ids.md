# Reference IDs -- Single Source of Truth

All hardcoded IDs in one place. If an ID changes, update HERE and nowhere else. Other rules reference this file.

## Notion

- **Sprint board collection**: `collection://YOUR_COLLECTION_ID`
- **Master Table view (ALL tasks, no filters)**: `view://YOUR_MASTER_TABLE_VIEW_ID` <-- USE THIS for planners/sweeps
- **Backlog view (Sprint="Backlog" only)**: `view://YOUR_BACKLOG_VIEW_ID` <-- CAUTION: misses tasks not in "Backlog" sprint
- **My Backlog view (owner=me only)**: `view://YOUR_MY_BACKLOG_VIEW_ID`
- **My Current Sprint view (Sprint="Current Sprint")**: `view://YOUR_CURRENT_SPRINT_VIEW_ID` <-- USE THIS for Sprint Worker/Task Planner
- **My Tasks In Review view**: `view://YOUR_IN_REVIEW_VIEW_ID`
- **Status values**: "To-do", "Not started", "Start_Test", "Start", "Blocked", "In progress", "In Review", "Done"
- **Source values**: "Human" (default for all tasks)

### People
- **Team Lead**: `teamlead@example.com` -- user ID `YOUR_TEAM_LEAD_NOTION_USER_ID`
- **CTO**: `cto@example.com` -- user ID `YOUR_CTO_NOTION_USER_ID`

## Webflow (optional -- for blog/marketing site)

- **Site ID**: `YOUR_WEBFLOW_SITE_ID`
- **Blog Posts collection ID**: `YOUR_WEBFLOW_BLOG_COLLECTION_ID`
- Use Webflow MCP tools -> `list_collection_items` to query

## Slack

- **Swarm Digest channel**: `#swarm-digest` -- channel ID `YOUR_SLACK_CHANNEL_ID`
- **Team Lead DM**: user ID `YOUR_SLACK_USER_ID`
- All swarm notifications, critical PR alerts, and daily digests go to `#swarm-digest`

## Discord (optional -- for feature announcements)

- **Feature Launch webhook** (channel: `#internal-feature-request`): `YOUR_DISCORD_WEBHOOK_URL`
- Used for: Feature launch announcements, team training posts

## ClickUp (optional -- for business task tracking)

- **Task list ID**: `YOUR_CLICKUP_LIST_ID`
- Use ClickUp MCP tools for business tasks

## S3 -- Code Review Screenshots

- **Bucket**: `YOUR_SCREENSHOT_BUCKET` (your AWS region)
- **Base URL**: `https://YOUR_SCREENSHOT_BUCKET.s3.YOUR_REGION.amazonaws.com`
- **Path pattern**: `pr-{number}/{timestamp}/{filename}.png`
- **Upload script**: `scripts/upload-pr-screenshots.sh` (in your frontend repo)
- **Policy**: Public read on objects, no public listing, account-only write

## Command Center (optional -- for business ops tracking)

- **Database path**: `YOUR_COMMAND_CENTER_DB_PATH`
- **Task docs**: `YOUR_COMMAND_CENTER_TASKS_PATH`
- **Tables**: `tasks`, `agents`, `repos`, `achievements`, `stats`, `lines_ledger`

## Sprint Points Scale (Fibonacci)

| Points | Meaning | Level |
|--------|---------|-------|
| 0 | No effort (tracking only) | Task |
| 1 | Half day | Task |
| 2 | Full day | Task |
| 3 | Day and a half | Task |
| 5 | Half week (~2.5 days) | Task |
| 8 | Full week | Task **<-- MAX** |
| 13 | 2 weeks | Epic -- break into sub-tasks |
| 21 | 4 weeks | Epic/Phase |
| 34 | Month and a half | Epic |
| 55 | Quarter | Epic |
