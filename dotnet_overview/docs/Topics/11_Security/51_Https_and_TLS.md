## 51. HTTPS & TLS

## Short Introduction

HTTPS (HTTP Secure) and TLS (Transport Layer Security) provide encrypted communication between clients and servers. In .NET Core applications, HTTPS is essential for secure data transmission, authentication, and meeting compliance requirements.

## Official Definition

HTTPS is HTTP over TLS/SSL, providing encryption, data integrity, and authentication. TLS is the cryptographic protocol that secures the connection between client and server, replacing the older SSL protocol.

## Setup/Usage with .NET 8+ Code

**Basic HTTPS Configuration in Program.cs:**

```csharp
var builder = WebApplication.CreateBuilder(args);

// Configure HTTPS redirection
builder.Services.AddHttpsRedirection(options =>
{
    options.RedirectStatusCode = StatusCodes.Status307TemporaryRedirect;
    options.HttpsPort = 5001;
});

// Configure HSTS (HTTP Strict Transport Security)
builder.Services.AddHsts(options =>
{
    options.Preload = true;
    options.IncludeSubDomains = true;
    options.MaxAge = TimeSpan.FromDays(365);
});

var app = builder.Build();

// Use HTTPS redirection
if (!app.Environment.IsDevelopment())
{
    app.UseHsts();
}

app.UseHttpsRedirection();
```

**Custom Certificate Configuration:**

```csharp
// Program.cs - Custom certificate
var builder = WebApplication.CreateBuilder(args);

builder.WebHost.ConfigureKestrel(options =>
{
    options.Listen(IPAddress.Any, 5000); // HTTP
    options.Listen(IPAddress.Any, 5001, listenOptions =>
    {
        listenOptions.UseHttps(httpsOptions =>
        {
            httpsOptions.ServerCertificate = new X509Certificate2("certificate.pfx", "password");
        });
    });
});
```

**Development Certificate Management:**

```bash
# Generate development certificate
dotnet dev-certs https --trust

# Export certificate
dotnet dev-certs https --export-path certificate.pfx --password YourPassword

# Clean certificates
dotnet dev-certs https --clean
```

### Use Cases

- **Secure API Communication**: Protecting sensitive data in REST APIs
- **Authentication Token Protection**: Securing JWT tokens and cookies
- **Compliance Requirements**: Meeting PCI DSS, HIPAA, GDPR requirements
- **SEO Benefits**: Search engines favor HTTPS sites
- **User Trust**: Browser security indicators for user confidence
- **Preventing MITM Attacks**: Protection against man-in-the-middle attacks

## When to Use vs When Not to Use

**Use HTTPS when:**

- Handling sensitive data (passwords, personal info, payment data)
- Implementing authentication/authorization
- Meeting compliance requirements
- Building production applications
- Requiring data integrity guarantees

**Consider alternatives when:**

- Internal development/testing only
- Non-sensitive static content delivery
- Performance is critical and security is not required
- Legacy system constraints

### Market Alternatives & Pros/Cons

### Alternatives:

- **Cloudflare**: CDN with automatic HTTPS
- **AWS Certificate Manager**: Free SSL certificates for AWS resources
- **Let's Encrypt**: Free automated certificates
- **Commercial CAs**: DigiCert, GlobalSign, Comodo

### Pros:

- Data encryption and integrity
- Authentication of server identity
- Browser compatibility and trust indicators
- SEO and performance benefits (HTTP/2)
- Compliance with security standards

### Cons:

- Additional CPU overhead for encryption
- Certificate management complexity
- Initial setup and configuration effort
- Certificate renewal requirements

### Complete Runnable Sample

**Enhanced HTTPS Configuration:**

