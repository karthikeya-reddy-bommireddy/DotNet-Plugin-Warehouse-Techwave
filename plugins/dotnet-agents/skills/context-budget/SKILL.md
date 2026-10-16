---
name: context-budget
description: Use when get_context_budget or the host warns that tokens or time are running low, or at a gate boundary, to keep the orchestrator context small and to compact only at safe points. Do not use mid-step, and never compact an active run without a recorded handoff.
license: Proprietary
metadata:
  version: 1.1.0
  agent: dotnet-orchestrator
  contract: context-budget/1
disable-model-invocation: true
---
# Context budget

## Keep context small every turn
- Read reports and IDs, not files or logs.
- Ask the engine for summaries (`get_verification_summary`) instead of reading raw results.
- Do not repeat earlier messages; refer to IDs.
- Do not load skills the engine did not name.

## When the budget is high
The run's budget is minutes of specialist work against `budget.max_minutes` in `.agents/config.json`.
`get_context_budget(run_id)` gives `minutes_used`, `minutes_limit`, `percent_used` and `level`;
`get_run_state.budget_remaining` is the percentage left. Tokens are not counted yet.

| Signal | Action |
|---|---|
| `level` notice (70%) | Mention it in the status report |
| `level` ask (90%), or `next_step.reason` starts with "Budget:" | Ask the developer to continue, pause or narrow the scope before you delegate |
| `level` exhausted: the run is Paused (`limit_reached`) | Tell the developer: only they raise `budget.max_minutes`, then `agents resume <run>`; never ask for a reset |
| Host context above about 80% | Compact at the next safe point |

## Safe points to compact
Only at: after a gate decision is recorded, between modules, or when the run is paused.
Never in the middle of the build loop.

## Before compacting
Load `resume-and-handoff` and record a handoff. After compaction, resume from `get_resume_brief`,
not from memory.
