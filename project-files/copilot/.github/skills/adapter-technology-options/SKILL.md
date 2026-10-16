---
name: adapter-technology-options
description: Use when adaptation batch A5 runs, after architecture, data, API, dependency, standards and security mapping, to turn every unresolved technology or version choice into a decision request with 2-3 options from the approved list, pros, cons and one recommendation. Do not use to make a choice, to offer options outside the approved list, or before the design mapping exists.
license: Proprietary
metadata:
  version: 1.1.0
  agent: dotnet-adapter
  contract: adapter-technology-options/1
disable-model-invocation: true
---
# Technology options

The adapter skills deliberately leave technology choices as `unknown`. This skill collects those
choices and prepares them for the developer. The developer decides; you never do.

## Inputs
- Outputs of earlier adapter skills (`design/adapted/*.json`): unknowns, assumptions and decisions
  marked unresolved.
- Approved options: call `get_approved_options(category)` for each category.
- Decisions already recorded: the brief's `decisions` and the tech profile. Never raise them again.

## Categories
`architecture-style`, `target-framework`, `api-style`, `api-documentation`, `data-access`, `database`,
`messaging`, `caching`, `authentication`, `authorization`, `object-mapping`, `input-validation`,
`testing-framework`, `integration-test-database`, `assertion-library`, `logging`, `hosting`, `ci-cd`,
`code-exploration`. A choice outside these categories is not a decision: record an open question asking
whether the approved list needs that category.

## Procedure
1. List every unresolved choice from the earlier outputs. Skip choices already recorded (the brief's
   `decisions` or the tech profile).
2. Merge duplicates (the same choice raised by two skills becomes one decision).
3. For each choice, call `get_approved_options(category)`.
   - An option marked `fixed: true`: the company tech spec has decided. Raise no decision; record the
     option as an assumption with the source "company tech spec", and use it in the design.
   - 0 approved options, or 1 that is not fixed: do not invent options. Record an open question asking
     whether the approved list should be extended.
   - 2 or more: keep the 2-3 most relevant to the requirements.
4. For each option: one-line summary, up to 3 pros, up to 3 cons, impact on this design.
5. Recommend exactly one option, with the requirement, NFR or existing-code evidence that supports
   it. In brownfield work, prefer what the codebase already uses unless a requirement says otherwise,
   and say so.
6. Write the result to `design/adapted/adapter-technology-options.json`. With no unresolved choice left,
   write `"decisions": []`. Every decision listed here must also reach the report as a decision
   (`adapter-report`); the engine rejects a report that leaves one out.

## Output
```json
{
  "skill": "adapter-technology-options", "version": "1.0.0", "status": "complete|partial|failed",
  "decisions": [ {
    "id": "DEC-001", "category": "data-access", "question": "<text>",
    "options": [ { "id": "<approved-option-id>", "label": "<label>", "version": "<from approved list>",
                   "pros": ["..."], "cons": ["..."], "impact": "<impact on this design>" } ],
    "recommended": "<approved-option-id>", "rationale": "<= 300 characters",
    "linked_requirements": ["REQ-001", "NFR-002"], "raised_by": ["adapter-data-modeling"] } ],
  "openQuestions": [ { "questionId": "Q-...", "question": "<text>", "affectedRequirementIds": [] } ]
}
```

## Never
Choose, imply a choice ("we will use"), copy a version from memory instead of the approved list, or
ask about a choice the developer already made.
