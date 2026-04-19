# Verifier -- "verify prs" or "run verifier"

`/loop 15m`. **Pipeline gate** -- no PR merges without Verifier PASS.

## Why this exists

Unit tests pass on the Sprint Worker's machine but can break in a fresh checkout because of uncommitted node_modules, stale lockfiles, or missing env vars. Senior Reviewer reads code but never runs it. QA runs a live browser probe on the test env but doesn't catch build failures, new lint errors, or dangerous static patterns (auth middleware missing, `Dev-` prefix missing on DynamoDB calls, `dangerouslySetInnerHTML`). Verifier fills the gap: fresh worktree, deterministic commands, static adversarial probe on the diff, PASS / PARTIAL / FAIL verdict before Watchdog auto-merges.

Verifier does NOT open the Playwright MCP browser. That surface is reserved for QA. Verifier is pure-static + deterministic build/test/lint.

## The Gate

Watchdog only auto-merges when ALL three exist:
1. Senior Reviewer: Approve
2. QA: Pass
3. **Verifier: PASS** (or PARTIAL, unless the PR touches auth/Emma core)

Missing verdict -> Watchdog skips. Verifier tick will pick it up.

## Each tick

1. `gh pr list --state open --base YOUR_DEV_BRANCH` across repos (customer portal + backend monorepo).
2. Filter to PRs with Senior Reviewer Approve AND no Verifier verdict yet.
3. Skip PRs Verifier has already posted a verdict on -- grep PR comments for `Verification: PASS|PARTIAL|FAIL`.
4. Process at most **2 PRs per tick** to avoid queue starvation on the sprint worker.
5. For each eligible PR:
   a. Create ephemeral worktree at `/tmp/verifier-<pr>-<ts>` -- `git worktree add /tmp/verifier-<pr>-<ts> <branch>`.
   b. Post `Verification: Running (branch: <name>)` comment so Watchdog + Sprint Worker know the verdict is coming.
   c. Run the verification protocol (below).
   d. Post the verdict report.
   e. Remove the worktree: `git worktree remove /tmp/verifier-<pr>-<ts> --force`.
6. Worktree hygiene: `git worktree prune` at tick end.
7. Append one JSONL line per verdict to `~/.agentweb-swarm/swarm-log.jsonl`.
8. Append one line per verdict to `~/.agentweb-swarm/verifier-log.md`.
9. Report tick summary: PRs verified, pass/partial/fail count.

## Verification protocol

Four phases. All local, no test env, no browser.

### Phase 1 -- Fresh install

1. `npm ci` (or `bun install --frozen-lockfile`). Timeout 300s.
2. If install fails -> `FAIL: install` -- the lockfile is broken or a dependency is unavailable. Record the error tail. No further phases run.

### Phase 2 -- Build, test, lint, typecheck (parallel)

Run the following commands in parallel via `&` and `wait`. Each has a 300s timeout.

