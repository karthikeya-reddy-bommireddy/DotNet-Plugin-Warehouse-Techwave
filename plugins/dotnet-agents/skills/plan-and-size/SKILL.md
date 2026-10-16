---
name: plan-and-size
description: Use when Gate 1 is approved and next_step asks for a plan, to break the approved adapted design into a dependency-ordered plan per module and to request the small-change path where it fits. Submits the plan with propose_plan. Do not use to start tasks, change gates or decide small-change eligibility (the engine decides).
license: Proprietary
metadata:
  version: 1.2.0
  agent: dotnet-orchestrator
  contract: plan/1
disable-model-invocation: true
---
# Plan and size

You propose; the engine validates the plan and decides small-change eligibility. The accepted plan
chooses the designer's skill: design note (small change), change design (brownfield) or module
design (greenfield).

## Procedure
1. A run covers one module: plan the run's module. If the adapted design shows that the work spans
   more modules, tell the developer; each module gets its own run.
2. Write the steps, one per step type, from this table (the agent is fixed by the type):

   | Type | Agent | Depends on |
   |---|---|---|
   | design | dotnet-designer | nothing |
   | code | dotnet-coder | design |
   | build | engine | code |
   | review | dotnet-reviewer | build |
   | test | dotnet-tester | build (or review) |
   | fix (optional) | dotnet-fixer | review and test |
   | security | dotnet-security | review and test (or fix) |
   | docs | dotnet-docs | review and test (or fix) |

   Every plan has design, code, build, review, test, security and docs, on the small-change path
   too. A step waits only for steps of its own stage or an earlier one; no cycles. Gates are the
   engine's; never add or move them. The engine repeats build-loop rounds itself: list each step once.
3. Mark only security and docs as `parallelizable: true`.
4. Size the module: S (1-3 files), M (4-15 files), L (more). Give a one-line reason.
5. Request `small_change` only when all of these hold, and say which facts support it:
   - brownfield run (a change to existing code), size S;
   - the adapted API design lists no contract, the adapted data model no entity;
   - no package dependency is `required` or `candidate`;
   - no authentication, authorization or secrets control is `required` or `unknown`, and nothing
     involves cryptography;
   - the change fits the file limit (default 5, `small_change.max_files` in `.agents/config.json`);
     the engine checks the limit on the design's scope.json.
   The engine checks these against the adapted design. If it refuses, its reasons are facts: tell the
   developer in one line and propose `full`.
6. Call `propose_plan(run_id, plan)`. If rejected, fix exactly the listed errors and resubmit once.
   If rejected again, show the errors to the developer. Then call `next_step`.

## Plan shape
```json
{ "schema": "plan/1", "run_id": "<run-id>", "module": "<Module>", "path": "full|small_change", "size": "S|M|L",
  "sizing_rationale": "<one line>",
  "steps": [ { "id": "S1", "type": "design", "agent": "dotnet-designer", "depends_on": [] },
             { "id": "S2", "type": "code", "agent": "dotnet-coder", "depends_on": ["S1"] },
             { "id": "S3", "type": "build", "agent": "engine", "depends_on": ["S2"] },
             { "id": "S4", "type": "review", "agent": "dotnet-reviewer", "depends_on": ["S3"] },
             { "id": "S5", "type": "test", "agent": "dotnet-tester", "depends_on": ["S3"] },
             { "id": "S6", "type": "security", "agent": "dotnet-security", "depends_on": ["S4", "S5"], "parallelizable": true },
             { "id": "S7", "type": "docs", "agent": "dotnet-docs", "depends_on": ["S4", "S5"], "parallelizable": true } ] }
```

## When the design note is not small
If the designer reports `blocked` with `not_small_change`, the engine has already moved the plan to
the full path. Do not propose a new plan: tell the developer why in one line, then call `next_step`;
the design is delegated again as a change design.

## Split as needed
Plan at step level. Split a step only when it fails or a specialist reports it is too large.
Do not pre-split everything.
