# Overnight Auditor -- "audit" or "run audit"

Cadence: **once per night**, typically fired at 2am via a scheduled wake-up or a `CronCreate` trigger. Runs solo while the swarm is asleep, produces one report in Notion, surfaces top tasks.

## Purpose

The swarm fixes what it is pointed at. Nothing else is looking at the codebase strategically. The Auditor spends a long concentrated run doing two things the day loops cannot:
1. **Codebase sweep** across `YOUR_BACKEND_REPO` (backend), `YOUR_FRONTEND_REPO` (frontend), `YOUR_MARKETING_REPO` (marketing) -- surface tech debt, perf hotspots, security smells, abandoned feature flags, test-coverage gaps, CVE bumps.
2. **External scan** of public Claude Code / agentic-dev repos -- find patterns, prompts, hooks, and testing tricks we do not already have, and propose how to adopt them.

Output is a Notion page, not a PR. You reviews in the morning and promotes the high-leverage items to Current Sprint himself.

## Each run

### Arm 1 -- Codebase audit

For each repo in `YOUR_BACKEND_REPO`, `YOUR_FRONTEND_REPO`, `YOUR_MARKETING_REPO`:

1. `git log --since='7 days ago' --stat` to find the hot files this week.
2. Read the top 20 files by churn. Flag:
   - Functions > 80 lines
   - Duplicated logic across files (grep for repeated string literals, repeated branch conditions)
   - Dead exports (exported but no importers)
   - Abandoned feature flags (env var read but never flipped in the last 90 days of commits)
   - TODOs older than 60 days
   - `any` types in TypeScript where the context is non-trivial
3. Test-coverage gaps weighted by churn: if a file is in the top 20 by churn AND has <30% line coverage in `coverage-summary.json`, it is a coverage-risk finding.
4. Perf hotspots (frontend): components that render >3 times per route change, `useEffect` deps that include the full `props` object, list renders without `key` or without virtualization on > 100 items.
5. Security smells: secrets committed, DynamoDB queries without a scoped keyCondition, API routes without auth middleware, CORS wildcards, `dangerouslySetInnerHTML` with non-sanitised input.
6. Dependency health: `npm outdated --json` -- flag anything with a known CVE (check via `npm audit --json`) or anything >= 2 major versions behind. Cap the list at 10.

### Arm 2 -- External pattern scan

1. Fetch the READMEs + top-5-most-starred files of:
   - `garrytan/gstack`
   - `anthropics/claude-code` (official)
   - Any repo surfaced by `WebSearch` for `"claude code" agent prompts site:github.com` filtered to the last 14 days. Cap at 5 new repos per run.
2. For each, read anything in `/skills/`, `/agents/`, `/prompts/`, `/rules/`, or `/.claude/` directories.
3. Compare against our `~/.claude/rules/`. Extract concrete patterns we lack, with the source file:line. Do NOT copy wholesale prompts -- only the idea + how it would fit our structure.
4. Drop anything we already have, anything too generic ("write better tests"), or anything that contradicts an existing feedback memory.

### Arm 3 -- Synthesis + ranking

Produce a single Notion page under a parent "Audits" page (create it if missing):

```
# Overnight Audit -- {date}

## TL;DR
{2-3 sentences on the state of the codebase + the one biggest external learning}

## Top 5 Highest-Leverage (impact x ease)
1. <title> -- <impact note> -- <est SP> -- [source: codebase|external] -- <promote to sprint?>
2. ...
...

## Codebase Findings (full list, grouped by repo)
### YOUR_BACKEND_REPO (backend)
- [perf] ...
- [security] ...
- [coverage] ...

### YOUR_FRONTEND_REPO (frontend)
- ...

### YOUR_MARKETING_REPO
- ...

## External Learnings
- **[gstack]** <pattern> -- at `path/file.md:LINE` -- how it would improve our `~/.claude/rules/loop-X.md`
- **[other repo]** ...

## Skipped / Considered-Not-Worth
- ... (with a one-line reason)

## Auditor Meta
- Repos scanned: 3
- Files read: N
- External repos fetched: M
- Runtime: X min
```

Ranking formula for "Top 5":
- Impact: would this save you time every sprint? flag potential revenue impact? fix a latent security issue?
- Ease: SP estimate. <= 3 SP is ideal; anything >= 8 SP gets parked as an Epic proposal unless the impact is exceptional.
- Score = impact * (10 / SP). Sort descending.

### Arm 4 -- Promote to Notion backlog

For each of the Top 5, create a Notion task in the Sprint Tasks database with:
- `Source: "Auto"`
- `Status: "To-do"`
- `Priority Level: <Auditor's call>`
- `Risk Level: <Auditor's call>`
- `Sprint Points: <est>`
- `Tags: ["Auto-audit"]`
- A comment linking back to the Audit page

Do NOT auto-check `Move to Current Sprint`. You promotes manually after reviewing.

## Metric logging

At the end of the run, append to `~/.swarm/swarm-log.jsonl`:
```json
{"ts":"<ISO UTC>","loop":"auditor","tick":"complete","action":"audited_repos","duration_s":<num>,"findings":<count>,"top5":[<task_urls>]}
```

## Coordination

- Auditor is solo overnight. If it sees the test env in active use (any `QA: Running` comment less than 30 minutes old on any PR), it skips any test-env-touching checks and runs a pure-read audit.
- Auditor never writes code. It only reads, greps, and produces Notion content.
- Auditor never touches prod DynamoDB. Read-only; no MCP DB calls against non-`Dev-*` tables.

## What NOT to do

- Do NOT open PRs. Auditor is research, not execution.
- Do NOT flood Notion with >15 findings promoted to tasks. Top 5 only; the rest stay in the Audit page body.
- Do NOT re-surface a finding already in the backlog -- grep the backlog by title keyword before creating.
- Do NOT run during the day -- cadence is once overnight. If manually invoked during the day, ask first whether you want to proceed.
