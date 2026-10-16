---
name: adapter-compatibility-analysis
description: Use when, in brownfield work, a proposed change may affect existing consumers, contracts, persisted data, frameworks, dependencies or configuration. Do not use for greenfield work, to run compatibility tests, guarantee backward compatibility, make migration decisions or assume undocumented consumers exist or not.
license: Proprietary
metadata:
  version: 1.0.0
  owner: dotnet-adapter-team
  agent: dotnet-adapter
  contract: adapter-compatibility-analysis/1
  source: AdapterAgent skills library v1.0.0 (body unchanged; frontmatter added by the warehouse)
disable-model-invocation: true
---
# Compatibility Analysis

## Purpose

Assess evidence-based compatibility risks between proposed changes and existing .NET repository contracts, frameworks, dependencies, data, and consumers.

## When To Use

Invoke for a brownfield change when existing consumers, supported frameworks, contracts, or persisted data may be affected.

## When NOT To Use

Do not run compatibility tests, guarantee backward compatibility, make migration decisions, or infer consumers/support policies not represented in inputs.

## Inputs

```json
{
  "requirements": [{"requirementId": "string", "statement": "string", "source": "string", "evidence": "string"}],
  "repositoryAnalysis": {},
  "brownfieldMapping": {},
  "proposedDesign": {},
  "apiDesign": {},
  "dataModeling": {},
  "projectDetection": {},
  "dependencyMapping": {},
  "compatibilityPolicy": [{"source": "string", "content": "string"}]
}
```

At least `repositoryAnalysis` and one proposed change/design artifact are required.

## Outputs

```json
{
  "skill": "compatibility-analysis",
  "version": "1.0.0",
  "status": "complete|partial|failed",
  "assessments": [{"assessmentId": "string", "area": "api|data|framework|dependency|configuration|consumer|other", "outcome": "compatible|risk|incompatible|unknown", "requirementIds": ["string"], "source": "string", "evidence": "string", "confidence": 0.0, "detail": "string"}],
  "risks": [{"riskId": "string", "severity": "critical|high|medium|low|unknown", "requirementIds": ["string"], "source": "string", "evidence": "string", "confidence": 0.0, "detail": "string", "openQuestionIds": ["string"]}],
  "openQuestions": [],
  "assumptions": [],
  "conflicts": []
}
```

## Responsibilities

- Compare proposed changes with evidenced existing contracts, target frameworks, dependencies, configurations, and data models.
- Identify potential breaking changes and missing consumer/migration evidence.
- Apply only supplied compatibility policies and support commitments.
- Record unknowns separately from incompatibilities.

## Processing Rules

- Compare like-for-like facts; cite both the existing and proposed sources for each assessment.
- A risk indicates plausible incompatibility requiring review; `incompatible` requires direct conflicting evidence or explicit policy violation.
- Do not infer that an undocumented consumer exists or that none exists.
- Do not assign severity without a supplied scale; otherwise use `unknown`.

## Decision Rules

- `compatible` is limited to the assessed dimension and available evidence; it is not a system-wide guarantee.
- Use `unknown` when consumers, effective framework versions, policy, or migration behavior are not supplied.
- Treat additive contract or schema changes as unknown for compatibility unless consumer and evolution rules establish the result.

## Constraints

- No repository changes, code generation, tests, migration scripts, production access, or blanket compatibility guarantees.
- Keep this skill separate from brownfield mapping: compare compatibility, do not merely map current behavior.

## Required Context

Repository facts plus a proposed change artifact. Consumer inventory, support policy, API/data evolution rules, and explicit requirement IDs are highly relevant.

## Skill Dependencies

Requires repository-analysis and a proposed design/change; brownfield-mapping, dotnet-project-detection, dependency-mapping, api-design, and data-modeling are useful inputs. Informs testing-strategy and validation.

## Tool Requirements

Read-only comparison of supplied artifacts and relevant repository descriptors. No execution, network, or writes.

## Traceability

Every assessment and risk references requirement IDs where applicable, both evidence sources, and confidence. Repository-only findings may use an empty requirement ID list.

## Uncertainty Handling

If target consumers, compatibility policy, or current values are unknown, report unknown with an open question; never assume semantic-versioning rules or migration defaults.

## Failure Conditions

- No verifiable existing repository baseline.
- No proposed change/design to compare.
- Sources are too ambiguous to distinguish current and proposed states.

## Validation

Ensure every assessment has evidence on both sides, outcomes follow the decision rules, risk severity is policy-based or unknown, and question references resolve.

## Example Input

```json
{"repositoryAnalysis":{"repositoryFacts":[{"factId":"RF-1","fact":"Existing API returns invoiceId and total.","source":"contracts/invoice-response.json","evidence":"properties: invoiceId, total","confidence":0.99}]},"proposedDesign":{"source":"api-design-output","evidence":"Response lists invoiceId and downloadUrl; consumer inventory not supplied.","requirementIds":["REQ-001"]}}
```

## Example Output

```json
{
  "skill":"compatibility-analysis","version":"1.0.0","status":"partial",
  "assessments":[{"assessmentId":"COMPAT-001","area":"api","outcome":"unknown","requirementIds":["REQ-001"],"source":"contracts/invoice-response.json; api-design-output","evidence":"Existing response has invoiceId and total; proposed response lists invoiceId and downloadUrl. Consumer inventory/evolution rules are absent.","confidence":0.98,"detail":"The proposed shape may affect consumers; additive/removal behavior cannot be determined."}],
  "risks":[{"riskId":"RISK-001","severity":"unknown","requirementIds":["REQ-001"],"source":"contracts/invoice-response.json; api-design-output","evidence":"Existing and proposed response fields differ; no compatibility policy supplied.","confidence":0.91,"detail":"Potential response-contract compatibility risk.","openQuestionIds":["Q-001"]}],
  "openQuestions":[{"questionId":"Q-001","question":"Which consumers depend on total, and what API evolution policy applies?","affectedRequirementIds":["REQ-001"]}],"assumptions":[],"conflicts":[]
}
```

## Common Mistakes

- Declaring a change backward-compatible merely because it is additive.
- Assuming no consumers exist when none are listed.
- Labeling uncertainty as incompatibility.
- Assigning severity without a policy basis.

## Security Considerations

Do not expose secrets, customer data, or internal endpoints in compatibility evidence. Use safe identifiers and read-only analysis.

## Token Optimization

Assess only dimensions affected by proposed changes. Cite concise paired evidence and deduplicate shared risks.

## Version

1.0.0
