# PR Formats & Back-Testing Reference

## Reviewer-First PR Format (all task PRs)

```markdown
## Risk: Low / Medium / High | Priority: High / Medium / Low

## What was wrong (before)
[1-2 sentences]

## What this fixes (after)
[1-2 sentences]

## How to test
1. Go to [specific route/page]
2. [Specific action -- click, type, navigate]
3. **Expected**: [What you should see]
4. [Next step]
5. **Expected**: [What you should see]

[Use specific routes like /agent, /settings, /admin/users]
[For admin features: "Log in as admin (admin@example.com)"]
[For UI changes: describe the exact visual change]
[For behavioral changes: describe before vs after]

## Edge cases to check
- [One thing automated tests didn't cover]

## Test Evidence
- Unit tests: X passed, 0 failed
- Tested as: [accounts]

## Files changed
[List, max 10. Group by area if more.]

## What was NOT tested
[Honest list]

## Related PRs
- Backend: [#NNN or "standalone"]
- Customer Portal: [#NNN or "standalone"]

## Notion task
[Link]
```

**Critical PRs add:** Backend curl proof, negative testing, rollback plan, dependency map.

## Mega PR Format (`YOUR_DEV_BRANCH` -> `main`)

```markdown
## Mega PR: YOUR_DEV_BRANCH -> main

## Summary
[X tasks -- Y features, Z fixes. All passed both gates.]

## Sub-PRs (sorted by risk, then priority)
| # | Risk | Priority | Type | Title | PR | Reviewer | Docs | Notion |
|---|------|----------|------|-------|----|----------|------|--------|
| 1 | High | High | feat | ... | [#N](url) | Pass | Pass | [link] |

**Each sub-PR contains**: screenshots, test evidence, resolved comments.

## High-Risk Items -- CTO: start here
- **#N -- [title]**: [why risky]. Rollback: [how].

## Test Status
- Full suite on `YOUR_DEV_BRANCH`: X passed
- All sub-PRs: both gates passed

## What was NOT tested
[Aggregate across all sub-PRs]

## Rollback
- Single task: `git revert <merge-commit>` on `YOUR_DEV_BRANCH`
- Everything: `git revert -m 1 <mega-merge>` on `main`
```

## Multi-Account Test Matrix

| Feature type | Test WITH | Test WITHOUT |
|-------------|-----------|-------------|
| Admin-only | `admin@example.com` / `adminpassword` | `basicuser@example.com` / `userpassword` -> verify 403 |
| Tier-gated | `prouser@example.com` / `userpassword` | `basicuser@example.com` -> verify upgrade prompt |
| All-user | `basicuser@example.com` / `userpassword` | N/A |
