---
name: delegation-brief
description: Use when next_step returns action delegate for a specialist (dotnet-adapter, dotnet-designer, dotnet-coder, dotnet-reviewer, dotnet-tester, dotnet-fixer, dotnet-security, dotnet-docs). Builds brief/1 from the engine's context pack, starts the task with start_task and invokes the specialist. Do not use to decide what runs next, to retry on your own initiative, or to do the specialist's work.
license: Proprietary
metadata:
  version: 1.3.0
  agent: dotnet-orchestrator
  contract: brief/1
disable-model-invocation: true
---
# Delegation brief

The specialist starts with an EMPTY context. It cannot see this conversation.
Everything it needs must be in the brief or reachable through the brief's links.

## Steps
1. Call `build_context_pack(run_id, step_id)` with the step from `next_step`. Never guess paths.
2. Start from `brief_template`. Keep the fields the engine filled in exactly as they are:
   `schema`, `task_id`, `run_id`, `step_id`, `attempt`, `agent`, `skills`, `decisions`, `constraints`,
   `output` and `untrusted_data_note`. The engine rejects a brief that changes them.
3. Add `objective`: ONE outcome sentence a tester could check (10 to 300 characters).
4. Add `inputs`: refs from the pack's `required` and `optional` lists, and BA, decision or question IDs,
   each with a one-line `why` (at most 160 characters). Never paste file contents.
5. Add `done_criteria`: copied from the acceptance criteria and the step. Invent none.
6. Add `boundaries`: what not to touch, and "return needs_decision, needs_input or blocked instead of
   guessing". Name every entry of the pack's `unavailable` list the specialist must not assume. Add each
   of the pack's `lessons` that fits this task as a one-line note (approved by the developer).
7. For a re-brief after a failure, add `previous_attempts`: one line per earlier attempt with its task ID
   and failure IDs (at most 200 characters each). No earlier reasoning, no logs.
8. Call `start_task(run_id, brief)`. If it fails with `invalid_brief`, fix exactly the listed `details`
   and call it once more; if it fails again, tell the developer.
9. Invoke the specialist named in the result (`agent`) with the brief as its prompt and the result's
   `dispatch_id`: tell it "Your task is <task_id>, dispatch <dispatch_id>." It ends by handing its report to
   the engine with both IDs. When `next_step` says the task is already running, invoke
   it again with the dispatch ID `next_step` names, never an older one.
10. Call `next_step`, then load `report-synthesis`.

## Mandatory facts (from the context pack)
| Fact | Source |
|---|---|
| Recorded developer decisions | `decisions` (already in the template) |
| Answers to earlier questions | `answers`: cite their IDs in `inputs` |
| BA package and requirement registry | `required` |
| Technology decisions | tech profile ref in `constraints`; if `unavailable` names it, every technology choice is a decision |
| Where the specialist may write | `scope` (already in `constraints`) |
| Attempt number | `attempt` (already in the template) |

## Never
- Include secrets, credentials, connection strings or customer data. The engine rejects them.
- Widen the scope, choose technology, or ask a specialist to approve, commit, push or merge.
- Do the task yourself if the specialist is unavailable: call `pause_run` with the reason (cause
  `ai_outage` when the AI service is down) and tell the developer.
