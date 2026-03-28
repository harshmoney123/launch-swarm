# launch-swarm

**Everyone else open-sourced their `.claude` file. I open-sourced my engineering team.**

This is the exact `.claude` configuration I use daily to run a 6-agent AI engineering swarm from [Claude Code](https://claude.ai/code). It plans sprints, writes code, reviews PRs, generates documentation with annotated screenshots, auto-merges after dual-gate approval, and hands off to manual QA — autonomously.

I'm the CEO of [AgentWeb](https://www.agentweb.pro), a 4-person team building AI marketing agents. I'm a YC founder (W23) who still codes part-time — the rest of my day is sales, marketing, and ops. This system is how I stay dangerous as a part-time engineer. It's not a weekend experiment. It's the production workflow behind a real startup.

**The core insight:** AI agents fail for the same reason human teams fail — no org design. Prompts rot. Agents step on each other. Nobody owns quality. This system treats AI agents like a real engineering team: defined roles, strict boundaries, accountability gates, and self-healing instructions.

---

## Table of Contents

- [System Architecture](#system-architecture)
- [The 6 Agents](#the-6-agents)
- [Key Concepts](#key-concepts)
- [Installation](#installation)
- [What You Need (Infrastructure)](#what-you-need)
- [Mix and Match](#mix-and-match)
- [Magic Keywords](#magic-keywords)
- [PR Templates](#pr-templates)
- [Customization Guide](#customization-guide)
- [FAQ](#faq)
- [Credits](#credits)

---

## System Architecture

The swarm follows a linear pipeline with two quality gates before anything merges:

```
                        YOU
                         |
                    "prep sprint"
                         |
                  ┌──────▼──────┐
                  │  Curate 3-5  │
                  │  sprint tasks │
                  └──────┬──────┘
                         |
                   "launch swarm"
                         |
          ┌──────────────▼──────────────┐
          │     SWARM ACTIVE            │
          │                             │
          │  ┌─────────────────────┐    │
          │  │   Task Planner      │    │
          │  │   (architects)      │    │
          │  └────────┬────────────┘    │
          │           │                 │
          │  ┌────────▼────────────┐    │
          │  │   Sprint Worker     │    │
          │  │   (codes + tests)   │    │
          │  └────────┬────────────┘    │
          │           │                 │
          │       Opens PR              │
          │           │                 │
          │  ┌────────▼────────────┐    │
          │  │  GATE 1:            │    │
          │  │  Senior Reviewer    │    │
          │  │  (code review)      │──┐ │
          │  └─────────────────────┘  │ │
          │                           │ │
          │  ┌────────────────────┐   │ │
          │  │  GATE 2:           │   │ │
          │  │  Docs Agent        │◄──┘ │
          │  │  (screenshots +    │     │
          │  │   blog drafts)     │     │
          │  └────────┬───────────┘     │
          │           │                 │
          │  ┌────────▼────────────┐    │
          │  │  PR Watchdog        │    │
          │  │  (auto-merge after  │    │
          │  │   both gates pass)  │    │
          │  └────────┬────────────┘    │
          │           │                 │
          │  ┌────────▼────────────┐    │
          │  │  Log Watcher        │    │
          │  │  (monitors errors)  │    │
          │  └─────────────────────┘    │
          │                             │
          └──────────────┬──────────────┘
                         │
                  Swarm goes idle
                         │
              ┌──────────▼──────────┐
              │  QA Handoff         │
              │  (structured test   │
              │   plan generated)   │
              └──────────┬──────────┘
                         │
                    You test manually
                         │
              ┌──────────▼──────────┐
              │  "prep mega pr"     │
              │  dev → main         │
              │  (CTO reviews)      │
              └─────────────────────┘
```

### The Dual-Gate Rule

Nothing merges until **two independent gates** pass:

| Gate | Agent | What it checks |
|------|-------|---------------|
| **Gate 1** | Senior Reviewer | Code correctness, architecture, security, reliability |
| **Gate 2** | Docs Agent | Screenshots captured, blog draft written (for features), UI not broken |

The PR Watchdog only auto-merges after **both** gates approve. This catches an entire class of problems that tests alone miss: undocumented features, UI regressions that "work" but look wrong, missing screenshots for stakeholder review.

---

## The 6 Agents

### 1. Sprint Worker

> **Role:** The coder. Pulls tasks from your sprint board, writes failing tests first, implements the fix, and opens a PR. Trigger: `sprint worker` or `go ham`.

**Runs up to 3 concurrent instances** — you can parallelize across multiple tasks.

**What it does every 30 minutes:**
1. Queries your sprint board for `To-do` tasks in the current sprint
2. Skips tasks owned by others or marked as blocked
3. Classifies risk level (Low / Medium / High)
4. Writes failing tests (minimum 3 failing + 1 passing), then implements until all pass
5. Opens a PR targeting your dev branch using the [Reviewer-First format](#pr-templates)
6. Moves the task to `In Review` and picks up the next one

**What it does NOT do:**
- Never takes screenshots (that's the Docs Agent's job)
- Never self-assigns work from the backlog — only works on tasks **you** curated into the current sprint
- Never auto-converts research tasks into code changes

**Risk classification drives rigor:**

| Risk | When | What's required |
|------|------|----------------|
| Low | UI tweaks, CSS, copy changes | Unit tests |
| Medium | New pages, frontend logic | Unit tests |
| High | External APIs, payments, DB mutations | Unit + integration tests + manual QA steps + rollback plan |

**Domain gating** — certain areas require human approval before merge:
- **Payments/billing:** Ships with robust test plan, team lead owns
- **Auth (login, tokens, sessions):** PR opens but holds for CTO review
- **Core AI/agent infrastructure:** PR opens but holds for CTO review
- **Everything else:** Ships automatically through the pipeline

**Special mode — "Investigate:" tasks:**
Tasks prefixed with `Investigate:` trigger research-only mode. The agent reads code, checks logs, and writes findings — but **never writes code or opens PRs**. This creates a deliberate pause point where you decide whether to proceed with a fix.

**Standalone use:** Great on its own if you just want an autonomous coding agent. Pair it with the PR Watchdog for the minimum viable pipeline.

---

### 2. Task Planner

> **Role:** The architect. Reads code deeply, generates implementation plans with multiple options, but never writes a single line of code.

**What it does every 15 minutes:**
1. Finds `To-do` tasks in the current sprint that don't have plans yet
2. Deep-reads the codebase: finds relevant files, checks architecture, maps dependencies
3. Generates 3-4 implementation options, each with files to change, pros, cons, and risk assessment
4. Posts the plan as a comment on the task with a recommended approach
5. Marks the task as `Planned` so the Sprint Worker knows to follow the plan

**What it does NOT do:**
- Never writes code, opens PRs, or modifies files
- Never plans tasks owned by other team members
- Never works on Epics (13+ story points) — flags them for breakdown instead

**Why separating planning from coding matters:**
When the same agent plans and codes, it tends to commit to the first approach it thinks of. By separating these roles, the Planner can objectively evaluate multiple options without the sunk-cost bias of already having written code. The Sprint Worker then follows a vetted plan instead of improvising.

**Plan format:**
```markdown
## Implementation Plan
### Risk: Low / Medium / High
### Options
**A: [Name]** — files, approach, pros, cons
**B: [Name]** — files, approach, pros, cons
### Recommended: [X]
**Why**: [2-3 sentences]
### Checklist
1. [ ] Failing tests: [specific test cases]
2. [ ] Files to modify: [list]
3. [ ] Verify: [what to test after]
### Dependencies
Backend PR: [yes/no] | Blocked by: [list]
```

**Standalone use:** Excellent on its own as a "senior architect on demand." Use it before any complex task to get a second opinion on approach.

---

### 3. Senior Reviewer

> **Role:** The code reviewer. Reads the full diff, traces imports two levels deep, checks usage sites, and posts categorized comments.

**What it does every 20 minutes:**
1. Finds open PRs that haven't been reviewed yet
2. Prioritizes by risk level (high-risk PRs reviewed first)
3. Performs a deep read — minimum 5 minutes per PR:
   - Full diff + entire changed files
   - Imports traced 2 levels deep
   - Usage sites checked for breaking changes
   - Test files reviewed for coverage gaps
4. Evaluates against five criteria: Architecture, Reliability, Elegance, Security, Compatibility
5. Posts comments with severity tags and a verdict

**What it does NOT do:**
- Never fixes the code — only identifies issues (the Watchdog handles fixes)
- Never merges PRs
- Never takes screenshots

**Comment severity system:**

| Tag | Meaning | Action required |
|-----|---------|----------------|
| Bug | Will break in production | Must fix before merge |
| Nit | Style/convention issue | Should fix, won't block |
| Pre-existing | Problem existed before this PR | Informational, no action |
| Suggestion | Improvement idea | Optional |

**Verdict options:** `Approve` / `Needs changes` / `Rethink approach`

An `Approve` verdict triggers Gate 1 — the Docs Agent is next in line.

**Standalone use:** The highest-value standalone agent. Just install the Senior Reviewer + PR templates for dramatically better code review without the full swarm.

---

### 4. PR Watchdog

> **Role:** The merge bot and janitor. Fixes review comments, syncs branches, resolves conflicts, and auto-merges after both gates pass.

**What it does every 10 minutes:**

*Core loop:*
1. **Syncs your dev branch** with `main` to prevent drift
2. **Resolves merge conflicts** — checks out in a worktree, merges, runs tests, pushes
3. **Fixes CI failures** — diagnoses and pushes a fix
4. **Fixes review comments** — reads unresolved Bug and Nit comments, makes minimal fixes, runs tests, commits, and replies "Fixed"
5. **Auto-merges** when both gates pass (Senior Reviewer `Approve` + Docs `Documented` + CI green + no unresolved bugs)
6. **Redeploys test env** after each merge — rsync repos, sync containers, verify freshness

*Mega PR ops:*
7. **Handles CTO feedback** on mega PRs — investigates comments, branches from dev, fixes, opens PRs, replies with fix links
8. **Triggers Feature Launch** when a mega PR merges to `main` (fires the Docs Agent's announcement flow)
9. **Auto-closes tasks** on mega PR merge — parses sub-PR table, finds linked tasks, marks Done
10. **Flags mega PR health** — warns if dev branch has >20 commits or >7 days since last merge to `main`

*Housekeeping:*
11. **Cross-repo PR linking** — matches PRs across backend and frontend repos, comments links
12. **Prunes worktrees** for merged branches to prevent disk bloat
13. **Flags stale PRs** older than 7 days

**What it does NOT do:**
- Never merges PRs targeting `main` — only your dev branch. The mega PR to `main` requires human approval.
- Never reviews code quality — that's the Senior Reviewer's job
- Never takes screenshots

**Fix priority:** Bug comments before Nit comments. Smallest diff first. If a fix would touch more than 3 files, it skips and flags for human attention. If tests fail after a fix, it reverts.

**Standalone use:** Useful on its own for any team that wants automated branch syncing, conflict resolution, and review comment fixes. Pairs naturally with the Senior Reviewer.

---

### 5. Docs Agent

> **Role:** The documentarian and second quality gate. Takes before/after screenshots, annotates them with red arrows, generates blog drafts, and blocks merges if the UI is broken. Trigger: `docs` or `document prs` or `ship it`.

**What it does every 20 minutes:**
1. Finds PRs that have Senior Reviewer approval but no Docs approval yet
2. Navigates affected pages via Playwright and captures "before" screenshots
3. Checks out the PR branch and captures "after" screenshots
4. **Annotates "after" screenshots** with red SVG arrows pointing at the changed elements
5. Uploads screenshots to your storage bucket
6. For `feat:` PRs, generates a blog draft using your content automation tool
7. Posts before/after images on the PR
8. Approves with `Documented` — or blocks with `Documentation blocked` if the UI is broken

**What it does NOT do:**
- Never reviews code logic — that's the Senior Reviewer's job
- Never writes code or fixes bugs
- Doesn't generate docs for backend-only PRs (auto-approves those)

**The screenshot annotation technique:**

The Docs Agent programmatically injects red arrow SVG overlays pointing at changed elements. This makes it immediately obvious what changed without reading any text:

```javascript
// Get the bounding box of the changed element
const box = await page.locator('.changed-element').boundingBox();

// Inject a red arrow SVG overlay pointing at the element
await page.evaluate(({x, y, w, h}) => {
  const svg = document.createElement('div');
  svg.style.cssText = 'position:fixed;top:0;left:0;width:100%;height:100%;z-index:99999;pointer-events:none';
  const tipX = x + w/2;
  const tipY = y + h/2;
  const startX = tipX - 80;
  const startY = tipY - 100;
  svg.innerHTML = `<svg width="100%" height="100%" style="position:absolute;top:0;left:0">
    <defs><marker id="arrowhead" markerWidth="10" markerHeight="7"
      refX="10" refY="3.5" orient="auto">
      <polygon points="0 0, 10 3.5, 0 7" fill="red"/>
    </marker></defs>
    <line x1="${startX}" y1="${startY}" x2="${tipX}" y2="${tipY}"
      stroke="red" stroke-width="5" marker-end="url(#arrowhead)"/>
  </svg>`;
  document.body.appendChild(svg);
}, {x: box.x, y: box.y, w: box.width, h: box.height});

// Screenshot now has the arrow baked in
await page.screenshot({ path: 'after-annotated.png' });
```

**Why docs as a merge gate?**
Tests verify code works. Reviews verify code is correct. But neither verifies that the feature is **documented, visible, and makes sense to stakeholders**. By making documentation a hard gate, you catch: undocumented features, UI regressions that "pass" tests but look wrong, and missing context for anyone who wasn't in the PR discussion.

**Standalone use:** Very useful on its own for any team that wants automated screenshot documentation on PRs. Install the Docs Agent + Playwright MCP for instant visual PR reviews.

---

### 6. Log Watcher

> **Role:** The monitor. Tails your test environment logs every minute and reports actionable errors. Never fixes anything.

**What it does every 1 minute:**
1. Grabs the last 200 lines of logs from each service
2. Filters for actionable signals:
   - `ERROR`, unhandled exceptions, stack traces
   - HTTP 500/502/503 responses
   - Database errors, connection failures
   - Auth failures, token errors
   - Process crashes, OOM kills
3. Skips noise: deprecation warnings, hot-reload messages, health checks, static asset 304s
4. If errors are found, prints a clear summary with timestamp, service, error, and stack trace snippet
5. Correlates errors with recent commits to suggest which change likely caused it
6. Detects crash loops (same error 3+ ticks in a row) and flags as CRITICAL
7. If logs are clean, stays silent

**What it does NOT do:**
- Never fixes code — just reports
- Never creates tasks — this is live feedback, not bug filing
- Never restarts services
- Never runs browser tests — this is server-side only

**Standalone use:** Perfect as a "test buddy" during manual testing sessions. Run it with `/loop 1m` while you're testing your app and get instant feedback on server errors you might miss.

---

### Support Infrastructure (3 additional loops)

Beyond the 6 primary agents, three support loops run in the background:

| Loop | Interval | What it does |
|------|----------|-------------|
| **Notion Sync** | 5m | Bidirectional glue between GitHub and your task tracker. Open PRs → tasks "In Review". Merged PRs → tasks "Done". Flags stale "In Progress" tasks with no recent commits. Dedup sweep every 5th tick. |
| **Rules Hygiene** | 12h | Checks Claude Code version and changelog, reviews rules for staleness against the [Prompt Health](#prompt-health-self-learning-anti-rot) tags, proposes updates. |
| **Swarm Digest** | Daily (8am) | Morning briefing: what shipped, mega PR readiness, high-risk items, today's priorities. Delivered to your digest channel. |

These aren't agents with standalone use cases — they're infrastructure that keeps the swarm healthy. The Notion Sync is especially critical; without it, your task tracker and GitHub drift apart within hours.

---

## Key Concepts

### Dual-Gate Merge

Most teams gate merges on tests and code review. This system adds a second gate: documentation.

**Why?** Because in practice, the problems that slip through code review aren't logic bugs — they're:
- Features that work but nobody documented
- UI changes that "pass" tests but visually regressed
- PRs that three people reviewed but nobody screenshotted

The dual-gate pattern catches these by requiring independent verification from two agents with different perspectives: one reads code, one looks at the UI.

### Prompt Health (Self-Learning Anti-Rot)

AI instructions degrade over time. Rules become outdated, edge cases accumulate, contradictions creep in. Most people don't notice until their agent starts behaving erratically.

This system includes a built-in health system:

**Health tags** — every rule is mentally tagged as:
- **ACTIVE:** Used regularly, still relevant
- **DORMANT:** Not triggered in 2+ weeks — candidate for removal
- **STALE:** References outdated tools/IDs — needs update or deletion

**Self-learning loop** — after every mistake:
1. Check if an existing rule should have prevented it → fix the rule
2. If no rule exists → add a concise one to `lessons.md`
3. If a rule *caused* the mistake → simplify or delete it

**Anti-patterns the system watches for:**
- Same instruction in 2+ files (merge them)
- "MUST" / "NEVER" / "CRITICAL" inflation — if everything is critical, nothing is
- Rules written for a one-time incident (delete after resolving)
- Rules with hardcoded IDs that could go stale (centralize in `reference-ids.md`)

### Risk Classification

Not all code changes deserve the same level of rigor. A CSS tweak doesn't need an integration test. A payment system change needs a rollback plan.

| Risk | Criteria | What's required |
|------|----------|----------------|
| **Low** | UI tweaks, copy changes, CSS | Unit tests |
| **Medium** | New pages, components, frontend logic | Unit tests |
| **High** | External APIs, payments, auth, DB mutations | Unit + integration + manual QA steps + rollback plan |

The Sprint Worker classifies risk automatically based on which files are being changed. High-risk PRs get held for human review regardless of whether both gates pass.

### Dev Branch Pattern

Instead of merging feature branches directly to `main`, everything flows through a dev branch:

```
feature-1 ──┐
feature-2 ──┤──► dev branch ──► mega PR ──► main
feature-3 ──┘    (auto-merge)   (human review)   (production)
```

**Why this matters:**
- Feature branches merge to `dev` automatically after gates pass — fast feedback loop
- `dev` accumulates a batch of changes, then ships to `main` as one mega PR — single review point
- The mega PR gives your CTO/team lead a structured view of everything that shipped, sorted by risk
- Rollback is clean: revert the mega PR to undo everything, or revert individual feature merges on `dev`
- `dev` becomes your "working production" — if a dependency is merged to `dev`, it's available for other features

### Magic Keywords

Instead of remembering which agents to launch with which parameters, natural language keywords orchestrate complex workflows:

| Keyword | What happens |
|---------|-------------|
| `prep sprint` | Queries your task board, proposes 3-5 tasks ranked by impact, waits for your approval |
| `launch swarm` | Starts all 6 agents at their defined intervals (requires curated sprint) |
| `work horses` | Starts just Sprint Worker + PR Watchdog (the execution pair) |
| `senior leadership` | Starts just Task Planner + Senior Reviewer (the quality pair) |
| `ship it` | Starts just the Docs Agent |
| `prep for deploy` | Runs tests, opens a PR, reports status |
| `prep mega pr` | Syncs branches, runs full test suite, opens mega PR with sub-PR table |

These work because Claude Code's `/loop` feature runs agents on recurring intervals. Each keyword maps to a specific combination of `/loop` commands.

### Separation of Concerns

Every agent has strict boundaries. This is intentional and critical:

| Agent | Does | Does NOT |
|-------|------|----------|
| Task Planner | Reads code, writes plans | Write code, open PRs |
| Sprint Worker | Writes code, opens PRs | Take screenshots, review code |
| Senior Reviewer | Reviews code, posts comments | Fix code, merge PRs |
| PR Watchdog | Fixes comments, merges PRs | Review code quality |
| Docs Agent | Screenshots, blog drafts | Fix bugs, review logic |
| Log Watcher | Reports errors | Fix code, create tasks |

**Why this works better than a "do everything" agent:**
- No conflicts — agents don't step on each other's work
- Clear accountability — when something goes wrong, you know which agent's rules to fix
- Composable — you can run any subset of agents independently
- Debuggable — each agent's behavior is defined in one file

---

## Installation

### Quick Start (5 minutes)

```bash
# Option A: Clone directly as your ~/.claude folder (recommended)
git clone https://github.com/harshmoney123/launch-swarm.git ~/.claude

# Option B: Clone somewhere and copy the files
git clone https://github.com/harshmoney123/launch-swarm.git
cp -r launch-swarm/{CLAUDE.md,rules,settings.json} ~/.claude/

# Option C: Clone into a specific project's .claude
git clone https://github.com/harshmoney123/launch-swarm.git /path/to/your/project/.claude
```

### Configure Your IDs

Open `.claude/rules/reference-ids.md` and replace the `YOUR_*` placeholders with your actual IDs:

```markdown
## Your Task Tracker
- Sprint board: YOUR_COLLECTION_ID
- Current Sprint view: YOUR_CURRENT_SPRINT_VIEW_ID
- Master Table view: YOUR_MASTER_TABLE_VIEW_ID

## Your Communication
- Digest channel: YOUR_SLACK_CHANNEL_ID
- Feature announcements: YOUR_DISCORD_WEBHOOK_URL

## Your Storage
- Screenshot bucket: YOUR_SCREENSHOT_BUCKET
```

See [`.private.example/reference-ids.md`](.private.example/reference-ids.md) for the full template with all fields explained.

### Private Overrides

For IDs and secrets you don't want in version control:

```bash
# Create a .private directory (gitignored)
mkdir -p ~/.claude/.private/

# Copy the example and fill in your real values
cp launch-swarm/.private.example/reference-ids.md ~/.claude/.private/reference-ids.md
```

Then update your `CLAUDE.md` to reference `.private/reference-ids.md` for actual values.

### Verify It Works

```bash
# Open Claude Code in your project
claude

# Try a magic keyword
> prep sprint
```

If your task tracker is connected, Claude will query your sprint board and propose tasks. If not, it will tell you what's missing.

---

## What You Need

The swarm requires a few infrastructure pieces to function. **The specific tools don't matter — the concepts do.** This section explains what each piece does and why it's needed, with examples of tools that work.

### 1. A Sprint Board / Task Tracker

**Why:** The swarm needs a source of truth for "what should I work on?" and "what's the status of each task?" Without this, the Sprint Worker has nothing to pull from and the Watchdog can't update task status on merge.

**What it needs to support:**
- A "Current Sprint" view that filters to active tasks
- Status fields (To-do, In Progress, In Review, Done)
- Assignee/owner fields (so agents don't touch other people's tasks)
- An API that Claude Code can access via MCP tools

**Options:**
| Tool | MCP Available | Notes |
|------|--------------|-------|
| **Notion** (what I use) | Yes (native) | Great for sprint boards with custom views |
| **Linear** | Yes | Clean API, built for sprint workflows |
| **Jira** | Yes (community) | Enterprise standard, heavier setup |
| **GitHub Projects** | Yes (via `gh` CLI) | Zero additional tools if you're already on GitHub |
| **Trello** | Yes (community) | Simple but works for small teams |

To swap: Update `notion-tracking.md` and `reference-ids.md` with your tool's API patterns. The status flow (To-do → In Progress → In Review → Done) and query patterns stay the same.

### 2. Screenshot / Asset Storage

**Why:** The Docs Agent needs somewhere to upload before/after screenshots so they can be linked in PR comments and blog posts. Without this, Gate 2 can't function.

**What it needs to support:**
- Public-readable URLs for uploaded images
- Programmatic upload (CLI or API)

**Options:**
| Tool | Notes |
|------|-------|
| **S3** (what I use) | Standard, cheap, public-read ACLs |
| **Cloudflare R2** | S3-compatible, no egress fees |
| **GitHub PR attachments** | Free, no setup, but less control |
| **Imgur API** | Quick and dirty for personal projects |

To swap: Update the upload paths in `loop-docs.md` and `reference-ids.md`.

### 3. A Communication Channel

**Why:** The daily digest, critical alerts, and feature launch announcements need somewhere to land. Without this, you're checking GitHub manually for pipeline status.

**Options:**
| Tool | Notes |
|------|-------|
| **Slack** (what I use) | Native MCP, channels + DMs |
| **Discord** | Webhook-based, great for small teams |
| **Email** | Works but slower feedback loop |
| **GitHub Discussions** | Zero additional tools |

To swap: Update channel IDs in `reference-ids.md` and webhook URLs in `loop-docs.md`.

### 4. A Content Automation Tool (for the Docs Agent)

**Why:** The Docs Agent generates blog drafts for `feat:` PRs. This requires a tool that can take screenshots and context and produce a user-facing writeup.

**Options:**
| Tool | Notes |
|------|-------|
| **[Emma](https://www.agentweb.pro)** (what I use) | AgentWeb's AI marketing agent — takes screenshots + context, outputs blog drafts and user guides. Full disclosure: I built this. |
| **Any LLM API** | Send screenshots + PR context to Claude/GPT and get a draft back |
| **Manual** | Skip the auto-draft, just require screenshots as the gate |

To swap: Update the content tool references in `auto-user-guide.md` and `loop-docs.md`.

### 5. Browser Automation (Playwright MCP)

**Why:** The Docs Agent needs to navigate your app, capture screenshots, and inject SVG annotations. The Log Watcher can optionally use it for browser-side monitoring.

```bash
# Install the Playwright MCP server for Claude Code
# Add to your .claude/settings.json or install via Claude Code plugin marketplace
```

This is the one dependency that's hard to swap — the screenshot annotation technique is Playwright-specific. If you're not using the Docs Agent, you don't need this.

### 6. GitHub CLI (`gh`)

**Why:** The PR Watchdog, Sprint Worker, and Senior Reviewer all use `gh` to create PRs, list open PRs, merge PRs, and post comments. This is essential for any agent that interacts with GitHub.

```bash
# Install
brew install gh  # macOS
# or see https://cli.github.com/

# Authenticate
gh auth login
```

### 7. A Test Environment (Optional)

**Why:** The Log Watcher monitors a running instance of your app. The Docs Agent captures screenshots from a live environment. Without this, those two agents can't function, but the other four still work fine.

This can be anything: a local dev server, a Docker Compose stack, a staging environment on AWS/Vercel/Fly.io.

---

## Mix and Match

You don't need the full swarm. Each agent is independent. Here are common configurations:

### "I just want better code review"
**Use:** Senior Reviewer + PR templates (`loop-pr-ops.md` + `loop-back-testing.md`)

Install just these two files and you get deep automated code review with severity-tagged comments on every PR. No task tracker needed.

### "I just want automated documentation"
**Use:** Docs Agent + Playwright MCP (`loop-docs.md`)

Install the Docs Agent and get before/after screenshots with annotated arrows on every PR. Great for teams where "nobody screenshots the PR" is a recurring problem.

### "I just want an autonomous coder"
**Use:** Sprint Worker + Task Planner (`loop-sprint-worker.md` + `loop-support.md`)

The Planner architects the approach, the Worker codes it. You still review and merge manually. Requires a task tracker.

### "I want the minimum viable pipeline"
**Use:** Sprint Worker + PR Watchdog (`loop-sprint-worker.md` + `loop-pr-ops.md`)

This is the **"work horses"** combo. The Worker codes and opens PRs, the Watchdog fixes review comments and merges. Add the Senior Reviewer if you want automated code review in the loop.

### "I want the quality layer without the coding"
**Use:** Task Planner + Senior Reviewer (`loop-support.md` + `loop-pr-ops.md`)

This is the **"senior leadership"** combo. The Planner architects every task before anyone touches code. The Reviewer catches problems after. You or your team still write the code.

### "I want the full swarm"
**Use:** All 6 agents (`launch swarm`)

The complete pipeline: plan → code → review → document → merge → monitor. Requires all infrastructure pieces.

---

## Magic Keywords

These are natural language triggers you type into Claude Code to orchestrate the swarm:

### `prep sprint`

**What happens:**
1. Claude queries your full task board
2. Filters out epics, blocked tasks, and tasks owned by others
3. Proposes 3-5 tasks ranked by impact, with one-line rationales
4. **Waits for your approval** — never auto-assigns sprint tasks
5. Approved tasks get moved to Current Sprint

**Why the human gate matters:** The swarm should never decide what to work on. It proposes; you decide. This prevents the common failure mode of autonomous agents grinding through low-priority busywork while ignoring what actually matters.

### `launch swarm`

**What happens:**
1. Verifies Current Sprint is curated (won't launch on an empty sprint)
2. Starts all 6 agents at their defined intervals:
   - Sprint Worker: `/loop 30m`
   - Task Planner: `/loop 15m`
   - Senior Reviewer: `/loop 20m`
   - PR Watchdog: `/loop 10m`
   - Docs Agent: `/loop 20m`
   - Log Watcher: `/loop 1m`
3. Agents begin working through the sprint autonomously

### `work horses`

Starts Sprint Worker + PR Watchdog. The execution pair — for when you want code written and merged without the overhead of the full pipeline.

### `senior leadership`

Starts Task Planner + Senior Reviewer. The quality pair — for when you want every task planned and every PR deeply reviewed, but you're writing the code yourself.

### `prep mega pr`

**What happens:**
1. Syncs your dev branch with `main`
2. Runs the full test suite
3. Verifies all sub-PRs passed both gates
4. Opens a mega PR from dev → main with a structured description:
   - Sub-PR table sorted by risk
   - High-risk items highlighted
   - Aggregate "What was NOT tested"
   - Rollback instructions

---

## PR Templates

### Reviewer-First Format

Every PR opened by the Sprint Worker follows this format:

```markdown
## Risk: Low / Medium / High | Priority: High / Medium / Low

## What was wrong (before)
[1-2 sentences describing the problem]

## What this fixes (after)
[1-2 sentences describing the solution]

## Test Evidence
- Unit tests: X passed, 0 failed
- Tested as: [which accounts/roles]

## Files changed
[List of files, max 10. Grouped by area if more.]

## What was NOT tested
[Honest list of what wasn't covered]

## Related PRs
- Backend: [#NNN or "standalone"]
- Frontend: [#NNN or "standalone"]

## Task link
[Link to sprint board task]
```

**Why each field exists:**

- **Risk/Priority:** Tells the reviewer where to focus attention. High-risk PRs get reviewed first.
- **Before/After:** Forces the author to articulate the problem and solution in plain language. If you can't explain it in 2 sentences, the PR is too big.
- **Test Evidence:** Proves tests actually ran. Not just "tests pass" but which tests and how many.
- **What was NOT tested:** This is the most important field. Every PR has blind spots. Making them explicit means the reviewer knows exactly where to look harder, and the QA plan can cover the gaps. Honesty here prevents production surprises.
- **Related PRs:** Cross-repo changes (e.g., backend + frontend) need to ship together. This links them.
- **Task link:** Bidirectional linking between code and task tracker. Anyone on the task can find the PR; anyone on the PR can find the task.

### Mega PR Format

```markdown
## Mega PR: dev → main

## Summary
[X tasks — Y features, Z fixes. All passed both gates.]

## Sub-PRs (sorted by risk, then priority)
| # | Risk | Priority | Type | Title | PR | Reviewer | Docs | Task |
|---|------|----------|------|-------|----|----------|------|------|
| 1 | High | High | feat | ...   | #N | Pass     | Pass | link |

## High-Risk Items — start here
- **#N — [title]**: [why risky]. Rollback: [how].

## Test Status
- Full suite on dev: X passed
- All sub-PRs: both gates passed

## What was NOT tested
[Aggregate across all sub-PRs]

## Rollback
- Single task: `git revert <merge-commit>` on dev
- Everything: `git revert -m 1 <mega-merge>` on main
```

### Multi-Account Test Matrix

Every PR should be tested from multiple perspectives:

| Feature type | Test WITH (should work) | Test WITHOUT (should be denied) |
|-------------|------------------------|--------------------------------|
| Admin-only | Admin account | Basic user → verify 403 |
| Tier-gated | Pro/privileged user | Basic user → verify upgrade prompt |
| All-user | Basic user (lowest tier) | N/A |

This ensures you're testing both the happy path AND the access control. A feature that "works" but is visible to users who shouldn't see it is a security bug.

---

## Customization Guide

### Swapping Your Task Tracker

The system references Notion throughout, but the pattern is tool-agnostic. To swap:

1. **Update `notion-tracking.md`** — rename it and replace Notion-specific API calls with your tool's equivalents
2. **Update `reference-ids.md`** — replace Notion view IDs with your tool's board/project IDs
3. **Keep the status flow** — To-do → In Progress → In Review → Done works regardless of tool
4. **Keep bidirectional linking** — every PR links to a task, every task links to its PR

**Key behaviors to preserve:**
- The Sprint Worker queries for `To-do` tasks and claims them by setting `In Progress`
- The Watchdog sets tasks to `Done` when their PR merges
- The Planner only touches tasks that don't have plans yet (`Planned !== true`)

### Swapping Your Communication Channel

Replace Slack references in `reference-ids.md` with your channel's webhook URL or API endpoint. The digest format (TL;DR → What shipped → High-risk → Priorities) stays the same.

### Adding Your Own Agents

Create a new file in `.claude/rules/` following this pattern:

```markdown
# Agent Name — "trigger keyword"

`/loop INTERVAL`. [One sentence describing the role.]

## Each tick

1. [What to check/query]
2. [What to do with the results]
3. [What to report/update]

## What NOT to do

- [Explicit boundaries]
```

Then add it to the agents table and combos in `loop-modes.md`.

### Modifying the Gate System

The dual-gate system is defined in `loop-modes.md` and enforced by the PR Watchdog in `loop-pr-ops.md`. To change it:

- **Remove a gate:** Delete the check from the Watchdog's auto-merge conditions
- **Add a gate:** Add a new check (e.g., "Security Scan Passed") to the Watchdog's merge requirements and create the agent that provides it
- **Change gate order:** The Docs Agent currently waits for Senior Reviewer approval before starting. To run them in parallel, remove the "skip PRs without Senior Reviewer Approve" rule in `loop-docs.md`

---

## FAQ

**Q: Does this require Claude Code Pro/Max?**
A: The swarm uses Claude Code's `/loop` feature for recurring agents. Check Claude Code's current pricing for loop/cron support. The individual rule files work with any Claude Code tier.

**Q: How much does this cost to run?**
A: It depends on your loop intervals and how many agents are active. The full swarm with default intervals uses significant context across 6 concurrent loops. Start with 1-2 agents and scale up. The "work horses" combo (Sprint Worker + Watchdog) is the most cost-effective starting point.

**Q: Can I use this with Cursor / Codex / other AI coding tools?**
A: The rule files are Claude Code-specific (they reference Claude Code features like `/loop`, `EnterWorktree`, and subagents). The *concepts* (dual-gate, risk classification, separation of concerns) apply to any tool. Community ports are welcome via PR.

**Q: What if I don't use Notion?**
A: See [Customization Guide](#customization-guide). The system is designed around concepts (sprint board, status flow, bidirectional linking), not specific tools. Swap the Notion references for your tool of choice.

**Q: How do I prevent the agents from breaking my codebase?**
A: Multiple layers of protection:
- Sprint Worker always runs tests before opening a PR
- Senior Reviewer does deep code review before approving
- PR Watchdog only merges after both gates pass
- Nothing auto-merges to `main` — only to your dev branch
- Mega PRs require human approval
- High-risk PRs are held for manual review regardless of gates

**Q: Can I run this with a team, not just solo?**
A: Yes — I run it on a 4-person team. The owner filter (agents only touch tasks assigned to you) prevents conflicts. Each team member can run their own swarm instance on their own tasks. The PR Watchdog handles cross-agent coordination (merge conflicts, branch syncing).

**Q: My prompts keep going stale. How does prompt health actually work?**
A: See [Prompt Health](#prompt-health-self-learning-anti-rot). The short version: every rule gets tagged ACTIVE/DORMANT/STALE. After any mistake, you check whether a rule should have prevented it and fix or add one. The Rules Hygiene agent reviews all rules every 12 hours and flags drift. This turns prompt maintenance from "maybe I'll update it someday" into a systematic process.

**Q: What's the "Investigate:" task pattern?**
A: Prefix any task with "Investigate:" and the Sprint Worker switches to research-only mode — reads code, checks logs, writes findings, but never opens a PR. This creates a deliberate pause point where you review the findings before deciding to proceed. It prevents the common failure of agents charging ahead with a fix for a problem they don't fully understand.

---

## Priority System

There are **two different orderings** for two different purposes:

### Sprint Curation Order ("prep sprint")

When deciding **what to work on**, prioritize by business value with least friction:

```
High Priority + Low Risk   →  first  (ship value fast, low blast radius)
High Priority + Med Risk   →  second
Med Priority  + Low Risk   →  third
High Priority + High Risk  →  fourth
Med Priority  + Med Risk   →  fifth
...
```

You want to deliver the most impactful stuff with the least chance of blowing up first.

### Execution/Review Triage Order (Sprint Worker, Senior Reviewer)

When deciding **what to review or merge next** from the active queue, prioritize by risk:

```
High Risk + Low Priority  →  first  (get scary stuff eyeballed early)
High Risk + Med Priority  →  second
Med Risk  + Low Priority  →  third
High Risk + High Priority →  fourth
Med Risk  + Med Priority  →  fifth
Med Risk  + High Priority →  sixth
Low Risk  + Low Priority  →  seventh
Low Risk  + Med Priority  →  eighth
Low Risk  + High Priority →  ninth
```

**Why risk-first for execution:** High-risk changes are the ones most likely to cause problems. Getting them reviewed early gives you more time to catch issues before they compound. A low-priority database migration deserves more scrutiny than a high-priority copy update.

Tiebreaker: CI green first, fewer unresolved review comments first.

---

## Sprint Lifecycle

The complete lifecycle from planning to production:

```
1. "prep sprint"     →  You curate 3-5 tasks
2. "launch swarm"    →  6 agents start working
3. Sprint Worker     →  Codes + tests + opens PRs
4. Senior Reviewer   →  Reviews PRs (Gate 1)
5. Docs Agent        →  Screenshots + blog drafts (Gate 2)
6. PR Watchdog       →  Auto-merges after both gates
7. Swarm goes idle   →  QA handoff generated automatically
8. You test manually →  Using the structured QA plan
9. "prep mega pr"    →  Opens dev → main PR
10. CTO/lead reviews →  Human approval on mega PR
11. Merge to main    →  Feature launch announced
```

**The swarm knows when to stop.** When all Current Sprint tasks are done, the Sprint Worker goes idle. The system detects this and automatically generates a structured QA handoff with step-by-step test plans for every task that shipped. It doesn't keep grinding or pull from the backlog.

**Between sessions:** Clear Done tasks from Current Sprint before curating new ones. This prevents the sprint view from accumulating stale completed items that confuse the agents.

---

## File Reference

| File | Purpose |
|------|---------|
| `.claude/CLAUDE.md` | Core rules: git workflow, TDD, lifecycle, magic keywords, priority matrix |
| `.claude/rules/loop-modes.md` | Agent definitions, intervals, combos, gates, sprint complete flow |
| `.claude/rules/loop-sprint-worker.md` | Sprint Worker: coding agent with risk classification |
| `.claude/rules/loop-pr-ops.md` | PR Watchdog + Senior Reviewer + Daily Digest |
| `.claude/rules/loop-docs.md` | Docs Agent: screenshots, annotations, blog drafts, feature launches |
| `.claude/rules/loop-log-watcher.md` | Log Watcher: real-time error monitoring |
| `.claude/rules/loop-support.md` | Task Planner + Notion Sync + Rules Hygiene |
| `.claude/rules/loop-back-testing.md` | PR templates: Reviewer-First and Mega PR formats |
| `.claude/rules/session-hygiene.md` | Context management: compact, clear, commit, prune |
| `.claude/rules/prompt-health.md` | Self-learning anti-rot system |
| `.claude/rules/notion-tracking.md` | Task tracking discipline and querying patterns |
| `.claude/rules/reference-ids.md` | All hardcoded IDs in one place (template) |
| `.claude/rules/auto-user-guide.md` | Auto-generated user guides for feature PRs |
| `.claude/rules/meeting-action-routing.md` | Meeting action item extraction and routing |
| `.claude/rules/loop-senior-leadership.md` | Combo: Task Planner + Senior Reviewer |
| `.claude/rules/loop-work-horses.md` | Combo: Sprint Worker + PR Watchdog |
| `.claude/settings.json` | Claude Code settings (Playwright plugin) |
| `.private.example/reference-ids.md` | Example private config with placeholder values |

---

## Contributing

This is a living system. I update it as I find better patterns.

**Ways to contribute:**
- **Bug fixes:** If a rule has a logic error or contradicts another rule, open a PR
- **New agents:** Built a useful agent? Add it to `.claude/rules/` with proper boundaries
- **Tool adapters:** Made this work with Linear/Jira/GitHub Projects? Share the modified files
- **Concept improvements:** Better gate systems, new health check patterns, improved priority algorithms

**Please don't:**
- Add rules that are specific to your project (keep it general)
- Add emojis or formatting flourishes to the rule files (Claude reads these, not humans)
- Combine agents that should stay separate (separation of concerns is core to the design)

---

## Credits

Built by [Harsha Vankayalapati](https://www.linkedin.com/in/harsha-vankayalapati/) — Co-Founder & CEO at [AgentWeb](https://www.agentweb.pro). YC W23 founder. Part-time coder, full-time CEO.

This system was built because I needed to stay dangerous as an engineer while running sales, marketing, and ops for a 4-person startup. Every rule exists because something broke without it. Every boundary exists because an agent overstepped without it.

The Docs Agent in this system uses [Emma](https://www.agentweb.pro) (AgentWeb's AI marketing agent) for automated blog drafts and user guide generation. If you're looking for an AI-powered content pipeline that integrates with your development workflow, check it out.

---

## License

MIT. Use it, fork it, adapt it, share it.

If this helped you, a star helps others find it.