```csharp
// Program.cs
using Microsoft.AspNetCore.HttpsPolicy;
using System.Security.Cryptography.X509Certificates;

var builder = WebApplication.CreateBuilder(args);

// Add services
builder.Services.AddControllers();

// Configure HTTPS redirection with custom options
builder.Services.AddHttpsRedirection(options =>
{
    options.RedirectStatusCode = StatusCodes.Status308PermanentRedirect;
    options.HttpsPort = builder.Environment.IsDevelopment() ? 5001 : 443;
});

// Configure HSTS
builder.Services.AddHsts(options =>
{
    options.Preload = true;
    options.IncludeSubDomains = true;
    options.MaxAge = TimeSpan.FromDays(365);
    options.ExcludedHosts.Clear(); // Remove localhost exclusion for development
});

// Configure Kestrel for production
if (!builder.Environment.IsDevelopment())
{
    builder.WebHost.ConfigureKestrel(options =>
    {
        options.ConfigureHttpsDefaults(httpsOptions =>
        {
            httpsOptions.SslProtocols = System.Security.Authentication.SslProtocols.Tls12 |
                                       System.Security.Authentication.SslProtocols.Tls13;
        });
    });
}

var app = builder.Build();

// Configure middleware pipeline
if (app.Environment.IsDevelopment())
{
    app.UseDeveloperExceptionPage();
}
else
{
    app.UseExceptionHandler("/Error");
    app.UseHsts(); // Add HSTS header
}

app.UseHttpsRedirection();
app.UseRouting();

// Security headers middleware
app.Use(async (context, next) =>
{
    context.Response.Headers.Add("X-Content-Type-Options", "nosniff");
    context.Response.Headers.Add("X-Frame-Options", "DENY");
    context.Response.Headers.Add("X-XSS-Protection", "1; mode=block");
    context.Response.Headers.Add("Referrer-Policy", "strict-origin-when-cross-origin");

    await next();
});

app.MapControllers();

app.Run();

// appsettings.Production.json
{
  "Kestrel": {
    "Endpoints": {
      "Http": {
        "Url": "http://0.0.0.0:80"
      },
      "Https": {
        "Url": "https://0.0.0.0:443",
        "Certificate": {
          "Path": "/app/certificates/certificate.pfx",
          "Password": "YourCertificatePassword"
        }
      }
    }
  },
  "HttpsRedirection": {
    "HttpsPort": 443
  }
}

// Controllers/SecureController.cs - Example secure endpoint
[ApiController]
[Route("api/[controller]")]
[RequireHttps] // Enforce HTTPS
public class SecureController : ControllerBase
{
    [HttpGet("secure-data")]
    public IActionResult GetSecureData()
    {
        // Check if request is secure
        if (!Request.IsHttps)
        {
            return BadRequest("HTTPS required");
        }

        return Ok(new {
            Message = "This is secure data",
            IsSecure = Request.IsHttps,
            Protocol = Request.Protocol,
            Scheme = Request.Scheme
        });
    }

    [HttpPost("sensitive-data")]
    public IActionResult PostSensitiveData([FromBody] SensitiveDataModel data)
    {
        // Additional security headers
        Response.Headers.Add("Cache-Control", "no-store, no-cache, must-revalidate");
        Response.Headers.Add("Pragma", "no-cache");

        return Ok(new { Message = "Data received securely" });
    }
}

public class SensitiveDataModel
{
    public string? PersonalInfo { get; set; }
    public string? CreditCardNumber { get; set; }
}

// Certificate validation middleware
public class CertificateValidationMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<CertificateValidationMiddleware> _logger;

    public CertificateValidationMiddleware(RequestDelegate next, ILogger<CertificateValidationMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        var clientCert = context.Connection.ClientCertificate;

        if (clientCert != null)
        {
            _logger.LogInformation("Client certificate: {Subject}", clientCert.Subject);

            // Validate certificate if required
            if (!IsValidCertificate(clientCert))
            {
                context.Response.StatusCode = 403;
                await context.Response.WriteAsync("Invalid client certificate");
                return;
            }
        }

        await _next(context);
    }

    private bool IsValidCertificate(X509Certificate2 certificate)
    {
        // Implement certificate validation logic
        return certificate.NotAfter > DateTime.Now && certificate.NotBefore <= DateTime.Now;
    }
}

// Docker configuration for HTTPS
# Dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS base
WORKDIR /app
EXPOSE 80
EXPOSE 443

# Copy certificate
COPY ["certificates/", "/app/certificates/"]

FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY ["SecureApp.csproj", "."]
RUN dotnet restore "./SecureApp.csproj"
COPY . .
WORKDIR "/src/."
RUN dotnet build "SecureApp.csproj" -c Release -o /app/build

FROM build AS publish
RUN dotnet publish "SecureApp.csproj" -c Release -o /app/publish

FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "SecureApp.dll"]
```

**NuGet Packages Required:**

```xml
<PackageReference Include="Microsoft.AspNetCore.HttpsPolicy" Version="2.2.0" />
```
