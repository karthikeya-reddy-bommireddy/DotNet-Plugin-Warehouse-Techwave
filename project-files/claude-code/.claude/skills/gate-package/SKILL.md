---
name: gate-package
description: Use when next_step requests approval for Gate 1 (adaptation and technology), Gate 2 (module design) or Gate 3 (final change against the approved design). Assembles gate-package/1 with evidence and calls request_approval. Do not use to approve anything; only the developer approves through the engine's channel.
license: Proprietary
metadata:
  version: 1.6.0
  agent: dotnet-orchestrator
  contract: gate-package/1
disable-model-invocation: true
---
# Gate package

A gate is a stop where a person approves. Agents cannot approve; you only prepare the package.
A "yes" in chat is not an approval: only `agents approve` records one.

## What each gate covers
| Gate | The developer approves | Evidence keys (required first) |
|---|---|---|
| Gate 1 | Adapted .NET design and technology decisions | `validator`: `design/adapted/adapter-validation.json`; optionally the adapted outputs |
| Gate 2 | Module design (or design note) and scope | `design`: `design/<Module>/design.md`, `change-design.md` or `design-note.md`; `scope`: `design/<Module>/scope.json`; `api` when there is an api.yaml |
| Gate 3 | Final change against the approved design | `build` and `tests`: the last build and test runs' `details_ref`; `verification_summary`: `.agents/runs/<run>/verification-summary.json`; `design_conformance`: `.agents/runs/<run>/design-conformance.json` (call `get_verification_summary` first; it writes both); `security`: the last security scan's `details_ref`; `docs`: `docs/changes/<run>.md` |

Evidence values are paths, never pasted content: relative to the run worktree, or under `.agents/`
for engine records such as accepted reports.

## Gate 3: what the engine checks first
Before the developer is asked, every check of `get_verification_summary` must pass on the current code (build,
analyzers, tests, review, acceptance criteria, fake-fix guard, scope, compatibility with the starting commit,
design conformance, rounds, security, docs). Security means the security agent reported and the last scan ran on the
current code with every scanner. Docs means the docs agent reported and, on the current code, every public API member
added or changed since the starting commit has XML documentation, the module README was updated in this run, and the
change notes name every requirement of the module and every decision taken before the docs task. When the docs row
fails because a change after the docs turn (the developer's fix) left public members undocumented (WD-80), tell the
developer which members, and offer two choices: document them yourself, or send the task back to the docs agent.
Nothing happens automatically: wait for the developer's choice; the next request checks again. Design conformance is the
diff-vs-design check: the design unchanged since Gate 2, the files inside the scope, the HTTP endpoints of the
approved `api.yaml` (none missing, none extra), a test per acceptance criterion, every requirement verified (its
criteria's tests passed and a changed code file is linked to it), every change in a report or a developer step-in. A critical or high finding (review or security, new or
touched by the change) that is still open needs the developer's decision: the engine raises it (fix, accept or defer)
when you request Gate 3. Pre-existing security findings the change does not touch never block; they are listed.
In the package and your message, show from the summary:
- the findings: resolved ones (fixed in a later round or no longer reported by a scan) separately from accepted or
  deferred ones, with who decided; security findings new and pre-existing in separate lists (never call a licence or
  vulnerability "safe" because it was there before);
- every scope widening the developer chose (`scope_amendments`), with its paths;
- the decisions taken for Gate 3, which the package lists;
- from `design-conformance.json`: what is listed without failing (unconfirmed operations, endpoints the change
  removed, routes the engine could not read, files the developer changed) and, per requirement, the code that
  verifies it (`requirements_verified`).
If the design changed since Gate 2, the engine sends the run back to Gate 2 instead: `next_step` names Gate 2
with the changed files. Request Gate 2 again with those files in the summary; after the developer's approval the
build loop starts a new pass.
Gate 3 approves exactly the current code: any change afterwards needs a new request.

## Procedure
1. Call `get_run_state` and confirm `next_step` names this gate.
2. Check preconditions: no pending decision and no open blocking question. If any is open, present
   it with its command and stop.
3. Build gate-package/1:
   - `schema` "gate-package/1", `run_id`, `gate` (the current gate);
   - `summary`: plain language, five lines at most (800 characters);
   - `decisions`: the decided decision IDs; every decision recorded for this gate must be listed;
   - `blocking_questions_open`: 0;
   - `evidence` as in the table; `risks` and `next_steps` as short lines;
   - `statement`: "Approval is yours; agents cannot approve."
4. Call `request_approval(run_id, gate, package)`. If `preconditions_met` is false, show `missing`
   and stop. Otherwise the run is paused and the engine records exactly what is approved: if the
   design or evidence changes afterwards, approval is refused and a new package is needed.
5. Show the summary, the evidence paths, the risks and non-blocking open questions, then end the
   message with this statement and the commands from the result, and end the turn:
   "Approval is yours; agents cannot approve.
    Approve: agents approve <run> <gate>
    Request changes: agents reject <run> <gate> --comment \"...\""

## After the developer decides
- Approved: call `next_step`. After Gate 2 the approved scope is fixed for the coder.
- Rejected: the engine records the comment as an answered question, re-opens the work (Gate 1 from
  the first adaptation batch, Gate 2 from the design, or the step the developer chose with `--step`)
  and the next brief carries the comment. Call `next_step`; never re-request approval for unchanged
  work.
