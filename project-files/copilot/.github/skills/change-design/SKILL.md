---
name: change-design
description: Use when the brief asks the dotnet-designer to design a change to an existing (brownfield) module. Produces a change design that separates new from modified elements, the exact allowed files in scope.json, and compatibility notes against the baseline. Do not use for new modules (use module-design) or for small changes that the engine routed to the design-note path.
license: Proprietary
metadata:
  version: 1.1.0
  agent: dotnet-designer
  contract: change-design/1
disable-model-invocation: true
---
# Change design (brownfield)

You design a change to existing code. The baseline (Roslyn scan, codebase profile, existing
failures) is the truth about today's code; do not contradict it.

## Inputs (from the brief, read with `read_context`)
- The requirement registry: the requirements of this change with their acceptance criteria.
- Adapted design for the affected requirement IDs, including `adapter-brownfield-mapping` and
  `adapter-compatibility-analysis` outputs.
- Codebase profile and symbol index from the engine's code scanner, when the brief lists them. Until
  then, the brownfield mapping is the evidence; read the code itself with your read and search tools.
- The plan, the tech profile and standards.
- A change request from a rejected gate, when the brief's answers hold one: apply it.

## Produce (under `design/<Module>/`)
1. `change-design.md` from `assets/change-design-template.md`:
   - What is NEW (types, endpoints, tables) and what is MODIFIED (with current symbol names).
   - Why each change is needed (requirement IDs).
   - Public API impact: none, additive or breaking. Breaking changes need a developer decision.
   - Data impact: migrations needed? Backward compatibility of stored data.
   - Existing patterns to follow (cite file and line ranges).
   - The acceptance table: every acceptance criterion of the change by ID, with its test level.
2. `scope.json` (scope/1): the exact files and folders that may change, read-only paths, and
   `public_api_changes`; `run_id` and `module` of this run and `approved_by_gate: "proposed"`. Keep it
   as small as possible. Never list `design/`, `.agents/`, `.git/` or the whole repository.
3. `api.yaml` changes, validated with `validate_openapi`, when an HTTP API changes (the engine requires
   it when the adapted API design has HTTP contracts).

## Rules
- Follow the existing conventions unless a requirement or recorded decision says otherwise.
- Never plan a broad refactor. Anything beyond the requirement is out of scope; list it as a
  suggestion for later.
- Pre-existing failures and warnings from the baseline are not yours to fix; list them separately.
- Unknown consumers of a changed contract are an open question, not "probably none".
- Use only IDs from the registry.

## Done when
scope.json lists every file the coder will need and nothing more; every modified public symbol is
listed with its impact; every acceptance criterion is in the acceptance table; report/1 lists every
file and is accepted.
