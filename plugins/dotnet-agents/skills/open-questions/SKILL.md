---
name: open-questions
description: Use when the BA validator, a specialist report (status needs_input) or a contradiction leaves information missing. Turns gaps into precise, answerable questions, records them with add_open_question and tracks them to closure, batched per gate. Do not use to answer questions yourself or to present technology choices (use decision-request).
license: Proprietary
metadata:
  version: 1.1.0
  agent: dotnet-orchestrator
  contract: open-question/1
disable-model-invocation: true
---
# Open questions

Missing information becomes a question for a person. It never becomes an assumption.

## Write a good question
Each question must:
- name what it is about in `refs`: requirement, rule, decision, task or finding IDs;
- quote the conflicting or missing text (short);
- say why it matters (what is blocked);
- offer answer `options` when the answer is discrete (for example "14 days" / "30 days" / "other");
- say who should answer in `ask`: `developer` or `ba`.

Example:
> BR-007 says refunds within 30 days; BR-009 says 14 days. Which applies to card payments?
> refs: BR-007, BR-009 · blocking · gate-1 · ask: ba · options: 14 days / 30 days / other

## Procedure
1. Collect gaps from the validator's findings and questions, `needs_input` reports and conflicting
   reports. One contradiction is one question naming both IDs.
2. Call `add_open_question(run_id, text, refs, blocking, gate, ask, options)`:
   `blocking` is true when the current step or the next gate cannot finish without the answer;
   `gate` is the gate the answer is needed for (`gate-1`, `gate-2`, `gate-3` or `none`).
   The engine returns the same ID for a question it already has (`duplicate: true`): ask it once.
3. Present blocking questions now, with the `command` the engine returns, and end the turn. Batch
   non-blocking questions with the next gate package.
4. The developer answers with `agents answer <run> <question-id> "..."`. An answer typed in chat is not
   recorded until the engine records it.
5. After an answer is recorded, call `next_step`; the answer is in the next context pack.

## Never
- Answer, close or downgrade a question yourself.
- Ask more than 5 questions in one message; group the rest by gate (`list_open_questions`).
- Ask about something already decided or answered (check `get_run_state` first).
