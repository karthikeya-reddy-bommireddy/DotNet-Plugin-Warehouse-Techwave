---
name: api-pipeline-hardening
description: Hardening of the ASP.NET Core .NET 10 API host on Azure App Service - Program.cs registration and middleware order, HTTPS/HSTS and forwarded headers, security response headers, CORS for the web and app clients, rate limiting, Kestrel/IIS request limits and timeouts, global exception handling with ProblemDetails, Key Vault and Managed Identity configuration, health checks, Swagger exposure, and safe outbound HttpClient use (SSRF, unsafe API consumption). Use whenever Program.cs, middleware, appsettings, CORS, rate limits, error handling, secrets/configuration or outbound HTTP calls are added or changed.
license: Proprietary
metadata:
  version: 1.7.1
  owner: api-layer-team
  agent: dotnet-coder
  contract: api-pipeline-hardening/1
  source: api-layer-engineer v1.7.1 (body unchanged; frontmatter added by the warehouse)
disable-model-invocation: true
---

# API host and pipeline hardening

> **Placeholders:** example names and numbers are illustrations. Use project-facts or ask; never copy them into code.

## Rules (most important first)
1. **Middleware order is a security control.** Use the order below; the exception handler first, CORS before authentication, authentication before authorization.
2. **Production shows nothing internal**: no Developer Exception Page, no Swagger, no detailed errors, no `ShowPII`, no `EnableSensitiveDataLogging`.
3. **HTTPS only**: HSTS outside Development and HTTPS redirection in code; App Service "HTTPS Only" and minimum TLS 1.2 on the platform (flag it if unknown).
4. **CORS is an explicit allowlist** of the web client's origins per environment, with explicit methods and headers, no `AllowAnyOrigin` and no credentials (bearer tokens need no cookies). CORS is not authorization.
5. **Rate limiting everywhere**: a global partitioned limiter (per user `oid`, otherwise per client IP) plus stricter named policies for writes, search, export and sensitive flows. 429 responses are ProblemDetails with `Retry-After`.
6. **Small default body limit** (for example 1 MB), raised per endpoint with `[RequestSizeLimit]`; request timeouts for slow endpoints. With in-process IIS hosting (Windows App Service), IIS limits apply instead of Kestrel's.
7. **One global exception handler** (`IExceptionHandler` + `AddProblemDetails`) that maps known Application exceptions to 4xx, everything else to a generic 500, logs the exception itself, and never returns exception text.
8. **Secrets come from Key Vault via Managed Identity** (or App Service Key Vault references); options are validated at startup (`ValidateOnStart`); `appsettings*.json` holds no secrets.
9. **Outbound HTTP uses typed clients** with fixed base addresses, timeouts and resilience; never a request-supplied URL without an allowlist; never disabled certificate validation.
10. **Health endpoints**: liveness anonymous with no details; readiness and detailed checks authenticated or internal.
11. **Forwarded headers only from trusted proxies.** Behind App Service front ends or Front Door, configure them so scheme and client IP are right; otherwise every client shares one IP-based rate-limit partition.

## Pipeline order
1. `UseForwardedHeaders` (with trusted proxies configured for the hosting)
2. `UseExceptionHandler`
3. `UseHsts` (outside Development), `UseHttpsRedirection`
4. Security headers middleware
5. `UseSerilogRequestLogging`
6. `UseCors(policy)`
7. `UseAuthentication`
8. `UseRateLimiter` (after authentication so per-user partitions work)
9. `UseAuthorization`
10. `UseRequestTimeouts`
11. Swagger (non-Production only), `MapControllers`, health checks

## References
- [program-cs-golden.md](references/program-cs-golden.md): annotated Program.cs plus the CORS, security header, rate limiting and exception handler code.
- [secrets-config-outbound.md](references/secrets-config-outbound.md): Key Vault + Managed Identity, options validation, typed HttpClients, SSRF guard.
