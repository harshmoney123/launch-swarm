# Log Watcher -- "watch logs" or "test buddy"

`/loop 1m`. Monitors the test environment and gives live feedback during manual testing.

## Prerequisites

- Test env must be running: `cd YOUR_PROJECT_ROOT && ./test-env/test-env.sh status`
- If not running, report and stop. Do NOT launch it automatically.

## Each tick

1. **Grab logs since last tick** (last 200 lines per service):
   - Docker mode: SSH into test env then `cd /home/ec2-user/YOUR_PROJECT/test-env && docker compose logs --tail=200 --no-color`
   - Dev mode: SSH into test env then `tail -200 /home/ec2-user/YOUR_PROJECT/logs/backend.log /home/ec2-user/YOUR_PROJECT/logs/customer-portal.log`
   - Use Bash with the SSH command directly: extract IP from `.test-env/*.env` state file, then `ssh -o StrictHostKeyChecking=no -i <key> ec2-user@<ip> '<command>'`

2. **Filter for actionable signals** -- skip noise, focus on:
   - `ERROR`, `Error`, `error`, unhandled exceptions, stack traces
   - `WARN` that indicate real issues (not deprecation spam)
   - HTTP 500/502/503 responses
   - `ECONNREFUSED`, `ENOTFOUND`, DNS failures
   - Database errors, AWS SDK errors
   - React/Vite build errors, chunk load failures
   - Auth failures, token errors
   - Process crashes, restarts, OOM kills

3. **Skip noise** -- ignore these:
   - Deprecation warnings
   - Hot-reload / HMR messages
   - Routine health checks
   - Static asset 304s
   - Debug-level logging

4. **Report** -- if errors found:
   - Print a clear summary: timestamp, service, error, stack trace snippet
   - Correlate with recent `YOUR_DEV_BRANCH` commits if possible (`git log YOUR_DEV_BRANCH -5 --oneline`)
   - Suggest which file/PR likely caused it
   - If it's a crash loop (same error 3+ ticks in a row), flag as CRITICAL

5. **Silent tick** -- if logs are clean, say nothing. Only report when there's something actionable.

## State tracking

Keep a mental note of:
- Errors seen in previous ticks (to detect repeats vs new issues)
- Which services are healthy vs unhealthy
- If a service stops producing logs (possible crash)

## What NOT to do

- Do NOT fix code -- just report. The developer is manually testing.
- Do NOT create Notion tasks -- this is live feedback, not bug filing.
- Do NOT restart services -- just flag if something is down.
- Do NOT run Playwright -- this watches server-side only. Use QA Personas for UI testing.
