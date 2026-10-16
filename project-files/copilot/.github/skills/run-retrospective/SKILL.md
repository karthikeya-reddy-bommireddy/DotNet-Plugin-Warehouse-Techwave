---
name: run-retrospective
description: Use when a run reaches Completed or Cancelled, or when the developer asks what could be improved. Drafts lesson proposals with evidence and stores them with propose_lesson for human review. Do not use to edit agents, skills, hooks or policies; lessons change the warehouse only through a reviewed pull request.
license: Proprietary
metadata:
  version: 1.1.0
  agent: dotnet-orchestrator
  contract: lesson/1
disable-model-invocation: true
---
# Run retrospective

## Procedure
1. Read the run summary: `get_run_state`, verification summary, escalations, rejected reports,
   decisions and `get_context_budget`. Check `get_approved_lessons` so you do not propose one twice.
2. Find at most 3 lessons. A lesson is a repeated, evidenced problem, for example:
   - a brief field that specialists kept asking about;
   - a recurring false positive from the reviewer;
   - a template that caused rework.
3. For each lesson call `propose_lesson` with lesson/1: `run_id`, `pattern` (one sentence), `evidence`
   (at least two IDs of this run's records: tasks, reports, findings, builds, dispatches, decisions,
   questions, checkpoints or approvals), `proposed_change` (`file`: a warehouse source file under
   agents/, skills/, hooks/, policies/, templates/, contracts/ or knowledge/; `change`: what to change)
   and `expected_effect`. The engine refuses evidence it has no record of.
4. Tell the developer each lesson ID and the commands that decide it: `agents lesson approve <id>` or
   `agents lesson reject <id>` (`agents lesson list` shows them). An approved lesson reaches later
   context packs as a note; the warehouse changes only through a reviewed pull request.

## Never
- Write or edit any agent, skill, hook, policy or template file.
- Store code, secrets or customer data in a lesson.
- Generalise from one occurrence; one occurrence is a note, not a lesson.
