## 49. OAuth 2.0 & OpenID Connect

### Introduction

OAuth 2.0 is an authorization framework that enables applications to obtain limited access to user accounts, while OpenID Connect (OIDC) is an identity layer on top of OAuth 2.0 that provides authentication. Together, they form the foundation of modern web application security.

## Official Definition

OAuth 2.0 is a protocol that allows third-party services to exchange web resources on behalf of a user without exposing their password. OpenID Connect 1.0 is a simple identity layer on top of the OAuth 2.0 protocol, allowing clients to verify the identity of the end-user and obtain basic profile information.

### Usage & Setup

```bash
# Install required packages
dotnet add package Microsoft.AspNetCore.Authentication.OpenIdConnect
dotnet add package Microsoft.AspNetCore.Authentication.OAuth
dotnet add package System.IdentityModel.Tokens.Jwt
```

```csharp
// Program.cs - OAuth 2.0 & OpenID Connect Configuration
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddAuthentication(options =>
{
    options.DefaultScheme = CookieAuthenticationDefaults.AuthenticationScheme;
    options.DefaultChallengeScheme = OpenIdConnectDefaults.AuthenticationScheme;
})
.AddCookie(CookieAuthenticationDefaults.AuthenticationScheme)
.AddOpenIdConnect(OpenIdConnectDefaults.AuthenticationScheme, options =>
{
    options.Authority = "https://your-identity-provider.com";
    options.ClientId = "your-client-id";
    options.ClientSecret = "your-client-secret";
    options.ResponseType = OpenIdConnectResponseType.Code;
    options.SaveTokens = true;
    options.GetClaimsFromUserInfoEndpoint = true;

    options.Scope.Add("openid");
    options.Scope.Add("profile");
    options.Scope.Add("email");
    options.Scope.Add("api1");
})
.AddOAuth("GitHub", options =>
{
    options.ClientId = builder.Configuration["GitHub:ClientId"];
    options.ClientSecret = builder.Configuration["GitHub:ClientSecret"];
    options.CallbackPath = "/signin-github";

    options.AuthorizationEndpoint = "https://github.com/login/oauth/authorize";
    options.TokenEndpoint = "https://github.com/login/oauth/access_token";
    options.UserInformationEndpoint = "https://api.github.com/user";

    options.ClaimActions.MapJsonKey(ClaimTypes.NameIdentifier, "id");
    options.ClaimActions.MapJsonKey(ClaimTypes.Name, "name");
    options.ClaimActions.MapJsonKey(ClaimTypes.Email, "email");
    options.ClaimActions.MapJsonKey("urn:github:login", "login");

    options.Events = new OAuthEvents
    {
        OnCreatingTicket = async context =>
        {
            var request = new HttpRequestMessage(HttpMethod.Get, context.Options.UserInformationEndpoint);
            request.Headers.Accept.Add(new MediaTypeWithQualityHeaderValue("application/json"));
            request.Headers.Authorization = new AuthenticationHeaderValue("Bearer", context.AccessToken);

            var response = await context.Backchannel.SendAsync(request, HttpCompletionOption.ResponseHeadersRead, context.HttpContext.RequestAborted);
            response.EnsureSuccessStatusCode();

            var json = JsonDocument.Parse(await response.Content.ReadAsStringAsync());
            context.RunClaimActions(json.RootElement);
        }
    };
});

var app = builder.Build();
app.UseAuthentication();
app.UseAuthorization();
```

### Use Cases

- Single Sign-On (SSO) implementations
- Third-party application integrations
- API access delegation
- Social media login (Google, Facebook, GitHub)
- Enterprise identity federation
- Mobile application authentication
- Microservices security

## When to Use vs When Not to Use

**Use When:**

- Integrating with external identity providers
- Building APIs that need secure access
- Implementing SSO across applications
- Need standard-compliant authentication
- Delegating user permissions to third parties
- Enterprise authentication requirements

**Don't Use When:**

- Simple internal applications
- No external integrations needed
- Basic username/password sufficient
- Performance is absolutely critical
- Compliance doesn't require it
- Team lacks OAuth expertise

### Market Alternatives & Adoption

### Alternatives:

- SAML 2.0 (older enterprise standard)
- Custom JWT implementation
- Basic Authentication
- API Keys
- Session-based authentication

**Market Position:** Industry standard for modern web authentication, adopted by all major platforms (Google, Microsoft, Facebook, etc.).

### Pros and Cons

### Pros:

- Industry standard protocol
- Secure token-based authentication
- Separates authentication from authorization
- Supports multiple grant types
- Excellent third-party support
- Scalable for distributed systems

