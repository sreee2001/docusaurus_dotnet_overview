## 48. Identity Server

### Introduction

Identity Server (now Duende Identity Server) is a comprehensive OpenID Connect and OAuth 2.0 framework for .NET applications. It provides centralized authentication and authorization services, enabling single sign-on (SSO) across multiple applications and APIs.

## Official Definition

Duende Identity Server is a certified OpenID Connect and OAuth 2.0 implementation that provides authentication as a service (AaaS) for modern web applications, APIs, and mobile applications. It supports various authentication flows and can act as a Security Token Service (STS).

### Usage & Setup

```bash
# Install Duende Identity Server
dotnet add package Duende.IdentityServer
dotnet add package Duende.IdentityServer.AspNetIdentity
dotnet add package Duende.IdentityServer.EntityFramework
```

```csharp
// Program.cs
using Duende.IdentityServer;
using Microsoft.AspNetCore.Identity;
using Microsoft.EntityFrameworkCore;

var builder = WebApplication.CreateBuilder(args);

// Add Entity Framework
builder.Services.AddDbContext<ApplicationDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection")));

// Add Identity
builder.Services.AddIdentity<ApplicationUser, IdentityRole>()
    .AddEntityFrameworkStores<ApplicationDbContext>()
    .AddDefaultTokenProviders();

// Add Identity Server
builder.Services.AddIdentityServer(options =>
{
    options.Events.RaiseErrorEvents = true;
    options.Events.RaiseInformationEvents = true;
    options.Events.RaiseFailureEvents = true;
    options.Events.RaiseSuccessEvents = true;
})
.AddInMemoryIdentityResources(Config.IdentityResources)
.AddInMemoryApiScopes(Config.ApiScopes)
.AddInMemoryClients(Config.Clients)
.AddAspNetIdentity<ApplicationUser>()
.AddDeveloperSigningCredential(); // For development only

var app = builder.Build();

app.UseIdentityServer();
```

### Use Cases

- Single Sign-On (SSO) across multiple applications
- Centralized user authentication for microservices
- API security and access token management
- Third-party application integration
- Enterprise identity federation
- Mobile app authentication
- B2B and B2C identity scenarios

## When to Use vs When Not to Use

**Use When:**

- Multiple applications need shared authentication
- Implementing microservices architecture
- Requiring enterprise SSO
- Need OAuth 2.0/OpenID Connect compliance
- Managing API access across applications
- Building identity-as-a-service solutions

**Don't Use When:**

- Single application with simple auth needs
- No external integrations required
- Very small teams/applications
- Extremely performance-sensitive scenarios
- Limited infrastructure/maintenance capability

### Market Alternatives & Adoption

### Alternatives:

- Auth0 (SaaS solution)
- Azure Active Directory B2C
- AWS Cognito
- Okta
- Keycloak (open source)
- Firebase Authentication

**Market Position:** Leading on-premises identity solution for .NET ecosystem with strong enterprise adoption.

### Pros and Cons

### Pros:

- Full OAuth 2.0/OpenID Connect compliance
- Highly customizable and extensible
- Strong security features
- Excellent .NET integration
- Active community and support
- On-premises deployment control

### Cons:

- Complex setup and configuration
- Requires significant expertise
- Higher maintenance overhead
- Licensing costs (Duende)
- Learning curve for team
- Infrastructure requirements

### Sample Implementation

