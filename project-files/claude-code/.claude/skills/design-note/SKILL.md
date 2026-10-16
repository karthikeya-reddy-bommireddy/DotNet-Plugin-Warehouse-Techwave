---
name: design-note
description: Use when the engine has routed a change to the small-change path and the brief asks for a design note. Produces a one-page design note and scope.json; every later check and gate still runs. Do not use when the change adds a public API, a schema change, a new package, or touches security-sensitive code (the engine would not route it here).
license: Proprietary
metadata:
  version: 1.1.0
  agent: dotnet-designer
  contract: design-note/1
disable-model-invocation: true
---
# Design note (small-change path)

A short design for a small change. It replaces the full module design, not the checks: Gate 2,
the build loop, security review and Gate 3 still run.

## Inputs (from the brief, read with `read_context`)
- The plan: it states the file limit for this project (default 5).
- The requirement registry: the requirements of this change with their acceptance criteria.
- The adapted design, the tech profile and the existing code (read and search tools).

## Produce (under `design/<Module>/`)
1. `design-note.md` from `assets/design-note-template.md`, one page (at most 6,000 characters):
   problem, change, files, tests with the acceptance criteria they verify, requirement IDs, risks.
   - The Requirements row names the requirement IDs the change serves.
   - The tests table lists every acceptance criterion of those requirements by ID.
   - The Files line names exactly the paths in scope.json.
2. `scope.json` (scope/1) with `run_id` and `module` of this run, `approved_by_gate: "proposed"`,
   `public_api_changes: "none"` and `allowed_paths` listing exact files (no folders or wildcards),
   no more than the limit.

No `api.yaml` and no data model: a change that needs them is not small.

## Rules
- If, while writing, you find the change needs a new public API, a schema change, a new package,
  more files than the configured limit, or security-sensitive code (authentication, authorization,
  secrets, cryptography): stop and return `blocked` with a summary that starts with
  `not_small_change:` and gives the reason. The engine moves the work to the full path and you are
  briefed again for a change design.
- Name the existing tests that cover the area and the new tests needed.
- Use only IDs from the registry.

## Done when
The note fits on one page, scope.json is exact and within the limit, every acceptance criterion of
the named requirements is linked, report/1 lists both files and is accepted.
