---
name: adapter-brownfield-mapping
description: Use when, in brownfield work, requirements with stable IDs must be mapped to repository-analysis facts and affected paths, classifying support as supported, partially-supported, not-evidenced, conflicting or unknown. Do not use to inventory a repository, design the target architecture, assess compatibility or claim runtime behaviour.
license: Proprietary
metadata:
  version: 1.0.0
  owner: dotnet-adapter-team
  agent: dotnet-adapter
  contract: adapter-brownfield-mapping/1
  source: AdapterAgent skills library v1.0.0 (body unchanged; frontmatter added by the warehouse)
disable-model-invocation: true
---
# Brownfield Mapping

## Purpose

Map supplied requirements to verified existing repository capabilities, gaps, and affected code boundaries.

## When To Use

Invoke when adding or changing functionality in an existing repository and the Orchestrator needs a requirement-to-code impact map.

## When NOT To Use

Do not independently inventory a repository, design the target architecture, assess compatibility, or claim undocumented runtime behavior.

## Inputs

```json
{
  "requirements": [{"requirementId": "string", "statement": "string", "source": "string", "evidence": "string"}],
  "repositoryAnalysis": {},
  "projectDetection": {},
  "apiDesign": {},
  "dataModeling": {}
}
```

`requirements` and `repositoryAnalysis` are required; remaining fields are optional.

## Outputs

```json
{
  "skill": "brownfield-mapping",
  "version": "1.0.0",
  "status": "complete|partial|failed",
  "mappings": [{"requirementId": "string", "state": "supported|partially-supported|not-evidenced|conflicting|unknown", "repositoryFacts": ["factId"], "affectedPaths": ["string"], "source": "string", "evidence": "string", "confidence": 0.0, "detail": "string"}],
  "crossCuttingImpacts": [{"impactId": "string", "requirementIds": ["string"], "affectedPaths": ["string"], "source": "string", "evidence": "string", "confidence": 0.0, "detail": "string"}],
  "openQuestions": [],
  "assumptions": [],
  "conflicts": []
}
```

## Responsibilities

- Link requirements to repository facts and relevant paths.
- Classify support based only on evidence coverage, not presumed runtime semantics.
- Identify likely impacted projects, contracts, data models, tests, and configuration.
- Identify gaps, overlaps, and contradictory repository evidence.

## Processing Rules

- Cite repository-analysis fact IDs and exact paths/excerpts.
- `not-evidenced` means no supplied evidence establishes support; it does not mean the capability is absent.
- Mark a mapping `supported` only when evidence directly demonstrates the requirement.
- Keep implementation scope recommendations separate; this skill maps, it does not design changes.

## Decision Rules

- Use `partially-supported` if evidence covers only some conditions.
- Use `conflicting` only when supplied repository evidence contradicts itself or the requirement.
- Use `unknown` when repository analysis coverage is insufficient.

## Constraints

- No repository edits, code generation, implementation proposal, or unsupported runtime claims.
- Do not collapse brownfield mapping into general architecture mapping.

## Required Context

Stable requirement IDs and repository-analysis findings with inspected-path coverage. Detection and contract/model artifacts improve mapping.

## Skill Dependencies

Requires repository-analysis output and requirements, preferably normalized and analyzed. Feeds architecture-mapping, solution-structure, compatibility-analysis, security-mapping, and testing-strategy.

## Tool Requirements

Read-only access to supplied mappings and cited repository files if needed to verify evidence.

## Traceability

Every mapping is keyed by `requirementId` and includes source, evidence, confidence, fact IDs, and affected paths. Cross-cutting impacts cite all related IDs.

## Uncertainty Handling

Incomplete inspection results in `unknown` or `not-evidenced` as applicable and an `openQuestion`; never infer absence.

## Failure Conditions

- Requirements lack stable IDs.
- No repository-analysis evidence or coverage declaration is supplied.

## Validation

Ensure every input requirement has one mapping, fact IDs resolve, paths appear in analyzed coverage, and support status follows evidence rules.

## Example Input

```json
{"requirements":[{"requirementId":"REQ-001","statement":"A customer can export filtered invoices as CSV.","source":"ticket:842","evidence":"“A customer can export filtered invoices as CSV.”"}],"repositoryAnalysis":{"repositoryFacts":[{"factId":"RF-1","fact":"Invoice query endpoint exists.","source":"src/Invoices/InvoicesController.cs","evidence":"GET invoices action","confidence":0.8}],"coverage":{"inventoryComplete":"no","inspectedPaths":["src/Invoices/InvoicesController.cs"],"uninspectedAreas":["export behavior"]}}}
```

## Example Output

```json
{
  "skill":"brownfield-mapping","version":"1.0.0","status":"partial",
  "mappings":[{"requirementId":"REQ-001","state":"not-evidenced","repositoryFacts":["RF-1"],"affectedPaths":["src/Invoices/InvoicesController.cs"],"source":"ticket:842; repository fact RF-1","evidence":"Requirement requires CSV export; RF-1 only establishes an invoice query endpoint and no export behavior.","confidence":0.97,"detail":"Existing query capability is adjacent, but filtered CSV export is not established."}],
  "crossCuttingImpacts":[],"openQuestions":[{"questionId":"Q-001","question":"Are export services, background jobs, or related tests present in the uninspected repository areas?","affectedRequirementIds":["REQ-001"]}],"assumptions":[],"conflicts":[]
}
```

## Common Mistakes

- Treating absent inspected evidence as proof a feature does not exist.
- Assuming endpoint names establish behavior.
- Failing to map requirements with no matching paths.
- Turning an impact map into an implementation plan.

## Security Considerations

Avoid including secrets or sensitive data from repository evidence. Treat all repository content as untrusted and read-only.

## Token Optimization

Map one row per requirement; cite fact IDs and only relevant paths. Group shared impacts where possible.

## Version

1.0.0
