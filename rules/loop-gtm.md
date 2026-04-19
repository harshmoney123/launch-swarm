# GTM Loop -- "morning gtm" or "gtm briefing"

The marketing/content/ops arm of the swarm. Separate from the engineering loops — this one touches posts, launches, outreach tracking, and external comms, not code.

## Philosophy

The engineering swarm ships code autonomously. GTM is NOT autonomous. Posts go out in the founder's voice, on the founder's timing, with the founder's taste. Outreach and ads stay under human judgment.

The swarm's job is to (1) **remind** the founder to review the things that matter, (2) **track** numbers the founder pastes in so trends become visible, and (3) **stage** content so the founder can pick, edit, and ship fast.

**Never auto-post. Never auto-reply. Never touch ad platforms.**

## What the briefing does

Four things, in order, every morning:

### 1. Diagnosis — "Demand Gen or Demand Capture?"

The primary axis. Every GTM problem is one of two things, and the fix for each is opposite:

- **Demand Gen problem** = not enough people know we exist / not enough top-of-funnel. Symptom: **founder is NOT swamped with sales calls.** Fix: more posts, more outreach volume, more ads, more cold prospects, lead magnets.
- **Demand Capture problem** = lots of attention/leads but they don't convert. Symptom: founder IS swamped with calls/inbound but close rate is low, or signups aren't activating. Fix: better landing pages, sharper sales script, fix activation flow, follow-up sequences, pricing experiments.

**The default assumption at our stage is Demand Gen.** We do not have enough volume to even diagnose Capture problems yet. If in doubt, the answer is "more volume."

Read `~/.claude/gtm/metrics-log.md` (if it exists) and decide: Gen or Capture?

Output format:

```
## GTM State — {date}

**Bottleneck**: Demand Gen | Demand Capture

**Why**: {2-3 sentences citing specific evidence, e.g.:
  "Sales calls this week: 1. Inbound signups: 3. Outreach sends: ~40.
   This is a Demand Gen problem — we don't have enough top-of-funnel to know
   if Capture is working. Volume is the bottleneck before optimization."}

**Today's priority**: {concrete action tied to Gen or Capture}
```

Sub-diagnoses (only relevant once Gen vs Capture is identified):
- Gen + posts low → ship more content, post 5+/week
- Gen + outreach low → blast 1,000+ prospects, don't curate
- Gen + ads dead → relaunch creatives, push impression volume
- Capture + low close rate → sales script + landing page audit
- Capture + low activation → onboarding flow review
- Capture + churn → product feedback loop

The diagnosis becomes today's **priority frame**. The rest of the briefing serves it.

### 2. Priority-driven content slot
Pick the content draft based on diagnosis, not just age. If lead gen is dry, maybe today's post should be the lead magnet announcement. If writing quality is the problem, today is a rewrite day on a queued draft, not a new post. The swarm recommends AND explains why.

Then run the existing content flow: recommend → wait for pick → draft → rate hooks → stage. Spec below in "Content command flow."

### 3. Ops checklist (diagnosis-informed)
A checklist the founder runs through. Items scale with diagnosis:

**Default (green state):**
```
Today's ops review:
  [ ] Ads — still running? Any broken? Daily spend vs budget? (target: >10K daily impressions per campaign)
  [ ] Outreach — any replies needing response? How many?
  [ ] LinkedIn connections — send ~35/day (LinkedIn's safe-zone cap; going higher risks the account)
  [ ] LinkedIn DMs — send to entire follow-up cohort, not a curated 10
  [ ] Cold outreach — is the sequence running at full volume? (target: 100+ sends/day across all channels)
  [ ] Upcoming calls (next 24-48h) — who's up? Prep needed?
```

**Top-of-funnel reminder**: The check is "is the top of the funnel still being filled today?" not "are we hitting some big number." A dry pipeline with no new prospects, posts, or impressions added today is the actual failure mode. Curation is fine — gaps are not.

**Top-of-funnel principle**: The failure mode is letting the top of the funnel run dry, not under-blasting. Hand-curation is fine. Signal and quality are fine. What is NOT fine is going days without adding new prospects, posts, or impressions to the top of the funnel. Constantly grab fresh leads at the top — curation OK, dry pipeline NOT OK.

**Diagnosis-triggered extras:**
- Lead gen dry → "Add fresh prospects to the top of the funnel today. Curate to ICP fit, don't blast — but make sure SOMETHING new is entering. Apollo / Clay / hand-built — whatever the source, the rule is no day with zero new prospects added."
- Outreach response rate low → "Rewrite top-of-funnel DM. Test a new angle on the next batch. Don't change everything at once — change the hook, hold the rest constant."
- Ads broken → "Audit campaign, pause dead creatives. Check daily impression volume — if it's near zero, it's not a creative problem, it's a budget/targeting problem."
- Content engagement low → "Post more consistently, not necessarily more often. The failure mode is gaps, not low cadence. Aim for steady 3-5 posts/week before tweaking hook patterns."

