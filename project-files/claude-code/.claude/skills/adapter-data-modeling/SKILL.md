---
name: adapter-data-modeling
description: Use when requirements imply creating, retrieving, retaining, exchanging or transforming data, to identify conceptual entities, attributes, relationships and lifecycle rules. Do not use to design API transport models, choose a database product, add conventional fields that no requirement states, or generate entities or migrations.
license: Proprietary
metadata:
  version: 1.0.0
  owner: dotnet-adapter-team
  agent: dotnet-adapter
  contract: adapter-data-modeling/1
  source: AdapterAgent skills library v1.0.0 (body unchanged; frontmatter added by the warehouse)
disable-model-invocation: true
---
# Data Modeling

## Purpose

Describe requirement-driven conceptual data entities, attributes, relationships, and lifecycle constraints without choosing an unsubstantiated database design.

## When To Use

Invoke when requirements imply creation, retrieval, retention, exchange, or transformation of data.

## When NOT To Use

Do not design API transport schemas, map every existing database object, choose a database product, or generate entity classes or migrations.

## Inputs

```json
{
  "requirements": [{"requirementId": "string", "statement": "string", "source": "string", "evidence": "string"}],
  "requirementAnalysis": {},
  "architectureMapping": {},
  "repositoryAnalysis": {},
  "existingDataModels": [{"path": "string", "content": "string"}],
  "dataPolicies": [{"source": "string", "content": "string"}]
}
```

`requirements` is required; other fields are optional.

## Outputs

```json
{
  "skill": "data-modeling",
  "version": "1.0.0",
  "status": "complete|partial|failed",
  "entities": [{"entityId": "string", "name": "string", "purpose": "string|unknown", "attributes": [{"name": "string", "meaning": "string|unknown", "type": "string|unknown", "required": "yes|no|unknown"}], "requirementIds": ["string"], "source": "string", "evidence": "string", "confidence": 0.0}],
  "relationships": [{"from": "string", "to": "string", "cardinality": "string|unknown", "requirementIds": ["string"], "source": "string", "evidence": "string", "confidence": 0.0}],
  "lifecycleRules": [{"rule": "string|unknown", "requirementIds": ["string"], "source": "string", "evidence": "string", "confidence": 0.0}],
  "openQuestions": [],
  "assumptions": [],
  "conflicts": []
}
```

## Responsibilities

- Identify conceptual data needed to satisfy requirements.
- Record evidenced attributes, cardinality, ownership, and lifecycle/retention rules.
- Distinguish existing models from proposed conceptual models.
- Surface data classification and policy questions for security-mapping.

## Processing Rules

- Do not infer attributes, keys, types, or cardinality from a familiar domain object name.
- Reuse existing model facts only when source files are supplied and readable.
- Separate conceptual meaning from storage type and physical schema.
- Preserve retention, deletion, audit, and ownership statements exactly in intent.

## Decision Rules

- Add an attribute only if a requirement or existing model supports it.
- Set type/cardinality/requiredness to `unknown` when evidence does not decide them.
- Treat storage technology and schema migration choices as out of scope.

## Constraints

- No production code, migrations, database product/version claims, or repository modifications.
- Do not include real personal or secret data in examples or output.

## Required Context

Data-related requirements and provenance. Existing model files and organization data policies are optional inputs.

## Skill Dependencies

Preferred predecessors: requirement-analysis and architecture-mapping; repository-analysis is useful for brownfield models. Informs api-design, dependency-mapping, security-mapping, and testing-strategy.

## Tool Requirements

Read-only access to requirement artifacts, model descriptors, and policy documents.

## Traceability

Every entity, attribute, relationship, lifecycle rule, and question references requirement IDs, source, evidence, and confidence. Existing-only facts may have an empty requirement ID list.

## Uncertainty Handling

Use `unknown` for unsupported details and raise focused questions. Do not infer standard identifiers, timestamps, or audit fields.

## Failure Conditions

- No usable requirements or existing model evidence is supplied.
- Inputs cannot be attributed to sources.

## Validation

Check unique entity IDs, resolvable relationship endpoints, known requirement references, and evidence for every non-unknown model decision.

## Example Input

```json
{"requirements":[{"requirementId":"REQ-031","statement":"Exports are retained for 30 days.","source":"spec:exports","evidence":"“Exports are retained for 30 days.”"}]}
```

## Example Output

```json
{
  "skill":"data-modeling","version":"1.0.0","status":"partial",
  "entities":[{"entityId":"DATA-001","name":"Export artifact","purpose":"Retained exported output","attributes":[{"name":"retention duration","meaning":"Time the export is retained","type":"unknown","required":"unknown"}],"requirementIds":["REQ-031"],"source":"spec:exports","evidence":"“Exports are retained for 30 days.”","confidence":0.9}],
  "relationships":[],"lifecycleRules":[{"rule":"Retain for 30 days; start event and deletion behavior unknown.","requirementIds":["REQ-031"],"source":"spec:exports","evidence":"“Exports are retained for 30 days.”","confidence":0.98}],
  "openQuestions":[{"questionId":"Q-001","question":"What event starts retention, and what should happen after 30 days?","affectedRequirementIds":["REQ-031"]}],"assumptions":[],"conflicts":[]
}
```

## Common Mistakes

- Adding conventional audit fields without evidence.
- Confusing API DTOs with persisted entities.
- Selecting SQL/NoSQL or physical types prematurely.
- Treating retention wording as a complete deletion policy.

## Security Considerations

Minimize sensitive data, flag unknown classification/access/retention controls, and never reproduce real records or credentials.

## Token Optimization

Include only requirement-relevant entities and attributes; cite source paths or excerpts rather than repeating model files.

## Version

1.0.0
