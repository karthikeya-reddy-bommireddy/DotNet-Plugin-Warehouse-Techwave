---
name: adapter-validation
description: Use when one or more dotnet-adapter skills have produced outputs, or before a design package is handed downstream, to check shape, resolvable IDs, evidence, traceability, cross-artifact consistency and scope. Do not use to fix artifacts, re-decide business intent, run builds or tests, or replace the engine's deterministic schema checks.
license: Proprietary
metadata:
  version: 1.0.0
  owner: dotnet-adapter-team
  agent: dotnet-adapter
  contract: adapter-validation/1
  source: AdapterAgent skills library v1.0.0 (body unchanged; frontmatter added by the warehouse)
disable-model-invocation: true
---
# Validation

## Purpose

Validate supplied skill outputs or planning artifacts for structure, traceability, internal consistency, and evidence without changing them.

## When To Use

Invoke after one or more skills produce outputs, or when a structured design package must be checked before downstream use.

## When NOT To Use

Do not re-perform requirement analysis, decide unresolved product questions, implement fixes, run builds/tests, or certify compliance.

## Inputs

```json
{
  "artifacts": [{"artifactId": "string", "skill": "string", "content": {}, "source": "string"}],
  "requirements": [{"requirementId": "string", "source": "string", "evidence": "string"}],
  "validationProfile": {"requiredFields": ["string"], "allowedSkillVersions": ["string"]}
}
```

`artifacts` is required. Other fields are optional.

## Outputs

```json
{
  "skill": "validation",
  "version": "1.0.0",
  "status": "complete|partial|failed",
  "result": "pass|pass-with-findings|fail|not-assessable",
  "checks": [{"checkId": "string", "category": "schema|traceability|consistency|evidence|scope|security", "result": "pass|fail|unknown", "artifactIds": ["string"], "requirementIds": ["string"], "source": "string", "evidence": "string", "confidence": 0.0, "detail": "string"}],
  "openQuestions": [],
  "assumptions": [],
  "conflicts": []
}
```

## Responsibilities

- Check output shape, required fields, stable IDs, and references among supplied artifacts.
- Check requirement traceability and whether decisions cite evidence.
- Identify contradictions, unsupported claims, scope violations, and unknowns.
- Report findings only; do not repair or rewrite artifacts.

## Processing Rules

- Validate against supplied schemas/profiles; if none are supplied, check each skill's declared output shape and common traceability fields.
- Distinguish a missing artifact from an invalid artifact.
- Check internal consistency only where both compared facts are present.
- Treat unresolved questions as legitimate unresolved outputs, not automatic failures.

## Decision Rules

- `pass` means all applicable checks pass; `pass-with-findings` means nonblocking issues exist; `fail` means a critical structural/traceability/scope defect prevents safe downstream use; `not-assessable` means inputs are insufficient.
- Use `unknown` for checks requiring unavailable policy or evidence.
- Do not assess factual truth beyond supplied evidence.

## Constraints

- Read-only; do not modify artifacts, invoke other skills implicitly, run code, install tools, or access production systems.
- No generated production code or compliance/security certification.

## Required Context

Structured artifacts and, for traceability checks, the normalized requirement registry or requirement IDs. A validation profile is optional.

## Skill Dependencies

Can validate any skill output independently. Usually runs after all selected core and brownfield skills; no upstream skill is mandatory.

## Tool Requirements

Read-only parsing and comparison of supplied artifacts. No repository writes or execution.

## Traceability

Each validation check includes artifact source, evidence, confidence, and applicable requirement IDs. Structural checks not tied to a requirement use an empty ID list.

## Uncertainty Handling

Mark non-verifiable checks unknown, explain missing context, and create open questions. Never interpret absence as proof of correctness.

## Failure Conditions

- No artifacts are provided.
- Artifacts cannot be parsed sufficiently to run any meaningful check.

## Validation

Validate this report itself: every check has a known artifact reference, valid outcome, evidence/source, and requirement IDs where applicable; overall result follows the decision rules.

## Example Input

```json
{"artifacts":[{"artifactId":"design-1","skill":"api-design","content":{"contracts":[{"contractId":"API-001","requirementIds":["REQ-404"]}]},"source":"api-output.json"}],"requirements":[{"requirementId":"REQ-001","source":"ticket:1","evidence":"“User can export.”"}]}
```

## Example Output

```json
{
  "skill":"validation","version":"1.0.0","status":"complete","result":"fail",
  "checks":[{"checkId":"CHK-001","category":"traceability","result":"fail","artifactIds":["design-1"],"requirementIds":["REQ-404"],"source":"api-output.json","evidence":"API-001 references REQ-404, which is absent from supplied requirements.","confidence":1.0,"detail":"Unresolved requirement reference; downstream traceability is unsafe."}],
  "openQuestions":[],"assumptions":[],"conflicts":[]
}
```

## Common Mistakes

- Treating unknown information as a pass.
- Silently fixing artifacts during validation.
- Re-judging business meaning without authority.
- Reporting unsupported factual claims as errors or successes.

## Security Considerations

Do not execute artifact content. Redact secrets from validation evidence and identify only the affected location.

## Token Optimization

Report failed and unknown checks first; summarize passing checks by category and avoid repeating full artifacts.

## Version

1.0.0
