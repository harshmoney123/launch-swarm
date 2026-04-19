# AFK vs Live -- Task Routing Rules

Every task in the backlog falls into one of two buckets. Getting this right is the difference between you waking up to shipped PRs and you waking up to a pile of half-done work waiting on his hands.

## The test

**Can the swarm verify "done" without you present?**

- Yes -> **AFK-able**. Eligible for `prep sprint` / `launch swarm`. Verified via tests + Playwright + git.
- No -> **Requires: Me**. Eligible for `prep session`. Batched into a focused human work block.

"External integration" is NOT the test. A task that calls the Gemini API from a new Agent tool is external but fully AFK-able (the swarm has the env var, the test asserts the response). A task that needs you to sign up for Okareo is NOT AFK-able -- the swarm has no hands to click OAuth.

## Requires: Me categories (session subtags)

Tag a task with `Requires: Me` AND one of the session subtags below.

| Subtag | Meaning | Examples |
|---|---|---|
| `Session: Creds` | Needs you to log in, sign up, grant OAuth, or verify against a gated external account | SmartLead outbound links, Okareo signup, Foreplay API keys, GCP Cloud Run deploy, LinkedIn auth |
| `Session: Taste` | Needs your judgment on design, copy tone, brand voice, or strategy. Swarm can propose options but can't pick | Landing page redesign, conditional onboarding flow design, email drip copy, Tinder-style creative voting UX |
| `Session: Money` | Touches real money in a way that's irreversible or blast-radius risky. Even if the code is "done", you verify live | Stripe anchor billing, Stripe Agentic Commerce, any live charge flow |
| `Session: CTO` | Blocked on the CTO's approval (auth, Agent core, MCP, security) | Auth middleware changes, Agent core rewrites, MCP server changes |
| `Session: Device` | Only reachable from your laptop (SSO, local env, physical device) | Tests behind Google SSO, iOS-specific debugging |

A task can have `Requires: Me` without a session subtag if none fit -- default is treat as `Creds`.

## What AFK-able looks like

Pure code/UI work the swarm can verify itself:
- Bug fixes with a clear repro (repro passes = done)
- Refactors with test coverage (snapshot tests catch regressions)
- New components, new pages, new API routes (Playwright screenshots + unit tests)
- Data migrations on `Dev-*` tables (never prod)
- Test writing
- Dependency bumps
- Documentation, blog drafts
- Calls to external APIs where the key is already in env and a test asserts the response

Key signal: **if the PR passes CI + Senior Reviewer + Docs, you trusts it enough to merge without manual QA**, it's AFK-able.

## What Requires: Me looks like

Not a verification problem, a *hands* problem:
- "Swarm built 3 landing page variants, pick one" -- swarm can't pick taste
- "Link SmartLead to the outbound pipeline" -- swarm has no SmartLead login
- "Verify the new Stripe upgrade flow with a real test card" -- swarm won't use a real card
- "Does this copy match how customers actually talk?" -- swarm doesn't have the call recordings in context
- "The new feature is behind Google SSO, test it" -- swarm can't auth as you

## `prep sprint` filter + sizing rule

```
WHERE Status IN ("To-do", "Not started")
  AND Owner IN (you, unassigned)
  AND NOT has_tag("Requires: Me")
```

**Sizing floor: 20 SP minimum per sprint.** Keep adding tasks until >= 20. Overshooting slightly is fine; undershooting is not. A sprint of 16 SP is incomplete.

**Priority order (top to bottom, exhaust each tier before moving down):**
1. **Full AFK-able epics** — an entire 13-21 SP epic that can ship without you. Rare but ideal because it closes a whole initiative.
2. **Prelim tasks inside epics** — phase tasks, foundation work, admin UIs, data model setup. Shipping these unblocks the rest of the epic.
3. **High priority** AFK-able tasks
4. **Medium priority** AFK-able tasks
5. **Low priority** AFK-able tasks

**Risk tiebreaker within each tier:** prefer **Low Risk first**, then Medium, then High. Reason: you sometimes ships without manually verifying, and Low Risk is safe to ship blind. High Risk should only be picked when the tier is otherwise empty.

**Epic handling:** A 13+ SP epic should NOT be added as a single task. If the epic has sub-tasks, pull its sub-tasks individually. If it doesn't, flag it for breakdown rather than skipping it entirely.

If a task is ambiguous (no clear AFK/Live signal), the planner asks you before adding it.

## `prep session` filter

```
WHERE Status IN ("To-do", "Not started")
  AND Owner IN (you, unassigned)
  AND has_tag("Requires: Me")
GROUP BY session_subtag
ORDER BY blocker_impact DESC  -- tasks unblocking the most AFK work go first
```

Output format:

```markdown
## Live Session Queue -- {date}

### Session: Creds (15-30 min) -- unblocks N AFK tasks
- [ ] Link Okareo -> unblocks "Build Agent Benchmarking & Eval System" (8 SP)
- [ ] Link SmartLead -> unblocks "Fix broken lead magnet CTA links" (2 SP)
- [ ] OAuth to Foreplay -> unblocks "Agent: Add Foreplay API tool call" (5 SP)

### Session: Taste (20-40 min)
- [ ] Pick landing page direction -- swarm has 3 variants in PR #NNN
- [ ] Review email drip tone -- draft on PR #NNN

### Session: Money (10 min)
- [ ] Test Stripe upgrade flow with real card on test env

### Session: CTO (5 min)
- [ ] Ping the CTO on PR #NNN (auth middleware change, blocked 3 days)
```

You do this queue in one focused block, then the AFK swarm gets unblocked.

## Tagging rules

- When creating a new Notion task, the creator (human or planner agent) MUST decide AFK vs Live and tag accordingly.
- When reviewing the backlog, if a task is untagged, default to AFK-able only if the planner can confidently confirm it. Otherwise flag as "needs tagging" and surface to you.
- If a task starts AFK but hits a wall that needs you (e.g., "I need an API key to finish"), the Sprint Worker re-tags it `Requires: Me` with the right session subtag and stops work, rather than opening a half-done PR.

## Anti-patterns

- **Don't tag everything `Requires: Me` out of caution.** That defeats the whole AFK pipeline. Only tag when the swarm genuinely can't verify done.
- **Don't tag AFK-able tasks as Live just because they're scary.** High Risk is a separate axis -- a scary refactor is still AFK-able if tests cover it.
- **Don't lump multiple session types into one task.** If it needs both credentials AND taste, break into sub-tasks.
