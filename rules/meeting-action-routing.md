# Rule 15: Meeting Action Item Routing

Pull action items from meeting sources and route to the correct system. Runs every 3rd Notion Sync tick.

## Sources

1. **Fathom** (source of truth for meeting recordings + action items) — pulled by `~/scripts/fathom-catchup-poller.sh` into `~/.claude/fathom-digest/YYYY-MM-DD.md` and `.notion-manifest`. The `/brief-fathom` companion command surfaces missed meetings and creates Notion tasks for items owned by you. Read the digest files directly; do not call Fathom API from inside commands.
2. **Google Drive** -- search knowledge base for docs and shared meeting notes (not Fathom recordings).
3. **Google Calendar** -- `gcal_list_events` for meeting context (not action items directly).

Fireflies and Granola are no longer used (removed 2026-04-07). Do not query `fireflies_*` or Granola MCP tools even though they may still be installed.

## Routing

| Type | Destination | Examples |
|------|-------------|---------|
| Engineering / Product | Notion sprint board (Rule 12) | Feature requests, bugs, infra, UX |
| Client / Sales / Business | Command Center DB + ClickUp (see `reference-ids.md`) | Proposals, follow-ups, content, scheduling |
| Ambiguous | Flag for human decision | Unclear ownership |

## Before Creating Items

1. Search Notion sprint board for existing matching task
2. Search Command Center DB for matching open task
3. If found -> skip. If status changed -> update.

## Report Format

Add a "Meeting Intelligence" section to Notion Sync reports with tables for: New Items Routed, Already Tracked (skipped), Needs Human Decision.
