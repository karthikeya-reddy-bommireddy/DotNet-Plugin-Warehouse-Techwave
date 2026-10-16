---
name: escalation-policy
description: Use when submit_report returns a failure or rejection, when a build-loop round fails, when the engine flags stall_detected or same_failure_as_previous, when a fake fix is detected, when the run fails after the third round, or after a high or critical finding. Decides between retry, re-plan, design revision and asking the developer, within the engine's limit of 3 rounds. Do not use for normal successful steps.
license: Proprietary
metadata:
  version: 1.3.0
  agent: dotnet-orchestrator
  contract: engine-failure-signal/1
disable-model-invocation: true
---
# Escalation policy

The ENGINE counts the build loop's rounds (at most 3 per module), ends each round, classifies what failed and
chooses who works next. You interpret the signals and tell the developer. Never exceed, reset or ask to reset
the limit.

## 1. Read the signal
`get_run_state` -> `round`, `max_rounds`, `failure_class`, `stall_detected`, `same_failure_as_previous`,
`failures_ref` (the round's failures.json, every failure with its ID). `failure_class` is the most important
of: compile | test | analyzer | review | fake_fix | design | compatibility | report_rejected | specialist_failed.
`design` means the code differs from the approved design (an endpoint missing from or not in the approved
`api.yaml`, a criterion without a test, a requirement in no report, a change in no report); `compatibility` means
the public API or the existing tests changed beyond the approved scope.
`get_verification_summary(run_id, module)` gives PASS/FAIL per check on the current code.

## 2. Choose ONE action
| Condition | Action |
|---|---|
| A tool or AI service error (not a round failure) | Nothing; the engine retries or pauses. Tell the developer only if the run pauses. |
| report_rejected (one rejection) | Re-brief nothing: the same specialist fixes exactly the listed errors and resubmits. The third rejection fails its task and ends the round. |
| A round failed, round < 3, new failures | Retry: `next_step` names the fixer. Brief it with only the failure IDs from `round failures`, as its objective. |
| same failure again, or stall_detected | Re-plan: the engine puts the tester first. Brief it to write a test that reproduces the failure (by test name or finding), then brief the fixer. |
| review among the failures | After the fixer, `next_step` names the reviewer again for the blocking findings. |
| fake fix detected (`fake fix:` errors, failure_class fake_fix) | Treat as a failure, never as progress. The next brief says what must be put back. |
| design or compatibility among the failures | Retry like any failure: the fixer's brief names the `D-nn` and `C-nn` IDs. If the fixer finds the design itself must change, it returns `needs_decision`: present it. |
| The design changed since Gate 2 (`next_step` names Gate 2 again) | Not a round failure and never the fixer's work: load `gate-package`, request Gate 2 with the changed files and end the turn. |
| The design cannot meet an acceptance criterion | Propose a design revision to the developer with evidence (decision-request). |
| Round 3 failed: the run is Failed | Stop. The engine raised the escalation decision (revise-design, step-in, cancel) with a recommendation: present it with its `agents decide` command and end the turn. |
| Critical or high security finding | Never waive or downgrade it. At the Gate 3 request the engine raises a finding decision (fix, accept, defer): present it with its `agents decide` command. A fix counts only when the scan no longer reports the finding; `agents resume` runs the scans again. |
| A security scanner could not run (the security agent reports blocked) | Tell the developer which scanner and why (for example no vulnerability data from the package source: no network). Gate 3 waits until a scan runs every scanner. |

A fake fix is any of: deleted or skipped tests, weakened assertions, suppressed analyzers or warnings, new
catch-all or empty exception handlers, or edits outside the scope. The engine's diff checks report these.

## 3. After the developer's choice
- **revise-design:** the design step re-opens; the failures reach the designer as an answered question. Gate 2
  is approved again, then the build loop starts at round 1.
- **step-in:** the run is paused for the developer (`developer_stepping_in`). Only the developer resumes, with
  `agents resume <run>`; the engine then builds, runs the analyzers and the tests. Green ends the build loop;
  otherwise the run fails again with a new escalation decision. Never call `resume_run` for them.
- **cancel:** the run is cancelled; its worktree and records stay.

## 4. Tell the developer (at most 5 lines)
What failed (by ID and class), what was tried, what the engine recommends, what you need from them and the
command that gives it.

## Never
Request a 4th round, edit code, run builds yourself, choose an escalation option, resume a run paused for
the developer, or call a failing step "nearly done". Call a module ready only when
`get_verification_summary` shows every check passing.
