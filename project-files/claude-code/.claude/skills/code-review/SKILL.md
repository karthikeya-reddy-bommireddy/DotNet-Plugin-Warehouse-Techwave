---
name: code-review
description: Use when the brief asks the dotnet-reviewer to review a module's diff after a successful build and analyzer run. Judges design conformance, correctness, security, maintainability and test gaps and returns finding/1 records with quoted evidence. Do not use to edit code, rerun builds, restate analyzer warnings that are already reported, or approve the change.
license: Proprietary
metadata:
  version: 1.2.0
  agent: dotnet-reviewer
  contract: finding/1
disable-model-invocation: true
---
# Code review

The build and the analyzers already passed. You review what tools cannot judge. You never edit code.

## Inputs
- `get_diff(task_id)`: every changed file with its status and whether it is inside the approved scope, and the
  unified diff (secrets redacted). A file outside the approved scope is a finding of its own.
- From the brief, with `read_context`: the design approved at Gate 2, the approved scope, the analyzer results
  (warnings marked preexisting were in the baseline) and, in brownfield runs, the codebase profile and baseline.
- Open the changed files themselves for context; read only the parts you need.

## Check, in this order
1. **Design conformance:** components, contracts and data match the approved design; nothing extra; nothing outside
   the approved scope.
2. **Correctness:** acceptance criteria and their edge cases (boundaries, empty input), null handling, async misuse
   (blocking calls, missing cancellation where the design requires it), resource disposal, concurrency.
3. **Security:** input validation, authorization and owner checks on every new endpoint or query, identity only
   from validated claims, no secrets or personal data in code or logs, parameterised SQL, safe serialization.
4. **Error handling:** no swallowed exceptions, no catch-all handlers, meaningful errors.
5. **Maintainability:** naming and structure follow the existing patterns and standards.
6. **Test gaps:** acceptance criteria with no obvious test path (for the tester), as `info`.

## Reviewing again (rounds 2 and 3)
When the last round failed on blocking findings, the engine sends you again after the fixer, and the context
pack lists `round failures` with those findings by ID. Review the whole diff as before. Report a finding again
only if the current code still has it, quoting the current code; a fixed finding is simply not reported. The
engine judges the round by your latest report.

## Each finding (finding/1)
`id` (your own, `F-` plus a number; the engine assigns the stored ID), `source: "reviewer"`, `rule` (a short name,
never an analyzer rule ID such as CA1822), `severity`, `file`, `line`, `evidence`, `recommendation`, `preexisting`,
`disposition: "open"`.

- `evidence`: the code exactly as it is at `line` or within three lines of it, at most two lines. The engine
  compares it with the file (spacing may differ); a finding whose evidence is not there is rejected.
- `preexisting: true` when the code was not changed in this run (the diff does not touch the file).
- Severity: critical = security hole or data loss; high = wrong behaviour against an acceptance criterion, a
  missing authorization check, or a design violation; medium = likely bug or maintainability risk; low = style;
  info = suggestion.

## Rules
- One finding per problem; never repeat an analyzer warning.
- Evidence or it did not happen: no finding without a quoted line.
- Do not request changes outside the design's scope; list them as `info`.
- Never set a disposition other than `open`: fix, accept and defer are the developer's decisions.
- `status: "done"` when the review is complete, even with findings. The engine decides blocking.
