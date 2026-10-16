---
name: implement-module
description: Use when the brief asks the dotnet-coder to implement an approved module design, change design or design note inside the run worktree and the scope approved at Gate 2. Writes production code as small, reviewable changes, builds with run_build and checks analyzers with run_analyzers. Do not use to change the design, write the acceptance-criteria tests (the tester does), touch files outside the approved scope, or fix failures from a review or test round (the fixer does).
license: Proprietary
metadata:
  version: 1.2.0
  agent: dotnet-coder
  contract: implement-module/1
disable-model-invocation: true
---
# Implement module

You write code for an approved design. The design, the approved scope and recorded decisions are fixed.

## Before writing
1. Read the brief's inputs with `read_context`: the approved design folder (`design/<Module>/`), the
   approved scope (`approved-scope.json`) and, in brownfield runs, the codebase profile, the code index and
   the baseline. Find code through the index (symbol, file and line) before reading files; read only the
   line ranges you need.
2. List every path you will create or edit and call `scope_check(task_id, paths)`. A denied path is never
   edited: if the work needs it, return `needs_decision` with the path and the reason (a scope change is the
   developer's decision).
3. API-layer work (controllers, use cases, entities, EF Core, authentication, authorization, validation,
   Program.cs, logging): open `api-layer-work` and follow it together with this skill.
4. Prefer the templates and generators the tech profile names for boilerplate; write by hand only the logic.

## While writing
- Implement one component at a time; keep each change small and reviewable.
- Put requirement IDs in the report, not in code comments.
- Follow the tech profile: framework, nullable reference types, async patterns, error handling, logging (no
  secrets or personal data in logs), configuration through options; never hard-coded secrets, connection
  strings or URLs.
- Never add a package that is not in the tech profile. If one is needed, stop and return `needs_decision`.
- No public API or schema change beyond the design. Brownfield: follow the patterns the change design cites.
- HTTP API: implement exactly the operations of the approved `api.yaml`, no fewer and no extra. Write routes as
  string literals or constants (minimal API `Map*` calls and their `MapGroup` prefixes, or controller route
  attributes), so the engine can compare them with the design; it reads them from the code at every round's end.

## Verify
1. `run_build`. Fix the errors your change caused, by their IDs; every message is in `details_ref`.
   Errors marked `preexisting` were in the baseline: list them in the report, do not fix them.
2. `run_analyzers`. Fix every new warning in the code. Never add `#pragma warning disable`,
   `[SuppressMessage]`, `NoWarn`, `.editorconfig` severity changes, empty catch blocks or catch-all handlers to
   get a clean result.
3. If a build cannot run (`timeout` or `error`), say so in the report with the problem the tool gave; never
   report a build as passed that did not run.

## Report (report/1)
`outputs`: every file your turn created, changed or deleted, with `kind: "code"` (security tests from
`api-layer-work`: `kind: "test"`) and its requirement IDs; a deleted file is listed by its path. The engine rejects
a report that leaves out a file the turn changed, and Gate 3 counts only listed files as explained. `summary`: what
was implemented, the last build and analyzer results by ID, preexisting errors, and what is left for the tester.
Submit with `submit_report`.
