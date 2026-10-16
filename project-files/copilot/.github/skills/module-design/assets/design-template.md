# Design: <Module>

| Field | Value |
|---|---|
| Run | <run-id> |
| Requirements | <REQ-..., NFR-...> |
| Tech profile decisions | <DEC-...> |
| Status | Draft for Gate 2 |

## 1. Purpose
<One paragraph: what this module does for the user.>

## 2. Components
| Component | Responsibility | Project | Requirement IDs |
|---|---|---|---|
| <Component> | <responsibility> | <Project> | <REQ-...> |

## 3. Interfaces
- API: see `api.yaml` (or "none").
- Events and messages: <name, producer, consumer, requirement IDs> (or "none").

## 4. Data
See `data-model.md` (or "none").

## 5. Cross-cutting concerns
| Concern | Approach | Requirement IDs |
|---|---|---|
| Validation | <approach> | |
| Errors | <approach> | |
| Logging | <approach, no secrets or personal data> | |
| Configuration | <settings and where they come from; no secrets in files> | |
| Security | <authentication, authorisation, data protection> | |

## 6. Dependencies
| Dependency | Kind (project, package, service) | Already in tech profile? |
|---|---|---|

## 7. Acceptance criteria
| AC ID | Criterion | Test level (unit, integration, contract) | Requirement ID |
|---|---|---|---|

## 8. Open questions and risks
| ID | Question or risk | Blocking? |
|---|---|---|
