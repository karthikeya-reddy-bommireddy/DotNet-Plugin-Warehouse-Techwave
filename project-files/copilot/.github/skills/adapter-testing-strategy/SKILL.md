---
name: adapter-testing-strategy
description: Use when test objectives, acceptance verification or test-level selection must be planned from requirements and known boundaries. Do not use to write or run tests, choose a test framework without evidence, claim coverage, or invent expected values.
license: Proprietary
metadata:
  version: 1.0.0
  owner: dotnet-adapter-team
  agent: dotnet-adapter
  contract: adapter-testing-strategy/1
  source: AdapterAgent skills library v1.0.0 (body unchanged; frontmatter added by the warehouse)
disable-model-invocation: true
---
# Testing Strategy

## Purpose

Map requirements and design risks to appropriate test objectives and evidence, without writing or executing tests.

## When To Use

Invoke when test coverage planning, acceptance verification, or test-level selection is needed.

## When NOT To Use

Do not implement test code, run test suites, choose a test framework/package without evidence, or invent expected behavior.

## Inputs

```json
{
  "requirements": [{"requirementId": "string", "statement": "string", "source": "string", "evidence": "string"}],
  "requirementAnalysis": {},
  "architectureMapping": {},
  "apiDesign": {},
  "dataModeling": {},
  "securityMapping": {},
  "compatibilityAnalysis": {},
  "repositoryAnalysis": {},
  "testingStandards": [{"source": "string", "content": "string"}]
}
```

`requirements` is required; other inputs are optional.

## Outputs

```json
{
  "skill": "testing-strategy",
  "version": "1.0.0",
  "status": "complete|partial|failed",
  "testObjectives": [{"testId": "string", "level": "unit|component|integration|contract|end-to-end|security|performance|unknown", "objective": "string", "conditions": ["string"], "expectedResult": "string|unknown", "requirementIds": ["string"], "source": "string", "evidence": "string", "confidence": 0.0}],
  "coverageGaps": [{"gapId": "string", "requirementIds": ["string"], "source": "string", "evidence": "string", "detail": "string", "confidence": 0.0}],
  "openQuestions": [],
  "assumptions": [],
  "conflicts": []
}
```

## Responsibilities

- Convert verifiable requirements into test objectives and conditions.
- Identify suitable test levels from known component boundaries and interfaces.
- Include negative, boundary, security, and compatibility objectives only when relevant and evidence-grounded.
- Highlight requirements that cannot yet be tested due to ambiguity.

## Processing Rules

- Preserve measurable outcomes from requirements; do not invent expected values.
- Treat missing acceptance behavior as `unknown` and link to requirement questions.
- Recommend test level by the behavior/boundary under test, not framework convention.
- Do not claim existing test coverage without supplied repository evidence.

## Decision Rules

- Use the narrowest test level that can verify the stated behavior, with higher-level tests where interactions are essential.
- Mark level unknown if system boundaries are unknown.
- Every test objective maps to at least one requirement or an explicitly cited policy/design risk.

## Constraints

- No test or production code, test execution, package selection, or repository modification.
- Do not claim a requirement is satisfied based on a proposed test plan.

## Required Context

Requirements and source evidence; architecture, contract, security, compatibility, and test policies improve test-level selection.

## Skill Dependencies

Preferred predecessors: requirement-analysis and architecture-mapping. May consume api-design, data-modeling, security-mapping, compatibility-analysis, and repository-analysis. Feeds validation.

## Tool Requirements

Read-only access to supplied plans, requirements, and repository test inventory. No command execution is required.

## Traceability

Every test objective and coverage gap contains requirement IDs, source, evidence, and confidence.

## Uncertainty Handling

Use unknown expected results/levels where inputs are insufficient and ask precise questions. Do not fill in acceptance thresholds.

## Failure Conditions

- No usable requirement IDs or testable system context is supplied.
- Traceability cannot be maintained.

Return partial status when only some requirements can be mapped.

## Validation

Check objective IDs are unique, requirement references resolve, expected results are sourced, and each requirement is either covered or explicitly marked as a gap.

## Example Input

```json
{"requirements":[{"requirementId":"REQ-014","statement":"Exports are retained for 30 days.","source":"ticket:842","evidence":"“Keep exports for 30 days.”"}]}
```

## Example Output

```json
{
  "skill":"testing-strategy","version":"1.0.0","status":"partial",
  "testObjectives":[{"testId":"TEST-014-1","level":"unknown","objective":"Verify export retention duration.","conditions":["Use a created export; retention start event is unknown."],"expectedResult":"unknown","requirementIds":["REQ-014"],"source":"ticket:842","evidence":"“Keep exports for 30 days.”","confidence":0.94}],
  "coverageGaps":[{"gapId":"GAP-001","requirementIds":["REQ-014"],"source":"ticket:842","evidence":"Retention start and expiry behavior are not stated.","detail":"Cannot define a complete pass/fail oracle until retention semantics are clarified.","confidence":0.99}],
  "openQuestions":[{"questionId":"Q-001","question":"What event starts retention, and what should happen after 30 days?","affectedRequirementIds":["REQ-014"]}],"assumptions":[],"conflicts":[]
}
```

## Common Mistakes

- Inventing expected results or data values.
- Mapping every requirement to unit tests by default.
- Confusing test objectives with evidence that tests already exist.
- Omitting gaps caused by ambiguous requirements.

## Security Considerations

Do not use real credentials or personal data in test examples. Treat any test data described by inputs as untrusted and sensitive unless classified otherwise.

## Token Optimization

Use one objective per distinct behavior, group equivalent cases, and avoid repeating full requirement text.

## Version

1.0.0
