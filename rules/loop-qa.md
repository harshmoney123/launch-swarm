# QA -- "qa tests" or "verify prs"

`/loop 20m`. **Pipeline gate** -- no PR merges without QA Pass.

## Why this exists

Unit tests + a single pre-PR screenshot don't prove a feature works end-to-end. The failure mode you kept hitting: wake up, manually smoke-test everything, find bugs the swarm never noticed because nothing actually *used* the feature on the test env. QA closes that gap by running the PR's "How to test" plan against the real test env and verifying the **end-state behavior**, not just that the UI rendered.

## The Gate

Watchdog only auto-merges when BOTH exist:
1. Senior Reviewer: Approve
2. **QA: Pass**

QA replaces Docs as the second gate. Docs rule is kept dormant.

## Each tick

1. `gh pr list --state open --base YOUR_DEV_BRANCH` in both repos (customer portal + monorepo).
2. Filter to PRs with Senior Reviewer Approve and no QA verdict yet.
3. Skip backend-only PRs if no UI is touched -- drop a `QA: Pass -- backend-only, no UI surface` comment and move on. For backend PRs with API changes that affect visible behavior, run the end-to-end flow through the UI.
4. For each eligible PR:
   a. Sync test env to the PR branch. Rsync standalone repos to the parent monorepo first, then `cd YOUR_MONOREPO_PATH && ./test-env/test-env.sh sync`. Verify with `status`.
   b. Post `QA: Running smoke test against test env (branch: <name>)` comment so humans + Sprint Worker know the env is in use.
   c. Run the smoke-test protocol below.
   d. Post the QA report.
   e. Restore env to `YOUR_DEV_BRANCH` when done.
5. Worktree hygiene: remove any leftover QA worktree.
6. Report tick summary: PRs QA'd, pass/fail count, env state.

## Smoke-test protocol

Four phases per PR. Every phase uses Playwright MCP against the cloud test env, not local dev.

### Phase 1 -- Parse the spec + derive the test surface
Read the PR description. Extract:
- The "How to test" numbered steps and their **Expected:** assertions
- The "Edge cases to check" list
- The account the PR specifies (admin, basic, etc.)
- Any linked Notion task's spec

**Diff-aware fallback** (gstack pattern): compute the PR's changed files with `git diff YOUR_DEV_BRANCH...HEAD --name-only` and map each changed file to the routes/pages it affects:
- `src/pages/**/*.tsx` -> the route slug (e.g. `src/pages/crm/paying.tsx` -> `/crm/paying`)
- `src/components/**/*.tsx` -> grep usages, collect the routes that import the component
- API route / backend service file -> the frontend routes that call it (grep for the endpoint string)
- CSS / tokens -> include pages that import the stylesheet plus `/`, `/crm`, `/settings` as affected

If the PR's "How to test" covers every derived route, trust it. If the derived set includes routes the PR didn't mention, QA those too and note it under "Extra surfaces covered" in the report. If the PR has NO "How to test" at all, fall back entirely to the derived surface and test the top 5 routes by change density -- do NOT hard-block; only `QA Block` if the diff itself is unparseable.

### Phase 2 -- Happy path on the test env
1. Log in with the specified account. Default back-testing matrix (see memory `feedback_playwright_ux_verification.md`):
   - Admin features: `admin@example.com` / `adminpassword`
   - Tier-gated: `prouser@example.com` / `Test123!`
   - All-user: `basicuser@example.com` / `Test123!`

   **CRITICAL login pattern for Playwright MCP** (discovered during the 2026-04-15 login no-op investigation): `browser_click` on the "Sign in with Email" button does NOT propagate to the form's `onSubmit` handler -- the click fires but zero network requests go out. The app code is correct; this is a Playwright-MCP-specific interaction quirk with React synthetic events. Use ONE of these two patterns instead of `browser_click` on the submit button:

   Pattern A -- fill-and-Enter (preferred):
   ```
   browser_fill_form with email + password inputs
   browser_press_key "Enter"   <- submits the form natively, fires onSubmit correctly
   ```

   Pattern B -- programmatic submit (fallback if Pattern A flakes):
   ```
   browser_fill_form with email + password inputs
   browser_evaluate "document.querySelector('form').requestSubmit()"
   ```

   Pattern C -- direct API (last resort, for unblocking QA when both above fail):
   ```
   Use Bash `curl -X POST <test-env>/api/auth/login -d '{...}'` to get a token,
   then browser_evaluate to inject token + user into localStorage,
   then browser_navigate to the protected route.
   ```

   Do NOT click the submit button directly. Do NOT report a login-click "silent no-op" as an app bug -- it is the Playwright-MCP quirk, already diagnosed.

