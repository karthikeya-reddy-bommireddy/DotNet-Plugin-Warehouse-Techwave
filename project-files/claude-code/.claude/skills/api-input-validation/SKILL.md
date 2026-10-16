---
name: api-input-validation
description: Input validation, model binding and mapping safety for the ASP.NET Core .NET 10 APIs repo - Data Annotations on request DTOs, [ApiController] automatic 400s, enum/Guid/number binding pitfalls, mass assignment and over-posting prevention, System.Text.Json hardening, request size limits, file uploads, and safe AutoMapper use (CVE-2026-32933 and v15+ licensing). Use whenever creating or changing a request or response DTO, an action's parameters, JSON options, a mapping profile or an upload endpoint, and whenever client data reaches code.
license: Proprietary
metadata:
  version: 1.7.1
  owner: api-layer-team
  agent: dotnet-coder
  contract: api-input-validation/1
  source: api-layer-engineer v1.7.1 (body unchanged; frontmatter added by the warehouse)
disable-model-invocation: true
---

# Input validation, binding and mapping

> **Placeholders:** example names and numbers are illustrations. Use project-facts or ask; never copy them into code.

## Rules (most important first)
1. **Every request field is constrained**: strings have a maximum length, numbers a range, collections a maximum count, enums are checked as defined, formats use a validator. Unconstrained input is a finding.
2. **No over-posting**: request DTOs contain only client-settable fields. `Id`, owner, tenant, role, status, price-like and audit fields are set server-side from claims or business logic.
3. **Keep `[ApiController]` automatic validation on** (never `SuppressModelStateInvalidFilter`): invalid input → 400 ValidationProblemDetails before the action runs.
4. **Enums and value types lie**: binding accepts any integer for an enum and a default value for a missing `int`/`Guid`. Use `[EnumDataType]`, nullable + `[Required]`, or a custom attribute.
5. **Regexes are anchored and time-limited** (`MatchTimeoutInMilliseconds`), or replaced by simpler checks.
6. **JSON is hardened**: modest `MaxDepth`, no type-name driven polymorphism, unknown members handled by team convention, framework exception messages hidden from clients.
7. **Size limits follow the endpoint's real needs** (`[RequestSizeLimit]`, `[Length]` on collections) on top of a small global default (api-pipeline-hardening).
8. **AutoMapper never writes client data into protected entity members**, mapping configuration is validated by a test, and the AutoMapper version is not affected by CVE-2026-32933.
9. **Business rules stay in Application/Domain** (stock, limits, state transitions). The API layer validates shape; it does not re-implement business rules.

## Where DTOs live
The spec puts DTOs, mapping profiles and validation logic in the **Application layer** (see application-use-cases). Follow where the repo already keeps them; if it has API-specific request contracts in the API project, keep that convention.

## Quick reference
| Field | Annotation |
|---|---|
| Required string | `[Required, StringLength(100, MinimumLength = 1)]` (+ `required` member) |
| Optional string | `[StringLength(500)]` |
| Constrained text | `[RegularExpression("^[A-Za-z0-9-]{1,64}$", MatchTimeoutInMilliseconds = 100)]` |
| Number that must be sent | `int?` + `[Required]` + `[Range(1, 1000)]` |
| Non-empty Guid | `Guid?` + `[Required]` + `[NotDefault]` (custom) |
| Enum | `[EnumDataType(typeof(T))]` |
| Fixed set of strings | `[AllowedValues("a", "b")]` |
| Collection | `[Length(1, 50)]`, and validate the element type too |
| Email | `[EmailAddress, StringLength(254)]` |
| Cross-field rule | `IValidatableObject` |

## References
- [data-annotations-cookbook.md](references/data-annotations-cookbook.md): DTO examples, custom attribute, `IValidatableObject`, binding pitfalls, validation tests.
- [automapper-safety.md](references/automapper-safety.md): CVE-2026-32933, licensing, safe profiles, configuration validation, options to present to the team.
- [json-and-uploads.md](references/json-and-uploads.md): global System.Text.Json options, injection reminders, secure file uploads.
