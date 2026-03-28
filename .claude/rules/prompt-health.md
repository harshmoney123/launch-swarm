# Prompt Health -- Self-Learning & Anti-Rot

Rules should evolve. This system prevents prompt rot and keeps instructions lean.

## Principles

1. **One concept, one location.** Other files reference, never restate.
2. **If a rule needs >30 lines, it's probably encoding process that should be a checklist.**
3. **Hardcoded IDs go in `reference-ids.md`, not scattered across rules.**
4. **If a rule hasn't been triggered in 2+ weeks, it's a candidate for removal.**

## Health Tags

When reviewing rules, mentally tag each as:
- **ACTIVE**: Used regularly, still relevant
- **DORMANT**: Not triggered recently -- candidate for removal or simplification
- **STALE**: References outdated tools/schemas/IDs -- needs update or deletion

## Review Triggers

- **Correction from user**: Update the rule that was wrong. If no rule exists, add one to `lessons.md`.
- **Rule confusion**: If two rules contradict, resolve immediately. Keep the newer/more specific one.
- **Loop failure**: If a loop mode fails due to wrong instructions, fix the rule before re-running.
- **Monthly review**: User reviews DORMANT rules and decides keep/kill.

## Self-Learning Loop

After any mistake or correction:
1. Check if an existing rule should have prevented it -> fix the rule
2. If no rule exists -> add a concise one to Command Center `tasks/lessons.md`
3. If a rule caused the mistake (over-specified, wrong) -> simplify or delete it
4. Never add a rule that duplicates an existing one -- find and update instead

## Anti-Patterns to Watch For

- Same instruction appearing in 2+ files (merge them)
- Rules with specific IDs that could go stale (move to `reference-ids.md`)
- Multi-page loop mode descriptions (compress to reference core rules)
- "MUST" / "NEVER" / "CRITICAL" inflation -- if everything is critical, nothing is
- Rules written for a one-time incident that won't recur (delete after resolving)
