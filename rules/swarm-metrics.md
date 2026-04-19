# Swarm Metrics -- "swarm metrics" / "swarm health"

On-demand diagnosis of the swarm's throughput, quality, and loop health. Not a loop -- invoked by you when he wants to know how the swarm is doing.

## Trigger

- Keyword: `swarm metrics`, `swarm health`, or `/swarm-metrics`
- Morning brief also inlines a compact version.

## Invocation

Run the compute script:

```bash
python3 ~/.swarm/scripts/swarm-metrics.py --days 7
```

Optional flags:
- `--days N` -- window size (default 7)
- `--json` -- raw JSON (useful when piping into other tools)

Print the rendered output verbatim. Then, if any diagnosis line starts with ⚠️, propose ONE concrete next action per finding. Keep action suggestions under 15 words each. Do not propose more than 3 -- if the script emits more, focus on the top 3 most severe.

## Data sources

- `~/.swarm/swarm-log.jsonl` -- one line per loop tick. Each loop appends on every fire (see "Log contract" below).
- `~/.swarm/qa-log.md` -- QA verdict log (written by `loop-qa.md`).
- `gh pr list` -- real-time PR state (used for cross-checking, not stored).
- Notion Master Table view -- current To-do/In-Progress counts.

The script reads only the first two. Real-time `gh`/Notion data is consulted manually by the agent only when a diagnosis calls for it (e.g., "why is Watchdog idle?" -> check open PRs).

## Log contract -- every loop MUST append one JSONL line per tick

Each loop, at the end of its tick, appends a single line to `~/.swarm/swarm-log.jsonl`:

```json
{"ts":"<ISO UTC>","loop":"<name>","tick":"complete|idle|blocked","action":"<verb>","pr":<num?>,"task_url":"<notion?>","duration_s":<num>,"blocker":"<reason?>"}
```

Required fields: `ts`, `loop`, `tick`. The rest are optional depending on what happened.

Canonical `loop` names (use exactly these so metrics group cleanly):
- `sprint-worker`
- `watchdog`
- `senior-reviewer`
- `planner`
- `qa`
- `auditor`

Canonical `action` verbs:
- `opened_pr` (sprint-worker)
- `merged_pr` (watchdog)
- `reverted_pr` (watchdog)
- `pushed_fix` (watchdog fixing review comments or CI)
- `synced_dev` (watchdog merged main into YOUR_DEV_BRANCH)
- `verdict` (qa -- must also set `verdict: pass|fail|block`)
- `approved` / `requested_changes` / `commented` (senior-reviewer)
- `planned_task` (planner)
- `audited_repos` (auditor)
- `idle` or omit -- nothing actionable this tick

## Diagnosis rules (encoded in the script)

The script emits warnings when:
- Any loop with >=4 ticks is idle >60% -> likely queue starving
- QA auto-pass < 60% over >=3 verdicts -> quality regression
- Sprint Worker fires 3+ ticks without opening a PR -> sprint empty or all blocked
- Watchdog fires 3+ ticks, PRs exist, but merged zero -> a gate is stuck
- Any loop hits the same blocker 3+ times -> systemic issue

Add a new rule by editing `diagnose()` in the script. Keep rules threshold-based; no ML.

## What NOT to do

- Do not propose fixes that require running the full swarm ("relaunch swarm"). Metrics is for diagnosis; swarm launch is a separate decision.
- Do not rewrite `swarm-log.jsonl` -- only append.
- Do not hide findings. If the script flags a warning, surface it even if it contradicts a prior optimistic take.