2. Execute each numbered step exactly. Wait for network settle between steps.

   **Form submits after login**: same rule -- prefer `browser_press_key "Enter"` inside a filled form over `browser_click` on a submit button. For non-form actions (menu items, links, toggles), `browser_click` is fine.

   **Typing into React inputs**: Playwright's `browser_type` sometimes fires input events without committing value to React state, which makes `required` validation block submit silently. If a submit appears to no-op, check the input's React value with `browser_evaluate "document.querySelector(selector).value"` before declaring a bug.
3. After each step, assert the **Expected:** outcome is visible. Screenshot desktop (1440x900) + mobile (375x667).
4. Capture console errors and network requests throughout.
5. **End-state verification**: after the last step, do the actions that prove the state actually persisted:
   - Reload the page -- is the change still there?
   - Log out + log back in -- did it survive?
   - For server writes: hit the API/GET endpoint directly and confirm the new data exists.
   - For Notion/Payload/DB writes: call the corresponding MCP tool or curl the admin endpoint and confirm the record.
   "The UI showed success" is not proof. The data must round-trip.

6. **DOM-diff verification** (gstack `snapshot -D` pattern): for each step that is supposed to change the page, call `browser_snapshot` (accessibility tree) immediately before the action, perform it, call `browser_snapshot` again, and diff the two. Assert the diff contains the expected node changes (new row appended, status text flipped, modal opened, etc.). Silent no-ops -- where the server accepted a write but the UI never updated -- are a common class of bug that screenshot comparison misses. If the diff is empty where it should not be, `QA Fail: silent no-op at step N`.

### Phase 3 -- Adversarial pass
Run these unless they are clearly N/A for the PR:

- **Authorization matrix** -- with the "should NOT see it" account (per memory), retry the happy path. Expected: 403 / upgrade prompt / feature hidden. If the wrong account can see the feature, that is a `QA Fail: auth`.
- **Console error budget = 0** -- any `console.error` during the flow is a `QA Fail: console error`. `console.warn` is a noted nit, not a fail.
- **Mobile viewport** -- repeat the happy path at 375x667. Check for overflow, unreachable buttons, sticky-header collisions.
- **Error states** -- empty input, too-long input (1000 chars), special chars (`'"<>&`), emoji, rapid double-click on submit. If the UI crashes or swallows errors silently, `QA Fail: error handling`.
- **Regression spot-check** -- visit three canonical pages unrelated to the PR: `/`, `/crm`, `/settings`. Any new console errors or layout breakage vs `YOUR_DEV_BRANCH` baseline is a `QA Fail: regression`.

### Phase 4 -- Cross-check the claims
- The PR's "What was NOT tested" section -- do any of those omissions cover real-user paths? If yes, note as a risk.
- The unit test count claimed in the PR -- does it actually cover the paths QA just exercised? Discrepancy is a nit, not a fail.

## Verdict + report format

Post one comment on the PR. Use exactly one of these statuses:

```
QA: Pass
```
or
```
QA: Fail -- <one-line reason>
```
or
```
QA: Block -- <what prevents testing>
```

Then include this report body:

```markdown
## QA Report -- <date>

**Test env**: branch `<branch>` synced at <time>
**Accounts used**: <list>

### Happy Path
- Step 1 (<summary>) -- ✅ Expected `<X>`, got `<X>`. [screenshot-desktop] [screenshot-mobile]
- Step 2 ...
- **End-state verification**: <what you confirmed and how>

### Adversarial
- Authorization: ✅ basic user gets 403 / ❌ basic user saw admin button
- Console errors: 0 / N
- Mobile 375x667: clean / <issue>
- Error states: <results>
- Regression spot-check: clean / <issue>

### Failures (if any)
1. **<one-line title>**
   - Reproduction: <steps>
   - Expected: <X>
   - Actual: <Y>
   - File/component likely at fault: <path:line>
   - Screenshot: <s3 url>
   - Console excerpt: <error>

### Not covered
- <anything QA could not test and why>
```

On `QA: Fail`, also post a GitHub review with `REQUEST_CHANGES` event so Watchdog's gate check is firm.

## Metric log -- auto-pass rate

After posting each verdict, append one line to `~/.swarm/qa-log.md`:

```
<ISO timestamp> | PR #<num> | <repo> | <verdict> | <SP> | <title>
```

Verdict is one of `PASS`, `FAIL`, `BLOCK`. your morning brief reads this file to compute the **auto-pass rate** (PASS / (PASS + FAIL + BLOCK)) per sprint. The metric tracks whether the swarm is shipping code you can approve vs code he has to micro-fix. Target: >= 70% PASS after 2 sprints, >= 85% by sprint 4.

**v2 planned** (not implemented yet): extend the log line with a 0-100 health score using the gstack rubric (Console 15%, Functional 20%, UX 15%, A11y 15%, Perf 10%, Visual 10%, Links 10%, Content 5%) + write a `baseline.json` per PR at pass time so regression mode can diff fixed / new / delta. v1 stays binary PASS/FAIL/BLOCK; upgrade once the morning brief is consuming the simple ratio reliably.

## Screenshots

Upload to S3 via `scripts/upload-pr-screenshots.sh` under `pr-<num>/qa-<timestamp>/`. Embed the URLs inline in the report. No red-arrow annotations needed -- QA is evidentiary, not marketing.

## Coordination with Sprint Worker

**QA is the exclusive Playwright owner in the swarm.** Sprint Worker does NOT open the MCP browser (as of 2026-04-15 -- see `feedback_playwright_is_qa_only.md`). This removes the race condition where concurrent ticks tried to grab the single shared chrome instance.

- QA freely syncs the test env to the PR branch, runs the full 4-phase protocol, then restores the env to `YOUR_DEV_BRANCH` before finishing.
- QA posts a `QA: Running` comment on start and a verdict comment on finish. Purely informational now -- no loop defers based on it.
- If QA ever finds that Sprint Worker HAS opened the browser (the `"Browser is already in use for mcp-chrome-..."` error), flag it in the tick report so the rule drift can be corrected. Do NOT retry silently -- it means a loop rule has regressed.
- QA is responsible for restoring the env to `YOUR_DEV_BRANCH` when done. Always.

## What NOT to do

- Do NOT run against a local dev server -- the whole point is the real test env with real auth, real AWS creds, real data.
- Do NOT skip the end-state verification even if the UI looked fine. "It showed a toast" is not proof.
- Do NOT fix the bug yourself -- file the report and let the Sprint Worker / Watchdog fix it.
- Do NOT auto-close as Pass if the PR description is missing "How to test". Block and ask for one.
- Do NOT touch prod DynamoDB tables. Only `Dev-*`.
- Do NOT QA PRs authored by the CTO or other non-swarm authors. Out of scope.
- Do NOT use `Bash(run_in_background=true)` for test-env sync, rsync, or npm install. Run synchronously with `timeout 300` prefix. Background polling loops (`until ... do sleep; done`) become zombies when the agent exits — never use them.