### Cons:

- Complex implementation
- Multiple flow types can be confusing
- Security misconfiguration risks
- Token management overhead
- Requires HTTPS
- Learning curve for developers

### Sample Implementation

```csharp
// Models/TokenResponse.cs
public class TokenResponse
{
    [JsonPropertyName("access_token")]
    public string AccessToken { get; set; }

    [JsonPropertyName("token_type")]
    public string TokenType { get; set; }

    [JsonPropertyName("expires_in")]
    public int ExpiresIn { get; set; }

    [JsonPropertyName("refresh_token")]
    public string RefreshToken { get; set; }

    [JsonPropertyName("scope")]
    public string Scope { get; set; }
}

// Services/IOAuthService.cs
public interface IOAuthService
{
    Task<TokenResponse> GetAccessTokenAsync(string authorizationCode);
    Task<TokenResponse> RefreshTokenAsync(string refreshToken);
    Task<ClaimsPrincipal> GetUserInfoAsync(string accessToken);
    string GenerateAuthorizationUrl(string state);
}

// Services/OAuthService.cs
public class OAuthService : IOAuthService
{
    private readonly HttpClient _httpClient;
    private readonly IConfiguration _configuration;
    private readonly ILogger<OAuthService> _logger;

    public OAuthService(HttpClient httpClient, IConfiguration configuration, ILogger<OAuthService> logger)
    {
        _httpClient = httpClient;
        _configuration = configuration;
        _logger = logger;
    }

    public string GenerateAuthorizationUrl(string state)
    {
        var authUrl = _configuration["OAuth:AuthorizationEndpoint"];
        var clientId = _configuration["OAuth:ClientId"];
        var redirectUri = _configuration["OAuth:RedirectUri"];
        var scope = _configuration["OAuth:Scope"];

        var queryParams = new Dictionary<string, string>
        {
            ["response_type"] = "code",
            ["client_id"] = clientId,
            ["redirect_uri"] = redirectUri,
            ["scope"] = scope,
            ["state"] = state
        };

        var query = string.Join("&", queryParams.Select(kv => $"{kv.Key}={Uri.EscapeDataString(kv.Value)}"));
        return $"{authUrl}?{query}";
    }

    public async Task<TokenResponse> GetAccessTokenAsync(string authorizationCode)
    {
        var tokenEndpoint = _configuration["OAuth:TokenEndpoint"];
        var clientId = _configuration["OAuth:ClientId"];
        var clientSecret = _configuration["OAuth:ClientSecret"];
        var redirectUri = _configuration["OAuth:RedirectUri"];

        var parameters = new List<KeyValuePair<string, string>>
        {
            new("grant_type", "authorization_code"),
            new("code", authorizationCode),
            new("redirect_uri", redirectUri),
            new("client_id", clientId),
            new("client_secret", clientSecret)
        };

        var response = await _httpClient.PostAsync(tokenEndpoint, new FormUrlEncodedContent(parameters));
        response.EnsureSuccessStatusCode();

        var json = await response.Content.ReadAsStringAsync();
        return JsonSerializer.Deserialize<TokenResponse>(json);
    }

    public async Task<TokenResponse> RefreshTokenAsync(string refreshToken)
    {
        var tokenEndpoint = _configuration["OAuth:TokenEndpoint"];
        var clientId = _configuration["OAuth:ClientId"];
        var clientSecret = _configuration["OAuth:ClientSecret"];

        var parameters = new List<KeyValuePair<string, string>>
        {
            new("grant_type", "refresh_token"),
            new("refresh_token", refreshToken),
            new("client_id", clientId),
            new("client_secret", clientSecret)
        };

        var response = await _httpClient.PostAsync(tokenEndpoint, new FormUrlEncodedContent(parameters));
        response.EnsureSuccessStatusCode();

        var json = await response.Content.ReadAsStringAsync();
        return JsonSerializer.Deserialize<TokenResponse>(json);
    }

    public async Task<ClaimsPrincipal> GetUserInfoAsync(string accessToken)
    {
        var userInfoEndpoint = _configuration["OAuth:UserInfoEndpoint"];

        _httpClient.DefaultRequestHeaders.Authorization =
            new AuthenticationHeaderValue("Bearer", accessToken);

        var response = await _httpClient.GetAsync(userInfoEndpoint);
        response.EnsureSuccessStatusCode();

        var json = await response.Content.ReadAsStringAsync();
        var userInfo = JsonDocument.Parse(json);

        var claims = new List<Claim>();

        foreach (var property in userInfo.RootElement.EnumerateObject())
        {
            claims.Add(new Claim(property.Name, property.Value.ToString()));
        }

        var identity = new ClaimsIdentity(claims, "OAuth");
        return new ClaimsPrincipal(identity);
    }
}

// Controllers/AuthController.cs
[ApiController]
[Route("api/[controller]")]
public class AuthController : ControllerBase
{
    private readonly IOAuthService _oauthService;
    private readonly ILogger<AuthController> _logger;

    public AuthController(IOAuthService oauthService, ILogger<AuthController> logger)
    {
        _oauthService = oauthService;
        _logger = logger;
    }

    [HttpGet("login")]
    public IActionResult Login()
    {
        var state = Guid.NewGuid().ToString();
        HttpContext.Session.SetString("oauth_state", state);

        var authUrl = _oauthService.GenerateAuthorizationUrl(state);
        return Redirect(authUrl);
    }

    [HttpGet("callback")]
    public async Task<IActionResult> Callback([FromQuery] string code, [FromQuery] string state)
    {
        var savedState = HttpContext.Session.GetString("oauth_state");
        if (state != savedState)
        {
            return BadRequest("Invalid state parameter");
        }

        try
        {
            var tokenResponse = await _oauthService.GetAccessTokenAsync(code);
            var userInfo = await _oauthService.GetUserInfoAsync(tokenResponse.AccessToken);

            // Store tokens securely (consider using secure cookies or session)
            HttpContext.Session.SetString("access_token", tokenResponse.AccessToken);
            HttpContext.Session.SetString("refresh_token", tokenResponse.RefreshToken ?? "");

            // Sign in the user
            await HttpContext.SignInAsync(CookieAuthenticationDefaults.AuthenticationScheme, userInfo);

            return Redirect("/dashboard");
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "OAuth callback failed");
            return Redirect("/login?error=oauth_failed");
        }
    }

    [HttpPost("refresh")]
    [Authorize]
    public async Task<ActionResult<TokenResponse>> RefreshToken()
    {
        var refreshToken = HttpContext.Session.GetString("refresh_token");
        if (string.IsNullOrEmpty(refreshToken))
        {
            return BadRequest("No refresh token available");
        }

        try
        {
            var tokenResponse = await _oauthService.RefreshTokenAsync(refreshToken);

            // Update stored tokens
            HttpContext.Session.SetString("access_token", tokenResponse.AccessToken);
            if (!string.IsNullOrEmpty(tokenResponse.RefreshToken))
            {
                HttpContext.Session.SetString("refresh_token", tokenResponse.RefreshToken);
            }

            return Ok(tokenResponse);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Token refresh failed");
            return BadRequest("Token refresh failed");
        }
    }

    [HttpPost("logout")]
    [Authorize]
    public async Task<IActionResult> Logout()
    {
        HttpContext.Session.Clear();
        await HttpContext.SignOutAsync(CookieAuthenticationDefaults.AuthenticationScheme);
        return Ok("Logged out successfully");
    }
}

// Middleware/TokenRefreshMiddleware.cs
public class TokenRefreshMiddleware
{
    private readonly RequestDelegate _next;
    private readonly IOAuthService _oauthService;

    public TokenRefreshMiddleware(RequestDelegate next, IOAuthService oauthService)
    {
        _next = next;
        _oauthService = oauthService;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        if (context.User.Identity.IsAuthenticated)
        {
            var accessToken = context.Session.GetString("access_token");
            if (!string.IsNullOrEmpty(accessToken))
            {
                // Check if token is about to expire (implement token validation logic)
                var refreshToken = context.Session.GetString("refresh_token");
                if (!string.IsNullOrEmpty(refreshToken))
                {
                    try
                    {
                        var newTokens = await _oauthService.RefreshTokenAsync(refreshToken);
                        context.Session.SetString("access_token", newTokens.AccessToken);
                        if (!string.IsNullOrEmpty(newTokens.RefreshToken))
                        {
                            context.Session.SetString("refresh_token", newTokens.RefreshToken);
                        }
                    }
                    catch
                    {
                        // Token refresh failed, redirect to login
                        context.Response.Redirect("/login");
                        return;
                    }
                }
            }
        }

        await _next(context);
    }
}

// appsettings.json configuration
{
  "OAuth": {
    "ClientId": "your-client-id",
    "ClientSecret": "your-client-secret",
    "AuthorizationEndpoint": "https://provider.com/oauth/authorize",
    "TokenEndpoint": "https://provider.com/oauth/token",
    "UserInfoEndpoint": "https://provider.com/oauth/userinfo",
    "RedirectUri": "https://localhost:5001/api/auth/callback",
    "Scope": "openid profile email"
  }
}
```
