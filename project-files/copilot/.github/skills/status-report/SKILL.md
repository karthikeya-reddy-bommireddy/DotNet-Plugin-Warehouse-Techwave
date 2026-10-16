---
name: status-report
description: Use when the developer asks for status, at the end of a turn that changed the run, or when a run pauses. Produces a short plain-language status from get_run_state. Do not use to make decisions, re-plan or summarise from chat memory when the engine record differs.
license: Proprietary
metadata:
  version: 1.1.0
  agent: dotnet-orchestrator
  contract: status/1
---
# Status report

## Procedure
1. Call `get_run_state(run_id)` and `next_step(run_id)`; if useful, `get_context_budget(run_id)`.
2. Write at most 5 bullets in this order:
   - **Where we are:** module, `phase`, `state` (Running, Paused, Failed, Out of date, Completed) and,
     when paused, why (`pause_reason`).
   - **What happened:** the last report (`last_report`: task, agent, accepted or not, status), one line.
   - **What I need from you:** each ID in `pending_decisions` and `open_questions` with the command that
     answers it (`agents decide ...`, `agents answer ...`). Write "Nothing right now" when empty.
   - **What happens next:** `next_step.action` and its reason, in plain words.
   - **Budget:** tokens and minutes used against the limit, only when above 70%.
3. Link evidence (paths, IDs). Never paste logs. The developer can see the same record with
   `agents status <run>`.

## Example
- Where we are: module Orders, adaptation (Paused: waiting for a decision).
- What happened: dotnet-adapter's report T-0003 was accepted (needs_decision).
- What I need from you: DEC-001, the integration-test database: `agents decide R-0001 DEC-001 <option-id>`.
- What happens next: adaptation continues with your decision.

## Rules
The engine record wins over chat memory. Label uncertainty. Never say "done" unless the engine
state is Completed.
