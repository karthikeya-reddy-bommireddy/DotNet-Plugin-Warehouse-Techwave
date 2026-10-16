---
name: api-secure-logging
description: Secure structured logging with Serilog for the ASP.NET Core .NET 10 APIs repo - bootstrap and configuration-driven sinks (console, file, APM), request logging with user and client enrichment, traceId correlation, the security event catalog (token rejected, authorization denied, BOLA attempts, rate limits, privileged actions) and strict rules on what must never be logged (tokens, secrets, bodies, personal data). Use whenever adding or changing log statements, Serilog configuration, request logging, audit trails or diagnostics, even for a single new log line.
license: Proprietary
metadata:
  version: 1.7.1
  owner: api-layer-team
  agent: dotnet-coder
  contract: api-secure-logging/1
  source: api-layer-engineer v1.7.1 (body unchanged; frontmatter added by the warehouse)
disable-model-invocation: true
---

# Secure logging with Serilog

> **Placeholders:** example names and numbers are illustrations. Use project-facts or ask; never copy them into code.

## Rules (most important first)
1. **Never log**: access tokens, `Authorization`/`Cookie` headers, passwords, secrets, connection strings, API keys, full request or response bodies, file contents, or personal data (names, emails, phone numbers, addresses) unless the team's data policy allows a masked form.
2. **Identify callers by `oid`** (plus `tid` and client app ID), never by email or name.
3. **Message templates, not interpolation**: `logger.LogWarning("Order {OrderId} rejected", id)`. `$"..."` in log calls breaks structure and enables log forging in text sinks.
4. **Destructure (`{@Value}`) only types designed for logging**; never request DTOs, entities, `HttpContext` or configuration objects.
5. **Security events use the catalog** with stable event IDs so they can be queried and alerted on.
6. **Production levels**: application `Information`; `Microsoft.AspNetCore`, `Microsoft.EntityFrameworkCore` and `System` at `Warning`; no `Debug`/`Verbose`; no HTTP logging of headers or bodies; EF Core `EnableSensitiveDataLogging` only in Development.
7. **Exceptions are logged once**, by the global exception handler, with the traceId; responses carry only the traceId.
8. **Logs are personal data** (object IDs, IPs): retention and access follow the team's policy; sink credentials come from App Service settings or Key Vault.

## Minimal setup
```csharp
builder.Services.AddSerilog((services, lc) => lc
    .ReadFrom.Configuration(builder.Configuration)
    .ReadFrom.Services(services)
    .Enrich.FromLogContext());

app.UseSerilogRequestLogging(o => o.EnrichDiagnosticContext = (diagnostic, http) =>
{
    diagnostic.Set("UserObjectId", http.User.GetObjectId());
    diagnostic.Set("ClientAppId", http.User.FindFirst("azp")?.Value ?? http.User.FindFirst("appid")?.Value);
});
```
Request logging records method, path (no query string by default), status and duration. Do not add headers, query strings or bodies.

## References
- [serilog-setup.md](references/serilog-setup.md): bootstrap logger, appsettings with production levels, file and APM sink notes, masking.
- [security-events.md](references/security-events.md): event catalog, `LoggerMessage` helpers, central logging of authorization denials.

## Checklist for any log line
- [ ] Template with named properties, no interpolation
- [ ] No token, secret, body or personal data; IDs only
- [ ] Right level (security denials = Warning; expected 404s not logged as errors)
- [ ] Security events through the catalog
- [ ] Exception logged once
