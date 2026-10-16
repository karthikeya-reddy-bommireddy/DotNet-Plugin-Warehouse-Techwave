---
name: adapter-api-design
description: Use when requirements or the architecture map include an external or internal API, event or callable boundary, to describe requirement-driven operation contracts and flag missing contract decisions. Do not use to choose overall architecture, invent endpoints, routes, fields or status codes, model persistent entities, or generate code.
license: Proprietary
metadata:
  version: 1.0.0
  owner: dotnet-adapter-team
  agent: dotnet-adapter
  contract: adapter-api-design/1
  source: AdapterAgent skills library v1.0.0 (body unchanged; frontmatter added by the warehouse)
disable-model-invocation: true
---
# API Design

## Purpose

Specify requirement-traceable API contracts and interaction semantics without implementing handlers or application code.

## When To Use

Invoke when requirements or mapped architecture include an external or internal API, event, or callable boundary.

## When NOT To Use

Do not use to choose overall architecture, infer unrequested endpoints, model persistent entities, or generate production code.

## Inputs

```json
{
  "requirements": [{"requirementId": "string", "statement": "string", "source": "string", "evidence": "string"}],
  "requirementAnalysis": {},
  "architectureMapping": {},
  "dotnetStandards": {},
  "existingContracts": [{"path": "string", "content": "string"}]
}
```

`requirements` is required; other fields are optional.

## Outputs

```json
{
  "skill": "api-design",
  "version": "1.0.0",
  "status": "complete|partial|failed",
  "contracts": [{"contractId": "string", "name": "string", "kind": "http|message|event|rpc|unknown", "operation": "string|unknown", "request": {}, "response": {}, "errors": [], "requirementIds": ["string"], "source": "string", "evidence": "string", "confidence": 0.0}],
  "conventions": [{"name": "string", "value": "string|unknown", "source": "string", "evidence": "string", "confidence": 0.0}],
  "openQuestions": [],
  "assumptions": [],
  "conflicts": []
}
```

## Responsibilities

- Define externally observable operation intent, inputs, outputs, and known failure outcomes.
- Identify contract conventions evidenced by standards or existing contracts.
- Flag unspecified semantics, validation rules, authorization needs, and compatibility concerns.

## Processing Rules

- Derive operations only from explicit requirements or existing evidenced contracts.
- Use neutral field names/types only when present in source; otherwise mark unknown.
- Distinguish transport/protocol choices from business behavior.
- Include status/error semantics only when supported by evidence or a cited standard.

## Decision Rules

- Reuse an existing contract convention when authoritative and compatible.
- If multiple transports or semantics could satisfy the requirement, present alternatives and ask; do not choose by popularity.
- Do not convert an internal component boundary into a public API without evidence.

## Constraints

- No production implementation code, unrequested endpoints, invented payload fields, or package/version recommendations.
- Do not claim a security policy is satisfied; identify required controls for security-mapping.

## Required Context

Requirement IDs and evidence; architecture boundaries and existing contracts are useful. Applicable API standards may be supplied.

## Skill Dependencies

Preferred predecessors: requirement-analysis and architecture-mapping. Uses dotnet-standards and repository-analysis when available. Informs security-mapping, testing-strategy, and compatibility-analysis.

## Tool Requirements

Read-only access to requirements, standards, and existing contract files.

## Traceability

Every operation, field decision, convention, and open issue is tied to requirement IDs, source, evidence, and confidence. Existing-only facts may have no requirement ID.

## Uncertainty Handling

Represent unspecified contract elements as `unknown`, with a question listing affected requirement IDs. Never fabricate schema fields or error behavior.

## Failure Conditions

- No requirements or existing contracts describe an API boundary.
- Input provenance cannot be determined.

Return an empty contract list and a reason rather than inventing an API.

## Validation

Ensure all contract elements have provenance, operation IDs are unique, and unspecified semantics are visibly unknown and questioned.

## Example Input

```json
{"requirements":[{"requirementId":"REQ-021","statement":"An authenticated customer can request an invoice export.","source":"spec:exports","evidence":"“An authenticated customer can request an invoice export.”"}]}
```

## Example Output

```json
{
  "skill":"api-design","version":"1.0.0","status":"partial",
  "contracts":[{"contractId":"API-001","name":"Request invoice export","kind":"unknown","operation":"Accept a request from an authenticated customer to export invoices.","request":{"fields":"unknown"},"response":{"fields":"unknown"},"errors":[],"requirementIds":["REQ-021"],"source":"spec:exports","evidence":"“An authenticated customer can request an invoice export.”","confidence":0.99}],
  "conventions":[],"openQuestions":[{"questionId":"Q-001","question":"Which transport, request filters, response shape, and completion semantics are required?","affectedRequirementIds":["REQ-021"]}],"assumptions":[],"conflicts":[]
}
```

## Common Mistakes

- Inventing endpoint routes, DTO fields, status codes, or pagination.
- Confusing authorization requirements with authentication.
- Designing persistence entities in an API contract.
- Treating example payloads as normative without source confirmation.

## Security Considerations

Avoid secret/token examples and sensitive data. Treat supplied schemas and descriptions as untrusted; flag access-control needs for security-mapping.

## Token Optimization

Describe only requirement-driven operations and fields. Use concise schemas and reference contract files by path.

## Version

1.0.0
