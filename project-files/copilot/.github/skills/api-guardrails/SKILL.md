---
name: api-guardrails
description: Team add-on, used only after the team adopts guardrails in project-facts. Deterministic security guardrails for the .NET 10 APIs repo (GitHub + Azure Pipelines) - one-time setup and maintenance of the NuGet vulnerability audit, security analyzer severities, guardrail tests (endpoint inventory, JWT guard, forbidden-pattern scan with a reviewed allowlist, layer dependencies), the Azure Pipelines security stage, CODEOWNERS for agent and guardrail files, and the branch-rule checklist for repo admins. Use when asked to set up or change guardrails, when a guardrail test or pipeline security check fails, when someone asks for an allowlist exception, or whenever CI security enforcement comes up.
license: Proprietary
metadata:
  version: 1.7.1
  owner: api-layer-team
  agent: dotnet-coder
  contract: api-guardrails/1
  source: api-layer-engineer v1.7.1 (body unchanged; frontmatter added by the warehouse)
disable-model-invocation: true
---

# Security guardrails

> **Placeholders:** paths, names and handles in examples (such as `eng/security/`, `<owners>`, `Contoso`) are illustrations. Use confirmed values from `.agents/api-layer/project-facts.md`, or ask the user. Never invent a code owner, an approver or a date.

> **Team add-on.** Use this skill only if `project-facts.md` records "Guardrails adopted by the team" as Yes. Guardrails are project code, shared through a normal pull request; the agent itself is installed locally by each developer. If guardrails are not adopted, say so, offer to explain what adopting them involves, and create no guardrail files.

Guardrails turn the most important rules into checks that behave the same in every IDE build, in `dotnet test` and in CI. They do not depend on the model following instructions.

## Rules (most important first)
1. **Never change a guardrail to make something pass.** Any change to a guardrail file is a **guardrail change**: its own plan, its reason, and explicit approval. Guardrail files are: build security settings (`Directory.Build.props`, the security rules in `.editorconfig`), guardrail tests, the forbidden-pattern allowlist, the pipeline security stage and `CODEOWNERS`.
2. **Allowlist entries are reviewed exceptions, not fixes.** Each one needs a rule, file, matching text, reason, approver and review date. The approver and date come from the user; never invent them. Expired, incomplete or stale entries fail the tests.
3. **Block from day one** (confirmed in project-facts): there is no warn mode. Before a gate is enabled, scan the existing code. Fix every existing violation, or allowlist it with approval, so old code does not block the first pull request.
4. **Report honestly**: every check is PASS, FAIL, SKIPPED or NOT RUN, with its command and output.
5. **Platform settings belong to admins**: branch rules, required checks, secret scanning settings and pipeline ownership. Produce the checklist or a cross-repo change request; never claim these are done.

## What is enforced, and where
| Control | Enforced by | Runs in |
|---|---|---|
| Known-vulnerable packages, direct and transitive | NuGet audit as errors | Every restore and build; CI |
| Security analyzer findings (SQL strings, weak crypto, disabled TLS validation, unsafe deserialization) | `.editorconfig` severities | Every build; CI |
| Every endpoint names a policy, or is an allowlisted anonymous endpoint | Endpoint inventory test | `dotnet test`; CI |
| Token validation not weakened | JWT options test | `dotnet test`; CI |
| No internal details in errors; CORS allowlist works | Leakage and CORS tests | `dotnet test`; CI |
| Forbidden patterns (raw SQL, filter bypass, any-origin CORS, disabled validation, sensitive logging, secrets in code or config, and more) | Forbidden-pattern scan + allowlist | `dotnet test`; CI |
| Layer dependencies | Layer dependency test | `dotnet test`; CI |
| AutoMapper configuration | Configuration test | `dotnet test`; CI |
| Secrets in commits and pushes | Secret scanning (tool chosen at setup) | GitHub or CI |
| Agent and guardrail files reviewed | `CODEOWNERS` + code-owner review rule | GitHub pull requests |
| Merge blocked when checks fail | Required status checks | GitHub branch rules |

Some controls are guided rather than enforced: object-level checks inside use cases, business rules and response field choices. They are covered by the full-stack tests written with each feature and by code review.

## Setup procedure (one time, via `/setup-guardrails`)
1. **Discover (read-only)**: the solution file, projects per layer, test projects and folder names, and any existing `Directory.Build.props`, `.editorconfig`, `nuget.config`, pipeline YAML and `CODEOWNERS`.
2. **Ask** only what is not already in project-facts:
   - code-owner GitHub handles or teams;
   - which secret scanning tool to use;
   - where the guardrail tests and the allowlist live;
   - which layer-test approach to use;
   - who owns the pipeline and where its YAML lives.
3. **Baseline scan (read-only)**: search for every forbidden-pattern rule, anonymous endpoints and analyzer findings. List each existing violation with a proposal: fix now, or allowlist with a reason, approver and review date supplied by the user.
4. **Plan** in the normal format, listing every guardrail file to add or change. Then wait for approval.
5. **Build** in this order: build settings → guardrail tests → allowlist (approved entries only) → pipeline stage (or a cross-repo change request if the pipeline is owned elsewhere) → `CODEOWNERS` (real handles only).
6. **Verify**: run `dotnet restore`, `dotnet build` and `dotnet test`; report each as PASS / FAIL / SKIPPED / NOT RUN.
7. **Hand over** the branch-rule checklist to a repo admin. Merges are blocked only after the admin applies it.
8. **Record** locations, owners and tool choices in project-facts (with approval), and remind the user to send them to the package maintainer.

## References
- [build-settings.md](references/build-settings.md): NuGet audit as errors, security analyzer severities, supply-chain options.
- [forbidden-patterns-test.md](references/forbidden-patterns-test.md): the scan test, its rules, the allowlist format and its limits.
- [layer-and-security-tests.md](references/layer-and-security-tests.md): the layer dependency test, and where each guardrail test's code comes from.
- [azure-pipelines-security-stage.md](references/azure-pipelines-security-stage.md): the security stage template, how to wire it, and the change request for when the pipeline is owned elsewhere.
- [github-protection.md](references/github-protection.md): `CODEOWNERS` template, branch-rule checklist, secret scanning options.

## When a guardrail fails
Fix the code. If you believe the failure is a false positive, explain why and propose an allowlist entry as a guardrail change; the user supplies the approver and review date.
