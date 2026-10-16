# JSON hardening and file uploads

> **Placeholders:** example names and numbers are illustrations. Use project-facts or ask; never copy them into code.

## Global JSON options (Program.cs)
```csharp
using System.Text.Json.Serialization;

builder.Services
    .AddControllers(o =>
    {
        o.MaxModelBindingCollectionSize = 1000;          // default 1024; lower if the API never needs more
    })
    .AddJsonOptions(o =>
    {
        o.JsonSerializerOptions.MaxDepth = 32;            // default 64
        o.AllowInputFormatterExceptionMessages = false;   // hide System.Text.Json details from clients
        // Team decision: reject unknown properties everywhere (stricter; may break lenient clients)
        // o.JsonSerializerOptions.UnmappedMemberHandling = JsonUnmappedMemberHandling.Disallow;
    });
```
- Per-DTO alternative: `[JsonUnmappedMemberHandling(JsonUnmappedMemberHandling.Disallow)]` on request DTOs.
- Polymorphism only through `[JsonPolymorphic]`/`[JsonDerivedType]` on known types. If any code uses Newtonsoft.Json, `TypeNameHandling` stays `None`.
- Never hand `object`, `dynamic` or raw `JsonElement` "free-form" payloads to the Application layer without a schema check.

## Injection reminders (mostly Infrastructure; review the call sites)
- EF Core LINQ is parameterized. `FromSqlRaw`/`ExecuteSqlRaw` with concatenated or `$"..."` strings is SQL injection; use `FromSql($"...")` (interpolation becomes parameters) or explicit parameters.
- Never build `OrderBy` or filter expressions from client strings; map allowlisted enums.
- The serializer JSON-encodes output; the web and app clients still encode for their own context (HTML, attributes, URLs).

## File uploads (controllers)
```csharp
private static readonly Dictionary<string, byte[][]> Signatures = new(StringComparer.OrdinalIgnoreCase)
{
    [".pdf"] = [[0x25, 0x50, 0x44, 0x46]],                                 // %PDF
    [".png"] = [[0x89, 0x50, 0x4E, 0x47, 0x0D, 0x0A, 0x1A, 0x0A]],
    [".jpg"] = [[0xFF, 0xD8, 0xFF]],
};
private const long MaxFileBytes = 5 * 1024 * 1024;

[HttpPost("{id:guid}/attachments")]
[Authorize(Policy = Policies.OrdersWrite)]
[EnableRateLimiting(RateLimitPolicies.Writes)]
[RequestSizeLimit(MaxFileBytes + 64 * 1024)]
[RequestFormLimits(MultipartBodyLengthLimit = MaxFileBytes + 64 * 1024)]
public async Task<IActionResult> Upload([FromRoute] Guid id, IFormFile file, CancellationToken ct)
{
    if (file.Length is 0 or > MaxFileBytes)
        return Problem(statusCode: StatusCodes.Status400BadRequest, title: "File size is not allowed.");

    var extension = Path.GetExtension(file.FileName);
    if (!Signatures.TryGetValue(extension, out var allowed))
        return Problem(statusCode: StatusCodes.Status400BadRequest, title: "File type is not allowed.");

    await using var stream = file.OpenReadStream();
    var header = new byte[8];
    var read = await stream.ReadAtLeastAsync(header, header.Length, throwOnEndOfStream: false, ct);
    if (!allowed.Any(sig => read >= sig.Length && header.AsSpan(0, sig.Length).SequenceEqual(sig)))
        return Problem(statusCode: StatusCodes.Status400BadRequest, title: "File content does not match its type.");
    stream.Position = 0;

    // The use case (injected via the constructor) checks the caller owns order {id} (BOLA)
    // and stores the file under a server-generated name in private storage.
    await addAttachment.ExecuteAsync(id, stream, extension.ToLowerInvariant(), ct);
    return NoContent();
}
```
Rules: never use the client's file name or `Content-Type` for decisions or storage paths; store outside the web root (private Blob container, served through the API or short-lived SAS); scan for malware when the business accepts untrusted files; enforce limits in both the host (Kestrel/IIS) and the action.
