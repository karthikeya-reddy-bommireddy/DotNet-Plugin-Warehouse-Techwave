---
name: documentation
description: 'Use when the brief asks the dotnet-docs agent to document a finished module: XML documentation for public APIs, the module README and change notes with requirement IDs. Writes only to documentation paths and XML doc comments in scope. Do not use to change code behaviour, invent behaviour that the code and design do not show, or document internal implementation details as public contracts.'
license: Proprietary
metadata:
  version: 1.2.0
  agent: dotnet-docs
  contract: documentation/1
disable-model-invocation: true
---
# Documentation

You work after the build loop and before the security review. The engine checks your work by code when you report
`done`, and again at Gate 3 on the current code.

## Produce
1. **XML documentation** for every public API member the run added or changed (since the run's starting commit,
   outside test projects): public and protected members of public types, interface members, enum members and a
   record's positional parameters. Each one needs:
   - a non-empty `<summary>`: what it is for and how it behaves, not how the code does it;
   - a `<param name="...">` for every parameter (a record's positional parameters go on the record);
   - a `<returns>` when it returns a value (not for `void`, `Task` or `ValueTask`);
   - an `<exception>` for each exception a caller must handle (not checked by the engine; still expected).
   `<inheritdoc/>` counts as complete when the member implements or overrides a documented one. Existing members the
   run did not change need nothing from you.
2. **Module README**, written or updated in this run, in the module's source folder inside the approved scope
   (greenfield `src/<Module>/README.md`; brownfield the first production folder of the scope): purpose, how to use it,
   configuration settings (names and meaning only, never values or secrets), dependencies, how to run its tests.
3. **Change notes** (`docs/changes/<run-id>.md`): what changed; every requirement ID of the module; every decision ID
   in your brief's `decisions` (the developer's decisions so far); known limitations; pre-existing issues not addressed.
   Write the IDs exactly (`REQ-001`, `DEC-003`).

## Rules
- Change code files only in comments and whitespace. The engine compares each C# file with how the last accepted turn
  left it: a changed statement, literal or signature, a new or deleted code file, a `#pragma` or `#if`, or a change to
  any file other than Markdown and C# comments rejects your report (WD-77). Do not touch `.csproj`, settings or
  configuration files.
- Source of truth: the code and the approved design. If they disagree, document the code and add an open question.
- No secrets, connection strings, internal hostnames or customer data in any document. The security review scans your
  documents too.
- Keep it short; link to `api.yaml` and design files instead of repeating them.
- Write only inside your task's scope: the approved scope and `docs/changes/<run-id>.md`. The engine rejects a report
  whose turn changed anything else, and one that leaves out a file it changed.

## If the report is rejected
The errors name each problem exactly: `Not documented yet: <file>:<line>: <member>: <what is missing>`, a README or
change-notes problem, or `<file>: it changes code, not only comments`. Fix exactly those and resubmit; put back any code
change you made.

## Done when
Every new or changed public API member is documented, the README is updated, the change notes name every requirement
and decision ID, and report/1 is submitted with every file the turn changed in `outputs` (`kind: "doc"`, with their
requirement IDs).
