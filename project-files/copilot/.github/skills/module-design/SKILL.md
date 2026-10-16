---
name: module-design
description: Use when the brief asks the dotnet-designer for a full design of one module after Gate 1, in greenfield work or for new modules. Produces design.md, api.yaml (when the module has an API), data-model.md and the acceptance-criteria links from the approved adapted design and tech profile. Do not use for changes to existing code (use change-design) or for small changes (use design-note).
license: Proprietary
metadata:
  version: 1.1.0
  agent: dotnet-designer
  contract: module-design/1
disable-model-invocation: true
---
# Module design

You turn the approved adapted design (Gate 1) into a buildable design for ONE module.
Decisions already recorded in the tech profile are fixed. New choices go back as `needs_decision`.

## Inputs (from the brief, read with `read_context`)
- The requirement registry: every requirement with its acceptance criteria. The design covers all of them.
- Adapted design files: `design/adapted/*.json` (architecture, solution structure, API, data,
  security, testing strategy).
- The plan and the tech profile `.agents/tech-profile.yaml` (architecture style, framework, data
  access, testing).
- Standards from `knowledge/` and project overlays, when listed.

## Produce (under `design/<Module>/`)
1. `design.md` from `assets/design-template.md`: purpose, requirements covered, components and
   responsibilities, project placement, dependencies, cross-cutting concerns (validation, errors,
   logging, configuration, security), and the acceptance-criteria table.
2. `api.yaml` (OpenAPI 3.0 or 3.1) when the module exposes an HTTP API; the engine requires it when the
   adapted API design has HTTP contracts. Every operation has:
   - a unique `operationId` and a `summary`;
   - a success (2xx) response and an error response (4xx, 5xx or `default`);
   - `x-requirement-ids`: the registry IDs it serves.

   No credentials in server URLs and no secrets in examples. Fields, routes and status codes trace to
   requirements or the adapted API design; unknown ones become open questions, never inventions. Call
   `validate_openapi(task_id, "design/<Module>/api.yaml")` and fix every error.
3. `data-model.md` when the module owns data (required when the adapted data model lists entities):
   entities, attributes, keys, relationships, lifecycle and retention rules, each with requirement IDs.
4. `scope.json` (scope/1), a proposal: `run_id` and `module` of this run, `approved_by_gate: "proposed"`,
   `allowed_paths` with the module's project and test folders (for example `src/<Module>/**` and
   `tests/<Module>.Tests/**`), optional `read_only_paths`, and `public_api_changes` (`additive` for a
   new API). Never list `design/`, `.agents/`, `.git/` or the whole repository.

## Rules
- Every component, endpoint, entity and test objective lists the requirement IDs it serves.
- Every acceptance criterion in the registry appears by ID in the acceptance table with the test
  level that will verify it (unit, integration or contract).
- Use only IDs from the registry.
- Use the patterns already recorded in the tech profile. Do not introduce new packages; if one
  seems necessary, return `needs_decision` with the reason.
- Keep the design small: no layers, interfaces or projects that no requirement needs.

## Done when
`validate_openapi` passes (if api.yaml exists); every requirement and acceptance criterion in the
registry is covered or listed as an open question (then the status is `needs_input`); no unapproved
technology appears; report/1 lists every file and is accepted.