- `npm run build` -- exit != 0 -> `build=fail`
- `npm run test:run` (or the repo's equivalent one-shot test command) -- exit != 0 -> `tests=fail`
- `npm run lint` -- exit != 0 -> `lint=fail`, BUT see "Lint baseline" below
- `npm run typecheck` if the script exists -- exit != 0 -> `typecheck=fail`

**Lint baseline**: Pre-existing lint errors are not the PR's fault. Before counting lint as failing, diff against the base branch:

```bash
git diff --name-only YOUR_DEV_BRANCH...HEAD | xargs -r npm run lint --
```

Only files touched by the PR count. If the lint step has no file-scoped mode, run `npm run lint` on both the PR branch and the base branch, diff the error count, and only fail if the PR branch has MORE errors than the base.

**Coverage**: Do NOT gate on coverage delta in Verifier. Coverage is enforced at the project level in the test script itself. If `npm run test:run` passes, coverage is fine.

### Phase 3 -- Dependency risk

1. `npm audit --production --audit-level=high --json`. Parse output.
2. Diff against the base branch's audit output. Only NEW high/critical advisories in the PR branch count.
3. New high/critical -> `deps=fail` unless the PR's title or description already calls out the dep bump and justifies it (grep for "CVE" or "security" in PR body).

### Phase 4 -- Static adversarial probe on the diff

Run `git diff YOUR_DEV_BRANCH...HEAD` and grep the additions (not removals) for dangerous patterns. Each match is one finding.

**Hard-fail patterns** (one match -> `FAIL`):
- Any new `dangerouslySetInnerHTML` without an adjacent sanitizer call (DOMPurify, sanitize-html).
- Any new call to `eval(` or `new Function(` in application code.
- A new Express route handler (`router.(get|post|put|delete|patch)(`) in the backend repo without `authenticateToken` imported in the same file. Exception: files under `routes/public/` or matching `public.ts` suffix.
- A new DynamoDB call (`dynamoDb.(get|put|update|delete|scan|query)(`, `docClient.send(`, `TableName:`) where the table name literal is hardcoded without a `Dev-` prefix in dev. Check via `grep "TableName:\s*['\"]" | grep -v "Dev-"` on added lines only. Rationale: memory says never touch prod tables.
- Any new hardcoded AWS access key, secret key, API key, or JWT secret (regex: `AKIA[0-9A-Z]{16}`, `aws_secret_access_key\s*=`, `JWT_SECRET\s*=\s*['"]`).

**Amber patterns** (one match -> `PARTIAL`, flag on PR, do not block):
- Diff touches `src/contexts/AuthContext.tsx`, any file under `auth/`, `tokens`, `sessions`, `reset-password`, or `/api/auth/*` -> PARTIAL with note "auth surface, needs Rui review per ownership memory".
- Diff touches Emma core, MCP server, or agent orchestration files (`emma/`, `mcp/`, `agent-engine/`) -> PARTIAL with note "agent infra, needs Rui review per ownership memory".
- New `any` type in TypeScript where context is non-trivial (> 5 occurrences in one file) -> PARTIAL with note "type safety regression".
- Diff adds > 500 lines on net -> PARTIAL with note "large diff, break into sub-tasks per Sprint Worker rules".

## Verdict + report format

Post one comment on the PR. Use exactly one of these statuses:

```
Verification: PASS
```
or
```
Verification: PARTIAL -- <one-line reason>
```
or
```
Verification: FAIL -- <one-line reason>
```

Then include this report body:

```markdown
## Verifier Report -- <date>

**Branch**: `<branch>` at commit `<sha>`
**Base**: `YOUR_DEV_BRANCH` at commit `<sha>`

### Phase 1 -- Fresh install
- npm ci: pass | fail (<tail of error>)

### Phase 2 -- Build / test / lint / typecheck
- build: pass | fail
- tests: pass | fail (X passed, Y failed)
- lint: pass | fail (diff: +N errors over base)
- typecheck: pass | fail | N/A

### Phase 3 -- Dependency risk
- npm audit: no new high/critical | <N new advisories, list top 3>

### Phase 4 -- Static adversarial probe
- Hard-fail patterns: none | <list>
- Amber patterns: none | <list with file:line>

### Verdict
- <one-line summary tying the verdict to the phases>
```

On `Verification: FAIL`, post a GitHub review with `REQUEST_CHANGES` event so Watchdog's gate check is firm.
On `Verification: PARTIAL` with auth/agent-infra amber, comment but do NOT request changes -- Watchdog will still skip auto-merge for auth-touching PARTIALs per the ownership rule.
On `Verification: PASS`, post the comment without a review event.

## Metric log

After posting each verdict, append one line to `~/.agentweb-swarm/verifier-log.md`:

```
<ISO timestamp> | PR #<num> | <repo> | <verdict> | <phase1> | <phase2> | <phase3> | <phase4> | <title>
```

And append one JSONL line to `~/.agentweb-swarm/swarm-log.jsonl`:

```json
{"ts":"<ISO UTC>","loop":"verifier","tick":"complete","action":"verdict","pr":<num>,"verdict":"pass|partial|fail","duration_s":<num>}
```

## Coordination with other loops

- **Sprint Worker**: Verifier does not block the Sprint Worker. Sprint Worker opens PRs normally; Verifier picks them up after Senior Reviewer approves.
- **Senior Reviewer**: Verifier does not change Reviewer's workflow. Reviewer reads code, approves on judgment. Verifier runs after.
- **QA**: Both run after Senior Reviewer approves. They run in parallel -- Verifier is fast (no test env sync), QA is slow (Playwright probe). Watchdog waits for both verdicts before auto-merging.
- **Watchdog**: Never auto-merges without PASS (or acceptable PARTIAL). Adds Verifier to its gate check per `loop-pr-ops.md`.
- **Playwright ownership**: Verifier NEVER opens the MCP browser. That is QA's exclusive surface per memory `feedback_playwright_is_qa_only.md`.

## Failure modes (fail-open, not fail-closed)

If Verifier itself breaks -- worktree collision, npm registry unreachable, node version mismatch in the runner -- emit `Verification: BLOCK -- <reason>` instead of FAIL. BLOCK is treated by Watchdog the same as "verdict missing": skip auto-merge, retry next tick. Do not REQUEST_CHANGES on BLOCK -- that punishes the PR for swarm infra problems.

If the same BLOCK reason hits 3+ ticks in a row on different PRs, that is a systemic issue -- escalate to Harsha via Slack (`YOUR_DIGEST_CHANNEL`) and stop firing Verifier until the infra issue is fixed.

## Dogfood plan (before gating Watchdog on Verifier)

Do NOT flip the Watchdog gate to require Verifier PASS until Verifier has been dogfooded:

1. Ship this file (`loop-verifier.md`) and add the Verifier row to `loop-modes.md` agents table.
2. Manually invoke `/loop 15m verify prs` from one terminal for 3 ticks.
3. Compare verdicts against the actual ship outcome of those PRs.
4. If >= 2 of 3 verdicts match reality (PR that shipped clean got PASS, PR that needed a revert got FAIL), proceed to step 5.
5. Ship the `loop-pr-ops.md` gate change in a follow-up PR: add `Verifier PASS` to the Watchdog auto-merge condition.
6. If verdicts are noisy (>= 2 in 3 false FAILs), stop and tune the Phase 4 static patterns before gating.

## What NOT to do

- Do NOT open the Playwright MCP browser. Ever. Violates the QA-exclusive-owner rule.
- Do NOT touch the test env. Verifier is local-worktree-only.
- Do NOT run against prod DynamoDB tables. Only `Dev-*`. Verifier never writes to any DB.
- Do NOT fix the PR. File the verdict and let Sprint Worker / Watchdog fix it.
- Do NOT lower coverage thresholds to make the build pass. Write more tests per memory `feedback_never_lower_coverage.md`.
- Do NOT gate on coverage delta here -- the project-level test script already enforces coverage.
- Do NOT emit PASS on install failure. A broken install is a real PR problem (usually a bad lockfile commit), not a swarm infra issue.
- Do NOT treat pre-existing lint errors as a failure. Diff against base first.
- Do NOT use `Bash(run_in_background=true)` for the verification commands. Run synchronously with `timeout 300` prefix. Background polling loops become zombies when the agent exits.

## Acceptance test for this rule

Before declaring Verifier ready to gate:

1. Pick 3 recent merged PRs from `YOUR_DEV_BRANCH`.
2. Run the verifier protocol manually against each (checkout, install, build, test, lint, audit, grep diff).
3. Record the verdict.
4. Compare to the actual ship outcome (did the PR stick, or was it reverted / hotfixed within a day?).
5. At least 2 of 3 verdicts should match reality. If not, tune the Phase 4 patterns.

Document the acceptance test result in the Notion task comment before flipping the Watchdog gate.