```csharp
// Config.cs - Identity Server Configuration
public static class Config
{
    public static IEnumerable<IdentityResource> IdentityResources =>
        new IdentityResource[]
        {
            new IdentityResources.OpenId(),
            new IdentityResources.Profile(),
            new IdentityResources.Email(),
            new IdentityResource("roles", "User roles", new[] { "role" })
        };

    public static IEnumerable<ApiScope> ApiScopes =>
        new ApiScope[]
        {
            new ApiScope("hotel.api", "Hotel Management API"),
            new ApiScope("booking.api", "Booking API"),
            new ApiScope("admin.api", "Admin API")
        };

    public static IEnumerable<Client> Clients =>
        new Client[]
        {
            // Interactive ASP.NET Core MVC client
            new Client
            {
                ClientId = "hotel.mvc",
                ClientSecrets = { new Secret("secret".Sha256()) },

                AllowedGrantTypes = GrantTypes.Code,

                RedirectUris = { "https://localhost:5001/signin-oidc" },
                PostLogoutRedirectUris = { "https://localhost:5001/signout-callback-oidc" },

                AllowedScopes = new List<string>
                {
                    IdentityServerConstants.StandardScopes.OpenId,
                    IdentityServerConstants.StandardScopes.Profile,
                    IdentityServerConstants.StandardScopes.Email,
                    "roles",
                    "hotel.api"
                },

                RequirePkce = true,
                AllowOfflineAccess = true
            },

            // JavaScript SPA Client
            new Client
            {
                ClientId = "hotel.spa",
                ClientName = "Hotel SPA",

                AllowedGrantTypes = GrantTypes.Code,
                RequireClientSecret = false,
                RequirePkce = true,

                RedirectUris = { "https://localhost:3000/callback" },
                PostLogoutRedirectUris = { "https://localhost:3000/" },
                AllowedCorsOrigins = { "https://localhost:3000" },

                AllowedScopes = new List<string>
                {
                    IdentityServerConstants.StandardScopes.OpenId,
                    IdentityServerConstants.StandardScopes.Profile,
                    IdentityServerConstants.StandardScopes.Email,
                    "hotel.api",
                    "booking.api"
                }
            },

            // Machine to machine client
            new Client
            {
                ClientId = "hotel.m2m",
                ClientSecrets = { new Secret("secret".Sha256()) },

                AllowedGrantTypes = GrantTypes.ClientCredentials,

                AllowedScopes = { "admin.api" }
            }
        };
}

// Models/ApplicationUser.cs (Extended)
public class ApplicationUser : IdentityUser
{
    public string FirstName { get; set; }
    public string LastName { get; set; }
    public DateTime DateCreated { get; set; }
    public string Department { get; set; }
    public bool IsActive { get; set; }
}

// Services/ProfileService.cs - Custom claims
public class ProfileService : IProfileService
{
    private readonly UserManager<ApplicationUser> _userManager;
    private readonly RoleManager<IdentityRole> _roleManager;

    public ProfileService(UserManager<ApplicationUser> userManager, RoleManager<IdentityRole> roleManager)
    {
        _userManager = userManager;
        _roleManager = roleManager;
    }

    public async Task GetProfileDataAsync(ProfileDataRequestContext context)
    {
        var user = await _userManager.GetUserAsync(context.Subject);
        if (user != null)
        {
            var roles = await _userManager.GetRolesAsync(user);
            var claims = new List<Claim>
            {
                new Claim("firstName", user.FirstName ?? ""),
                new Claim("lastName", user.LastName ?? ""),
                new Claim("department", user.Department ?? ""),
                new Claim("email", user.Email ?? "")
            };

            foreach (var role in roles)
            {
                claims.Add(new Claim("role", role));
            }

            context.IssuedClaims.AddRange(claims);
        }
    }

    public async Task IsActiveAsync(IsActiveContext context)
    {
        var user = await _userManager.GetUserAsync(context.Subject);
        context.IsActive = user?.IsActive == true;
    }
}

// Client Application Configuration (MVC App)
// Program.cs (in client MVC app)
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddAuthentication(options =>
{
    options.DefaultScheme = "Cookies";
    options.DefaultChallengeScheme = "oidc";
})
.AddCookie("Cookies")
.AddOpenIdConnect("oidc", options =>
{
    options.Authority = "https://localhost:5000"; // Identity Server URL
    options.ClientId = "hotel.mvc";
    options.ClientSecret = "secret";
    options.ResponseType = "code";

    options.SaveTokens = true;
    options.GetClaimsFromUserInfoEndpoint = true;

    options.Scope.Add("hotel.api");
    options.Scope.Add("roles");
    options.Scope.Add("offline_access");

    options.ClaimActions.MapJsonKey("role", "role", "role");
    options.TokenValidationParameters.NameClaimType = "name";
    options.TokenValidationParameters.RoleClaimType = "role";
});

var app = builder.Build();

app.UseAuthentication();
app.UseAuthorization();

// Controllers/HomeController.cs (in client app)
[Authorize]
public class HomeController : Controller
{
    private readonly IHttpClientFactory _httpClientFactory;

    public HomeController(IHttpClientFactory httpClientFactory)
    {
        _httpClientFactory = httpClientFactory;
    }

    public async Task<IActionResult> CallApi()
    {
        var accessToken = await HttpContext.GetTokenAsync("access_token");

        var client = _httpClientFactory.CreateClient();
        client.DefaultRequestHeaders.Authorization =
            new System.Net.Http.Headers.AuthenticationHeaderValue("Bearer", accessToken);

        var response = await client.GetStringAsync("https://localhost:6001/api/hotels");

        ViewBag.Json = response;
        return View();
    }

    public IActionResult Logout()
    {
        return SignOut("Cookies", "oidc");
    }
}

// API Configuration (Protected API)
// Program.cs (in API project)
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddAuthentication("Bearer")
    .AddJwtBearer("Bearer", options =>
    {
        options.Authority = "https://localhost:5000";
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateAudience = false
        };
    });

builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("ApiScope", policy =>
    {
        policy.RequireAuthenticatedUser();
        policy.RequireClaim("scope", "hotel.api");
    });
});

var app = builder.Build();

app.UseAuthentication();
app.UseAuthorization();

// Controllers/HotelsController.cs (in API)
[ApiController]
[Route("api/[controller]")]
[Authorize("ApiScope")]
public class HotelsController : ControllerBase
{
    [HttpGet]
    public ActionResult<IEnumerable<object>> Get()
    {
        var claims = User.Claims.Select(c => new { c.Type, c.Value });

        return Ok(new
        {
            message = "Hello from protected API",
            user = User.Identity.Name,
            claims = claims
        });
    }
}
```
