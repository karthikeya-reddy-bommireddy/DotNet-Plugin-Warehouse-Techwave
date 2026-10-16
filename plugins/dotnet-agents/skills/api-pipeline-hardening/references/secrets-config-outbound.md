# Secrets, configuration and outbound HTTP

> **Placeholders:** example names and numbers are illustrations. Use project-facts or ask; never copy them into code.

## Key Vault with Managed Identity
```csharp
using Azure.Identity;

public static class ConfigurationSetup
{
    public static WebApplicationBuilder AddApiConfiguration(this WebApplicationBuilder builder)
    {
        var vaultUri = builder.Configuration["KeyVault:Uri"];
        if (!string.IsNullOrWhiteSpace(vaultUri))
        {
            // Deterministic credential in Azure; DefaultAzureCredential probes many sources.
            var credential = new ManagedIdentityCredential(ManagedIdentityId.SystemAssigned);
            builder.Configuration.AddAzureKeyVault(new Uri(vaultUri), credential);
        }
        else if (builder.Environment.IsProduction() || builder.Environment.IsStaging())
        {
            throw new InvalidOperationException("KeyVault:Uri must be configured in Staging and Production.");
        }

        builder.Services.AddOptions<PartnerApiOptions>()
            .Bind(builder.Configuration.GetSection(PartnerApiOptions.Section))
            .ValidateDataAnnotations()
            .ValidateOnStart();

        return builder;
    }
}
```
- Packages: `Azure.Extensions.AspNetCore.Configuration.Secrets` and `Azure.Identity`. The `ManagedIdentityId` overload needs a recent Azure.Identity; for a user-assigned identity use `ManagedIdentityId.FromUserAssignedClientId(...)`.
- No-code alternative: App Service app settings with Key Vault references (`@Microsoft.KeyVault(SecretUri=...)`).
- Local development: `dotnet user-secrets`; tests: fictional values; never committed secret files.
- Key Vault secret names use `--` for the `:` separator.
- Grant the App Service identity only "Key Vault Secrets User" on the vault (RBAC).
- Startup validation turns a missing or malformed setting into a failed deployment instead of a fail-open runtime.
- Never log configuration values or return them from diagnostics endpoints.

## Outbound HTTP (API7 SSRF, API10 unsafe consumption)
```csharp
builder.Services
    .AddHttpClient<IPartnerClient, PartnerClient>((sp, client) =>
    {
        var options = sp.GetRequiredService<IOptions<PartnerApiOptions>>().Value;
        client.BaseAddress = options.BaseAddress;             // from configuration, https only
        client.MaxResponseContentBufferSize = 1 * 1024 * 1024;
    })
    .ConfigurePrimaryHttpMessageHandler(() => new SocketsHttpHandler { AllowAutoRedirect = false })
    .AddStandardResilienceHandler(o =>                        // Microsoft.Extensions.Http.Resilience
    {
        o.AttemptTimeout.Timeout = TimeSpan.FromSeconds(3);
        o.TotalRequestTimeout.Timeout = TimeSpan.FromSeconds(10);
    });
```
- Retries can repeat non-idempotent calls (POST): disable retries for unsafe methods where the library supports it, or send idempotency keys.
- Treat every response as untrusted: check status, validate shape, cap size, handle timeouts; never pass downstream error text to clients.
- Never forward the caller's access token to third parties. For downstream Microsoft-protected APIs use the on-behalf-of flow via Microsoft.Identity.Web.
- Never set `ServerCertificateCustomValidationCallback` or use `DangerousAcceptAnyServerCertificateValidator` outside tests.

## When a URL must come from the client (avoid if possible)
```csharp
using System.Net;
using System.Net.Sockets;

public static class OutboundUrlGuard
{
    public static async Task<bool> IsAllowedAsync(Uri uri, IReadOnlySet<string> allowedHosts, CancellationToken ct)
    {
        if (uri.Scheme != Uri.UriSchemeHttps || !allowedHosts.Contains(uri.IdnHost)) return false;

        var addresses = await Dns.GetHostAddressesAsync(uri.IdnHost, ct);
        return addresses.Length > 0 && addresses.All(IsPublic);
    }

    private static bool IsPublic(IPAddress ip)
    {
        if (ip.IsIPv4MappedToIPv6) ip = ip.MapToIPv4();
        if (IPAddress.IsLoopback(ip) || ip.IsIPv6LinkLocal || ip.IsIPv6SiteLocal || ip.IsIPv6UniqueLocal)
            return false;
        if (ip.AddressFamily != AddressFamily.InterNetwork) return true;

        var b = ip.GetAddressBytes();
        return !(b[0] == 10 || b[0] == 127 || b[0] == 0
            || (b[0] == 172 && b[1] >= 16 && b[1] <= 31)
            || (b[0] == 192 && b[1] == 168)
            || (b[0] == 169 && b[1] == 254)                     // link-local, incl. 169.254.169.254 metadata
            || (b[0] == 100 && b[1] >= 64 && b[1] <= 127));     // carrier-grade NAT
    }
}
```
The host allowlist is the real control; the IP check is defence in depth. DNS can change between check and request (rebinding), so keep redirects off and, for high-risk cases, re-check the connected address in a `SocketsHttpHandler.ConnectCallback`.
