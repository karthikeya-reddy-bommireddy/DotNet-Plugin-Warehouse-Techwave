---
name: failure-fix
description: Use when the brief asks the dotnet-fixer to resolve the failures of a build-loop round (build errors, failing tests, new analyzer warnings, blocking review findings, a rejected fake fix, a difference from the approved design or a compatibility break), identified by ID in the round's failures, within the module's scope. Makes the smallest change that removes each cause and verifies it with run_build, run_analyzers and run_tests. Do not use for new features, design changes, failures outside the brief, or anything that would weaken tests or suppress analyzers.
license: Proprietary
metadata:
  version: 1.2.0
  agent: dotnet-fixer
  contract: failure-fix/1
disable-model-invocation: true
---
# Failure fix

You fix exactly the failures of the round before. Nothing else. The engine counts the rounds (three at most)
and checks your result itself.

## Inputs
- `round failures` (a `failures.json` in the context pack, read with `read_context`): `failure_class`, every
  failure with its engine ID, class, message, file and line, and the flags `stall_detected` and
  `same_failure_as_previous`. The brief names the failure IDs you own.
- The design approved at Gate 2, the approved scope and the tech profile. Open only the files involved.

## Procedure
1. For each failure ID in the brief, find the root cause before changing anything:

   | class | What it is | Where the fix goes |
   |---|---|---|
   | compile | A build error (`B-nnnn-Enn`) | The code that fails to compile |
   | test | A failing test, by its full name | Production code. A test that contradicts its acceptance criterion is not edited: return `needs_decision` with the evidence |
   | analyzer | A new warning (`B-nnnn-Wnnn`) | The code the warning points to |
   | review | A blocking finding (`F-nnnn`) with its recommendation | The code the finding quotes, within the design; a finding that contradicts the design is `needs_decision` |
   | fake_fix | The previous attempt hid a failure | Put back what it removed or weakened first, then fix the real cause |
   | design | The code differs from the approved design (`D-nn`): an endpoint of the approved `api.yaml` is missing or one is extra, an acceptance criterion has no test, a requirement is in no report's outputs, or a changed file is in no report | Implement the missing endpoint or remove the extra one, as the design says; list a file you changed in your outputs (or revert it). A test belongs to the tester, a design change to the developer: return `needs_decision` |
   | compatibility | The public API lost or added members the approved scope does not allow, or an existing test is gone (`C-nn`) | Put back the removed member or test; a needed breaking change is `needs_decision` |
   | report_rejected, specialist_failed | The previous task ended without a fix | Read its errors or summary in `failures.json`; fix the failures it left |

2. `stall_detected` or `same_failure_as_previous` means the previous attempt did not touch the cause. Do not
   repeat it: look for a different cause (the reproducing test the tester wrote this round shows it).
3. Check every file you will edit with `scope_check`; make the smallest change that removes the cause.
4. Verify on the current code: `run_build`, then `run_analyzers`, then `run_tests`. Run them again after any
   later edit: the engine only counts runs on the code as it is when you report.

## The engine checks before `done` counts
- the last `run_build` and `run_analyzers` passed, and `run_tests` has no failing test (failures marked
  preexisting were already failing in the baseline), all on the current code;
- the code is not the same as at the end of an earlier round (`stall_detected`: an identical retry is never
  a fix);
- the fake-fix guard finds nothing. It compares your change with the code before your turn and rejects
  (`fake fix: ...`) deleted or renamed tests; newly skipped tests (`Skip`, `Ignore`, `Explicit`, `Assert.Skip`,
  `Skip.If`); tests with fewer assertions; `#pragma warning disable`, `[SuppressMessage]`, `<NoWarn>`,
  `RunAnalyzers`, `TreatWarningsAsErrors` or `EnableNETAnalyzers` set to false, `AnalysisLevel` none, and
  `.editorconfig` severities lowered to none, silent or suggestion; empty catch blocks and catch-all handlers
  that do not rethrow. A fake fix fails the round and stays visible to the guard until it is undone.

## When you cannot fix it
If the failure remains after your attempt, or its cause is in the design, return `failed` with the cause you
found, what you tried and what you recommend (for example a design change). Do not guess further and never
weaken a check to get green; after the third failed round the developer decides.

## Report (report/1)
`outputs`: every file your turn created, changed or deleted, with `kind: "code"` and its requirement IDs (a deleted
file by its path; the engine rejects a report that leaves one out). `summary`: each failure ID with its
root cause in one line, the last build, analyzer and test runs by ID, and anything still failing.
