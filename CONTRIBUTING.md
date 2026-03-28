# Contributing to launch-swarm

This is a living system. I update it as I find better patterns, and contributions from the community make it better for everyone.

## Ways to contribute

**Bug fixes.** If a rule has a logic error or contradicts another rule, open a PR with the fix.

**New agents.** Built a useful agent? Add it to `rules/` following the agent template:

```markdown
# Agent Name - "trigger keyword"

`/loop INTERVAL`. One sentence describing the role.

## Each tick

1. What to check/query
2. What to do with the results
3. What to report/update

## What NOT to do

- Explicit boundaries
```

Then add it to the agents table in `rules/loop-modes.md`.

**Tool adapters.** Made this work with Linear, Jira, or GitHub Projects instead of Notion? Share the modified rule files.

**Concept improvements.** Better gate systems, new health check patterns, improved priority algorithms.

## Guidelines

- Keep rules general. No project-specific configurations.
- Don't add emojis or formatting flourishes to rule files. Claude reads these, not humans.
- Don't combine agents that should stay separate. Separation of concerns is core to the design.
- Test your changes. If you modify a rule, explain what behavior changes and why.
- Keep PRs focused. One concept per PR.

## Reporting issues

Open an issue if:
- A rule doesn't work as described in the README
- Two rules contradict each other
- You found a better pattern for something the system already does

Include what you tried, what happened, and what you expected.
