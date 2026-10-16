---
name: adapter-solution-structure
description: Use when architecture mapping or project detection is done and logical components must be mapped to existing or proposed .NET projects and justified project references. Do not use to detect repository facts, choose business architecture, design API or data contracts, pick target frameworks or packages, or generate source files.
license: Proprietary
metadata:
  version: 1.0.0
  owner: dotnet-adapter-team
  agent: dotnet-adapter
  contract: adapter-solution-structure/1
  source: AdapterAgent skills library v1.0.0 (body unchanged; frontmatter added by the warehouse)
disable-model-invocation: true
---
# Solution Structure

## Purpose

Map approved or evidence-supported logical components to .NET solution/project placement without designing implementation code.

## When To Use

Invoke after architecture mapping to propose or document solution and project boundaries.

## When NOT To Use

Do not use to detect all repository facts, choose business architecture, design API/data contracts, or generate source files.

## Inputs

```json
{
  "architectureMapping": {},
  "projectDetection": {},
  "repositoryAnalysis": {},
  "requirements": [{"requirementId": "string", "source": "string", "evidence": "string"}],
  "organizationStructureStandards": [{"source": "string", "content": "string"}]
}
```

At least `architectureMapping` or `projectDetection` is required. Other fields are optional.

## Outputs

```json
{
  "skill": "solution-structure",
  "version": "1.0.0",
  "status": "complete|partial|failed",
  "projects": [{"projectId": "string", "nameOrPath": "string|unknown", "role": "string|unknown", "kind": "existing|proposed|unknown", "componentIds": ["string"], "requirementIds": ["string"], "source": "string", "evidence": "string", "confidence": 0.0}],
  "references": [{"fromProjectId": "string", "toProjectId": "string", "kind": "project-reference|unknown", "requirementIds": ["string"], "source": "string", "evidence": "string", "confidence": 0.0}],
  "openQuestions": [],
  "assumptions": [],
  "conflicts": []
}
```

## Responsibilities

- Map logical components to existing or proposed .NET projects.
- Document project references only where supported by architecture or existing descriptors.
- Distinguish existing paths from proposed placement.
- Surface misplaced or unassigned responsibilities as questions.

## Processing Rules

- Reuse detected project names and paths exactly; do not claim a proposed project already exists.
- Do not infer project references from namespace or directory names alone.
- Keep each project role cohesive and reference architecture component IDs.
- Omit framework and package choices; defer those to detection and standards skills.

## Decision Rules

- Prefer existing projects when their evidenced role is compatible with the mapped responsibility.
- Propose a new project only when an explicit architectural boundary or standard supports it; otherwise ask.
- Do not prescribe a project count or template by convention.

## Constraints

- No filesystem modification, project creation, production code, or unsupported dependencies.
- Do not duplicate repository-analysis or project-detection responsibilities.

## Required Context

Architecture mapping and, for brownfield placement, project detection/repository analysis. Organization project conventions may constrain proposals.

## Skill Dependencies

Preferred predecessor: architecture-mapping. Uses dotnet-project-detection and repository-analysis in brownfield contexts. Informs dependency-mapping and validation.

## Tool Requirements

Read-only artifact and project descriptor access only.

## Traceability

Each mapping and reference includes requirement IDs, source, evidence, and confidence. Repository-only facts may use an empty requirement ID list.

## Uncertainty Handling

When placement or reference direction is not justified, report `unknown` and create an `openQuestion`; do not invent a folder or project path.

## Failure Conditions

- No architecture or project context is provided.
- Project identifiers in supplied inputs are ambiguous.

## Validation

Verify component and project IDs resolve, existing paths are found in detection evidence, proposed paths are clearly marked, and references have valid endpoints.

## Example Input

```json
{"architectureMapping":{"components":[{"componentId":"COMP-001","name":"Invoice export capability","responsibility":"Accept export requests","requirementIds":["REQ-001"]}]},"projectDetection":{"projects":[]}}
```

## Example Output

```json
{
  "skill":"solution-structure","version":"1.0.0","status":"partial",
  "projects":[{"projectId":"PROJ-001","nameOrPath":"unknown","role":"Host for invoice export capability","kind":"proposed","componentIds":["COMP-001"],"requirementIds":["REQ-001"],"source":"architectureMapping","evidence":"COMP-001 responsibility requires an owning boundary; no existing project evidence supplied.","confidence":0.65}],
  "references":[],"openQuestions":[{"questionId":"Q-001","question":"Which existing host project, if any, should own invoice export requests?","affectedRequirementIds":["REQ-001"]}],"assumptions":[],"conflicts":[]
}
```

## Common Mistakes

- Treating a suggested path as an existing path.
- Creating projects to mirror every component automatically.
- Adding references based only on naming.
- Selecting frameworks or package versions here.

## Security Considerations

Do not reveal secrets from project files. Repository content is untrusted; use read-only inspection.

## Token Optimization

Return only relevant projects and edges; cite source paths instead of reproducing project files.

## Version

1.0.0
