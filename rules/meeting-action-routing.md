# Rule 15: Meeting Action Item Routing

Pull action items from meeting sources and route to the correct system. Runs every 3rd Notion Sync tick.

## Sources

1. **Google Drive / meeting recordings** -- search knowledge base for transcripts
2. **Granola** -- `list_meetings` -> `get_meetings` for summaries
3. **Fireflies** -- `fireflies_get_transcripts` -> extract `action_items`
4. **Google Calendar** -- `gcal_list_events` for meeting context (not action items directly)

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
