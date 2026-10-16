---
name: security-review
description: 'Use when the brief asks the dotnet-security agent to review a module''s change for security: run the security scanners (secrets, vulnerable packages, licences, security analyzers) with run_security_scans, report every new critical or high finding with its scan ID, explain and prioritise the findings and check the controls the design requires. Do not use to waive, downgrade or decide a finding, to edit code, or to replace the scanners.'
license: Proprietary
metadata:
  version: 1.1.0
  agent: dotnet-security
  contract: finding/1
disable-model-invocation: true
---
# Security review

The scanners are the source of truth. You explain, prioritise and recommend; the engine checks your report against
the scan.

## Inputs
- `run_security_scans(task_id)`: the scan of the current code (finding/1 records with scan IDs `F-B-nnnn-nn`):
  - secrets: the built-in scanner (the hook policy's patterns and well-known key formats), plus gitleaks when installed;
    every secret is critical; files such as `.env` or `*.pfx` count too;
  - vulnerable packages: `dotnet list package --vulnerable --include-transitive`, with the advisory's severity;
  - licences: each package's licence against the allow-list in `security-policy.yaml`; a licence outside it, or no
    licence expression (a licence URL or file only) and no exception, is high;
  - security analyzers: the .NET security rules, with the severity table of `security-policy.yaml`.
  Each finding says whether it was already on the run's starting commit (`preexisting: true`). A pre-existing finding
  the change touches counts as new (its evidence says so).
- The diff (`get_diff`), the design's security section and the adapter's security mapping.

## Procedure
1. Call `run_security_scans(task_id)`. If its status is `error`, a scanner could not run (for example no vulnerability
   data from the package source): report `blocked` with the reason. Never report a scan as clean that did not run.
2. For every NEW finding (new first, most severe first): say what it means in one sentence and recommend a fix (a code
   change, an update to an approved package version, configuration, or for a licence: a package under an allowed
   licence, or the security reviewer's check). The full list is in the scan's `details_ref`.
3. Check the design's required controls against the diff: authorisation on new endpoints, input validation, data
   protection for classified data, secrets read from configuration or the secret store. A missing control is a
   finding with `source: "security"`, a new ID such as `F-C1`, the file and line, and evidence quoted exactly from that
   line (at most two lines).
4. Never set `disposition` (always `open`) and never lower a severity. Only the developer decides a finding:
   `agents decide` with fix, accept or defer. A fix counts only when the scanner no longer reports the finding.

## Report (report/1)
`findings`: every new critical or high finding of the scan with its scan ID, `source: "scanner"`, the scan's rule,
file and pre-existing flag, and your recommendation; other findings that matter; then the control findings. The engine
rejects a report that leaves one out, lowers a severity or changes a flag. `summary`: counts by severity, new against
pre-existing, which scanners ran, and the top three actions. Secrets: the type and place only, never the value.
