# Serilog setup

> **Placeholders:** example names and numbers are illustrations. Use project-facts or ask; never copy them into code.

## Packages
`Serilog.AspNetCore`, `Serilog.Settings.Configuration`, `Serilog.Sinks.Console`, `Serilog.Sinks.File`, `Serilog.Formatting.Compact`, plus the team's APM sink (for example Application Insights). Packages and versions are team decisions.

## Program.cs
```csharp
using Microsoft.Identity.Web;
using Serilog;
using Serilog.Events;

Log.Logger = new LoggerConfiguration()          // bootstrap logger catches startup failures
    .WriteTo.Console()
    .CreateBootstrapLogger();

try
{
    var builder = WebApplication.CreateBuilder(args);
    builder.Services.AddSerilog((services, lc) => lc
        .ReadFrom.Configuration(builder.Configuration)
        .ReadFrom.Services(services)
        .Enrich.FromLogContext());
    // ... services ...

    var app = builder.Build();
    app.UseExceptionHandler();
    app.UseSerilogRequestLogging(o =>
    {
        o.EnrichDiagnosticContext = (diagnostic, http) =>
        {
            diagnostic.Set("UserObjectId", http.User.GetObjectId());
            diagnostic.Set("TenantId", http.User.GetTenantId());
            diagnostic.Set("ClientAppId", http.User.FindFirst("azp")?.Value ?? http.User.FindFirst("appid")?.Value);
        };
        // 4xx are expected; 5xx and exceptions are errors
        o.GetLevel = (http, _, ex) =>
            ex is not null || http.Response.StatusCode >= 500 ? LogEventLevel.Error : LogEventLevel.Information;
    });
    // ... rest of the pipeline ...
    app.Run();
}
catch (Exception ex) when (ex is not HostAbortedException)
{
    Log.Fatal(ex, "API host terminated unexpectedly");
}
finally
{
    Log.CloseAndFlush();
}
```

## appsettings.json (production defaults, no secrets)
```json
{
  "Serilog": {
    "Using": [ "Serilog.Sinks.Console", "Serilog.Sinks.File" ],
    "MinimumLevel": {
      "Default": "Information",
      "Override": {
        "Microsoft.AspNetCore": "Warning",
        "Microsoft.EntityFrameworkCore": "Warning",
        "System": "Warning"
      }
    },
    "WriteTo": [
      {
        "Name": "Console",
        "Args": { "formatter": "Serilog.Formatting.Compact.RenderedCompactJsonFormatter, Serilog.Formatting.Compact" }
      },
      {
        "Name": "File",
        "Args": {
          "path": "logs/api-.json",
          "rollingInterval": "Day",
          "retainedFileCountLimit": 14,
          "fileSizeLimitBytes": 104857600,
          "rollOnFileSizeLimit": true,
          "formatter": "Serilog.Formatting.Compact.CompactJsonFormatter, Serilog.Formatting.Compact"
        }
      }
    ],
    "Enrich": [ "FromLogContext" ],
    "Properties": { "Application": "Contoso.Api" }
  }
}
```
- File sink: on App Service write under the platform log directory, never under `wwwroot`; keep retention short and in line with policy.
- APM sink: configure it with the connection string from App Service settings or Key Vault, never from a committed file.
- Recent Serilog versions capture `TraceId`/`SpanId` from the current `Activity`; the `traceId` in ProblemDetails lets support find the matching log entry.
- `appsettings.Development.json` may raise verbosity; production configuration must not.

## Masking
If a value must appear partially (for example the last 4 characters of a reference), build the masked string yourself before logging, or use a destructuring policy for a dedicated log type. Masking never makes tokens or bodies safe to log.
