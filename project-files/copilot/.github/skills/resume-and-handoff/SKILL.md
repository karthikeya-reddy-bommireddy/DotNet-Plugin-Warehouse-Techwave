---
name: resume-and-handoff
description: Use when a session starts with an active run, before context compaction, when a run pauses, or when the developer says continue. Writes the structured handoff with record_handoff or reads it with get_resume_brief. Do not use to change run state, re-plan, or rebuild the story from chat memory.
license: Proprietary
metadata:
  version: 1.2.0
  agent: dotnet-orchestrator
  contract: handoff/1
disable-model-invocation: true
---
# Resume and handoff

## Write a handoff (before compaction, pause or session end)
1. `get_run_state` and `next_step`: they hold every fact the handoff needs.
2. `record_handoff` with handoff/1, every field taken from them, never from memory:
   - `state`, `phase`, `module` and `last_checkpoint` from `get_run_state`;
   - `pending_decisions`, `pending_approvals` and `open_questions` from `get_run_state`;
   - `last_reports`: `recent_reports` from `get_run_state`;
   - `next_step`: `step_id` from `next_step` (its `action` when there is no step).
3. If the engine refuses it (`handoff_mismatch`), it lists each difference: call `get_run_state` and
   `next_step` again and write the handoff from them. Never argue with the engine's record.
4. Tell the developer in one line that the state is saved. When the AI tool's Stop hook asks for a
   handoff, record it, then stop.

## Resume (session start or "continue")
1. `get_resume_brief` (with `run_id` when you know it). When several runs are active it lists them and
   picks none: ask the developer which one, then call it again with that `run_id`.
2. If the state is Out of date, call `check_staleness` and explain what changed, what it reaches and where the re-run
   starts; give the developer `agents rerun <run>` (only they re-run).
3. Present: where we are, what waits on the developer (with the commands), the task in flight, the next
   allowed step. Use only what the brief says.
4. Ask before continuing. Then call `resume_run` if the run is paused for a reason the developer has
   resolved (requested, ai_outage, interrupted), and `next_step`.

## Interruptions
- The AI service is unavailable (a sub-agent call fails: overloaded, server error, rate limit): call
  `pause_run(cause: "ai_outage")`. The run is Paused, never Failed. The AI tool's hooks may have paused it
  already; then there is nothing to do but tell the developer.
- A run paused while a task ran (an outage, the end of a session, a requested pause) restarts that task
  when it resumes: the engine puts the code back at the task's start and dispatches the same task again
  with a new dispatch ID. `next_step` names it; invoke the specialist with that ID.
- If the brief shows a task running that you did not start in this session (the last session ended
  without its hooks), call `pause_run`, tell the developer, and resume as above.
- If resuming is refused because files changed that the engine has no record of, show the developer the
  list. They decide whether to put the files back; never do it yourself.

## Rules
- The engine record wins when it disagrees with chat memory or a summary.
- Never resume a Cancelled run; offer a new run instead.
- Rolling back is the developer's: `rollback_run` returns their command (`agents rollback <run> --to
  <checkpoint>`); show it and wait.
