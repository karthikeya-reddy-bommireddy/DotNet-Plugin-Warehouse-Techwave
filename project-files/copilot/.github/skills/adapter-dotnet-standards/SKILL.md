---
name: adapter-dotnet-standards
description: Use when organization or repository .NET standards (framework targets, language, build, style, test or dependency rules) must constrain planning. Do not use to decide an approved .NET version without an authoritative source, impose generic preferences as policy, edit configuration or certify compliance.
license: Proprietary
metadata:
  version: 1.0.0
  owner: dotnet-adapter-team
  agent: dotnet-adapter
  contract: adapter-dotnet-standards/1
  source: AdapterAgent skills library v1.0.0 (body unchanged; frontmatter added by the warehouse)
disable-model-invocation: true
---
# .NET Standards

## Purpose

Map applicable .NET engineering standards to the supplied project facts and identify compliance gaps or unknowns.

## When To Use

Invoke when project-specific .NET conventions, framework targets, coding/build policy, or organization standards must be represented for planning.

## When NOT To Use

Do not infer a target framework, impose generic preferences as policy, edit configuration, or generate implementation code.

## Inputs

```json
{
  "projectDetection": {},
  "solutionStructure": {},
  "dependencyMapping": {},
  "organizationStandards": [{"standardId": "string", "source": "string", "content": "string"}],
  "repositoryStandards": [{"path": "string", "content": "string"}]
}
```

At least one input source (detected project or standards document) is required.

## Outputs

```json
{
  "skill": "dotnet-standards",
  "version": "1.0.0",
  "status": "complete|partial|failed",
  "applicableStandards": [{"standardId": "string", "area": "framework|language|build|style|testing|dependency|other", "rule": "string", "applicability": "applicable|not-applicable|unknown", "requirementIds": ["string"], "source": "string", "evidence": "string", "confidence": 0.0}],
  "gaps": [{"gapId": "string", "standardId": "string", "detail": "string", "requirementIds": ["string"], "source": "string", "evidence": "string", "confidence": 0.0}],
  "openQuestions": [],
  "assumptions": [],
  "conflicts": []
}
```

## Responsibilities

- Extract and classify explicit organization and repository standards.
- Assess applicability only against supplied project facts.
- Identify contradictions and missing evidence without silently prioritizing standards.
- Report standard version/authority only when supplied.

## Processing Rules

- Preserve normative wording and source identifiers.
- Treat repository configuration as observed practice, not automatically authoritative policy.
- Do not use external knowledge to fill an absent .NET support policy.
- Distinguish a detected project value from a required standard.

## Decision Rules

- Mark applicability `unknown` when target framework, project kind, or policy scope is unavailable.
- A gap requires evidence that an applicable rule is unmet; absence of evidence is an open question.
- Conflicting rules remain conflicts until an authorized precedence rule is provided.

## Constraints

- No framework/version recommendations without supplied authority.
- No package installation, build, configuration edit, or production code.

## Required Context

Relevant project detection data and authoritative organization or repository standards.

## Skill Dependencies

Uses dotnet-project-detection, solution-structure, and optionally dependency-mapping. Provides constraints to api-design, dependency-mapping, and validation.

## Tool Requirements

Read-only access to project descriptors and standards documents.

## Traceability

Every standard, applicability decision, and gap has source/evidence/confidence and requirement IDs when requirement-driven.

## Uncertainty Handling

If policy authority or project applicability is unclear, set applicability to `unknown` and ask for the missing policy or project fact.

## Failure Conditions

- Neither usable project data nor a standards source is supplied.
- No source attribution is possible.

## Validation

Check that every claimed standard quotes or cites its source, applicability matches evidence, and missing proof is not reported as noncompliance.

## Example Input

```json
{"projectDetection":{"projects":[{"path":"src/Billing/Billing.csproj","targetFrameworks":["net8.0"]}]},"organizationStandards":[{"standardId":"STD-DOTNET-01","source":"engineering-handbook:dotnet","content":"Services must target the approved LTS framework."}]}
```

## Example Output

```json
{
  "skill":"dotnet-standards","version":"1.0.0","status":"partial",
  "applicableStandards":[{"standardId":"STD-DOTNET-01","area":"framework","rule":"Services must target the approved LTS framework.","applicability":"unknown","requirementIds":[],"source":"engineering-handbook:dotnet","evidence":"“approved LTS framework”","confidence":0.98}],
  "gaps":[],"openQuestions":[{"questionId":"Q-001","question":"Which framework is currently approved as the LTS target, and is Billing classified as a service?","affectedRequirementIds":[]}],"assumptions":[],"conflicts":[]
}
```

## Common Mistakes

- Treating a detected target framework as approved policy.
- Assuming current .NET support status without a dated source.
- Claiming a gap because the evidence set is incomplete.
- Ignoring precedence conflicts.

## Security Considerations

Standards and configuration may contain sensitive internal details. Minimize disclosure and do not expose credentials.

## Token Optimization

Return only applicable or unresolved rules and cite concise excerpts/locators.

## Version

1.0.0