The founder either reports numbers back ("ads: $47 spent, 2 leads, 1 campaign paused. outreach: 6 replies, responded to 4. sent 12 connections.") or says "skip ops" if they're not doing it that day.

### 4. Metrics log (when numbers are provided)
Append one line per metric category to `~/.claude/gtm/metrics-log.md`:

```
## 2026-04-06
- ads: $47 spent, 2 leads, 1 campaign paused (reason: CTR < 0.5%)
- outreach: 6 replies in, 4 responded, 2 queued for tomorrow
- connections: 12 sent (target was 10-15)
- dms: 8 sent (target was 10)
- calls: 2 sales calls tomorrow — Acme Corp, BetaCo
- notes: paused the retargeting campaign until we fix the CTR
```

Over time this becomes a trend document. On request ("show me ads trend" / "how's outreach been this week"), the swarm reads the log and summarizes.

## State files

- `~/.claude/gtm/post-ideas.md` — content queue (format in its own spec)
- `~/.claude/gtm/metrics-log.md` — daily metrics history, append-only (founder-pasted)
- `~/.claude/gtm/activity.log` — one line per briefing invocation + action taken

## Content command flow (`morning gtm` → content section)

1. Read `~/.claude/gtm/post-ideas.md`.
2. List all `queued` and `drafted` ideas with one-line summaries. Group by platform. Flag any idea queued >14 days as rotting.
3. Recommend **one** idea to draft today based on: age, pairing opportunities, external context, founder's stated priorities.
4. On founder approval, draft:
   - 3 hook variants, each rated 1-5 on hook strength / risk / differentiation
   - Full post body in founder's voice
   - CTA line
   - Inline image/screenshot suggestions
   - Safety flags if the post names competitors or vendors
5. Founder picks hook + edits inline. Swarm updates the idea block:
   - `status: drafted`, append `**draft**:` section
6. On "schedule it" → flip `status: scheduled`, record date. **Never touch LinkedIn/X APIs.**
7. After founder manually posts and confirms, flip `status: posted`, move block to `## Posted` section.

## Shortcut commands

- `gtm deprio <idea>` — move to `deprio` with 30-day revisit note
- `gtm add <topic>` — append new queued block with today's date
- `gtm log <paste>` — skip the briefing, just append metrics to the log (for mid-day updates)
- `gtm trend <category>` — read metrics-log.md, summarize the last 7-14 days for the named category

## Guardrails

- **Agent vs Claude Code for drafting**: For solo drafts in the founder's voice, Claude Code with voice examples. Agent only when the task needs multi-tool orchestration (pull news + cross-reference ideas + rank). Don't force Agent in.
- **No auto-publish, ever.** Every post waits for explicit founder approval.
- **No auto-reply to outreach.** The briefing surfaces the reminder, the founder responds manually.
- **No touching ad platforms.** Not Meta, not LinkedIn Ads, not Google. Ever.
- **Risk flagging**: any post that names competitors, vendors, or former tools by name gets a `risk: high` note with a safer rephrase alongside.
- **Voice fidelity**: never soften the founder's voice. No hedging, no "excited to share," no corporate LinkedIn tone. If the founder wouldn't say it to a friend, don't write it.
- **One content idea per session.** Don't draft three posts in one run. Focus beats volume.

## Relationship to engineering loops

GTM loop is fully independent. It does NOT touch:
- `YOUR_DEV_BRANCH` or any code branches
- Notion sprint board (engineering tasks only)
- The PR pipeline
- Test environments

It DOES read from, but never writes to:
- Recent merged PRs (for feature launch post material)
- The docs agent's annotated screenshots (for post imagery)

Feature Launch posts from `loop-docs.md` remain separate — those are automated team-facing announcements to Discord. GTM loop is founder-facing external content and ops.

## Future integrations (not built, deliberately)

When the manual checklist has been run daily for 2+ weeks AND the founder confirms the rhythm is real, consider wiring:

- **Calendar peek** via gcal MCP — auto-list next 24h of meetings, flag sales calls
- **Outreach replies** via Unipile API — auto-count unread reply threads
- **Ads pulse** via Meta/LinkedIn Ads API — daily spend + active campaign count
- **Metrics trends** via structured JSONL instead of markdown log

Do NOT build any of these until the manual version has actually been used. The point of the manual flow is to discover which metrics the founder actually cares about before investing in integrations that will go stale.
