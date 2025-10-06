## 50. JWT Tokens

### Introduction

JSON Web Tokens (JWT) are a compact, URL-safe means of representing claims to be transferred between two parties. JWTs are commonly used for authentication and information exchange in modern web applications and APIs.

## Official Definition

JWT is an open standard (RFC 7519) that defines a compact and self-contained way for securely transmitting information between parties as a JSON object. This information can be verified and trusted because it is digitally signed using a secret (HMAC) or a public/private key pair (RSA or ECDSA).

### Usage & Setup

```bash
# Install JWT packages
dotnet add package System.IdentityModel.Tokens.Jwt
dotnet add package Microsoft.AspNetCore.Authentication.JwtBearer
dotnet add package Microsoft.IdentityModel.Tokens
```

```csharp
// Program.cs - JWT Configuration
using Microsoft.AspNetCore.Authentication.JwtBearer;
using Microsoft.IdentityModel.Tokens;
using System.Text;

var builder = WebApplication.CreateBuilder(args);

// JWT Configuration
var jwtSettings = builder.Configuration.GetSection("JwtSettings");
var secretKey = jwtSettings["SecretKey"];
var issuer = jwtSettings["Issuer"];
var audience = jwtSettings["Audience"];

builder.Services.AddAuthentication(options =>
{
    options.DefaultAuthenticateScheme = JwtBearerDefaults.AuthenticationScheme;
    options.DefaultChallengeScheme = JwtBearerDefaults.AuthenticationScheme;
})
.AddJwtBearer(options =>
{
    options.TokenValidationParameters = new TokenValidationParameters
    {
        ValidateIssuer = true,
        ValidateAudience = true,
        ValidateLifetime = true,
        ValidateIssuerSigningKey = true,
        ValidIssuer = issuer,
        ValidAudience = audience,
        IssuerSigningKey = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(secretKey)),
        ClockSkew = TimeSpan.Zero // Remove default 5-minute tolerance
    };

    options.Events = new JwtBearerEvents
    {
        OnAuthenticationFailed = context =>
        {
            if (context.Exception.GetType() == typeof(SecurityTokenExpiredException))
            {
                context.Response.Headers.Add("Token-Expired", "true");
            }
            return Task.CompletedTask;
        }
    };
});

var app = builder.Build();
app.UseAuthentication();
app.UseAuthorization();
```

### Use Cases

- Stateless API authentication
- Single Sign-On (SSO) implementations
- Microservices security
- Mobile application authentication
- Cross-domain authentication
- Information exchange between services
- Session management alternative

## When to Use vs When Not to Use

**Use When:**

- Building stateless APIs
- Implementing distributed systems
- Need cross-domain authentication
- Mobile/SPA applications
- Microservices architecture
- Performance is critical

**Don't Use When:**

- Sensitive data needs to be stored
- Tokens can't be revoked easily
- Network security is poor
- Simple session-based auth suffices
- Real-time revocation is required
- Token size becomes problematic

### Market Alternatives & Adoption

### Alternatives:

- Session-based authentication
- OAuth 2.0 access tokens
- SAML tokens
- API keys
- Cookies with server sessions

**Market Position:** De facto standard for modern API authentication, adopted by major platforms and frameworks worldwide.

### Pros and Cons

### Pros:

- Stateless and scalable
- Self-contained with claims
- Cross-domain support
- JSON format (widely supported)
- Compact size
- Cryptographically signed

### Cons:

- Cannot be revoked easily
- Token size can grow large
- Secret key management critical
- No built-in refresh mechanism
- Clock synchronization issues
- Base64 encoding (not encryption)

### Sample Implementation

````csharp
// Models/JwtModels.cs
public class JwtSettings
{
    public string SecretKey { get; set; }
    public string Issuer { get; set; }
    public string Audience { get; set; }
    public int ExpirationMinutes { get; set; }
    public int RefreshTokenExpirationDays { get; set; }
}

public class LoginRequest
{
    [Required]
    [EmailAddress]
    public string Email { get; set; }

    [Required]
    public string Password { get; set; }
}

public class AuthResponse
{
    public string AccessToken { get; set; }
    public string RefreshToken { get; set; }
    public DateTime Expiration { get; set; }
    public UserInfo User { get; set; }
}

public class UserInfo
{
    public string Id { get; set; }
    public string Email { get; set; }
    public string FirstName { get; set; }
    public string LastName { get; set; }
    public List<string> Roles { get; set; }
}

public class RefreshToken
{
    public string Id { get; set; }
    public string UserId { get; set; }
    public string Token { get; set; }
    public DateTime ExpiryDate { get; set; }
    public bool IsRevoked { get; set; }
    public DateTime CreatedDate { get; set; }
    public string CreatedByIp { get; set; }
    public DateTime? RevokedDate { get; set; }
    public string RevokedByIp { get; set; }
    public string ReplacedByToken { get; set; }
}

// Services/IJwtService.cs
public interface IJwtService
{
    string GenerateAccessToken(ApplicationUser user, IList<string> roles);
    RefreshToken GenerateRefreshToken(string userId, string ipAddress);
    ClaimsPrincipal ValidateToken(string token);
    Task<AuthResponse> RefreshTokenAsync(string token, string ipAddress);
    Task RevokeTokenAsync(string token, string ipAddress);
    Task<bool> IsTokenValidAsync(string token);
}

// Services/JwtService.cs
public class JwtService : IJwtService
{
    private readonly JwtSettings _jwtSettings;
    private readonly UserManager<ApplicationUser> _userManager;
    private readonly ApplicationDbContext _context;
    private readonly ILogger<JwtService> _logger;

    public JwtService(
        IOptions<JwtSettings> jwtSettings,
        UserManager<ApplicationUser> userManager,
        ApplicationDbContext context,
        ILogger<JwtService> logger)
    {
        _jwtSettings = jwtSettings.Value;
        _userManager = userManager;
        _context = context;
        _logger = logger;
    }

    public string GenerateAccessToken(ApplicationUser user, IList<string> roles)
    {
        var claims = new List<Claim>
        {
            new Claim(JwtRegisteredClaimNames.Sub, user.Id),
            new Claim(JwtRegisteredClaimNames.Email, user.Email),
            new Claim(JwtRegisteredClaimNames.Jti, Guid.NewGuid().ToString()),
            new Claim(JwtRegisteredClaimNames.Iat,
                new DateTimeOffset(DateTime.UtcNow).ToUnixTimeSeconds().ToString(),
                ClaimValueTypes.Integer64),
            new Claim("firstName", user.FirstName ?? ""),
            new Claim("lastName", user.LastName ?? ""),
            new Claim("userId", user.Id)
        };

        // Add role claims
        foreach (var role in roles)
        {
            claims.Add(new Claim(ClaimTypes.Role, role));
        }

        var key = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(_jwtSettings.SecretKey));
        var credentials = new SigningCredentials(key, SecurityAlgorithms.HmacSha256);

        var token = new JwtSecurityToken(
            issuer: _jwtSettings.Issuer,
            audience: _jwtSettings.Audience,
            claims: claims,
            expires: DateTime.UtcNow.AddMinutes(_jwtSettings.ExpirationMinutes),
            signingCredentials: credentials
        );

        return new JwtSecurityTokenHandler().WriteToken(token);
    }

    public RefreshToken GenerateRefreshToken(string userId, string ipAddress)
    {
        using var rngCryptoServiceProvider = new RNGCryptoServiceProvider();
        var randomBytes = new byte[64];
        rngCryptoServiceProvider.GetBytes(randomBytes);

        return new RefreshToken
        {
            Id = Guid.NewGuid().ToString(),
            UserId = userId,
            Token = Convert.ToBase64String(randomBytes),
            ExpiryDate = DateTime.UtcNow.AddDays(_jwtSettings.RefreshTokenExpirationDays),
            CreatedDate = DateTime.UtcNow,
            CreatedByIp = ipAddress
        };
    }

    public ClaimsPrincipal ValidateToken(string token)
    {
        var tokenHandler = new JwtSecurityTokenHandler();
        var key = Encoding.UTF8.GetBytes(_jwtSettings.SecretKey);

        try
        {
            var principal = tokenHandler.ValidateToken(token, new TokenValidationParameters
            {
                ValidateIssuerSigningKey = true,
                IssuerSigningKey = new SymmetricSecurityKey(key),
                ValidateIssuer = true,
                ValidIssuer = _jwtSettings.Issuer,
                ValidateAudience = true,
                ValidAudience = _jwtSettings.Audience,
                ValidateLifetime = true,
                ClockSkew = TimeSpan.Zero
            }, out SecurityToken validatedToken);

            return principal;
        }
        catch
        {
            return null;
        }
    }

    public async Task<AuthResponse> RefreshTokenAsync(string token, string ipAddress)
    {
        var refreshToken = await _context.RefreshTokens
            .FirstOrDefaultAsync(rt => rt.Token == token && !rt.IsRevoked);

        if (refreshToken == null || refreshToken.ExpiryDate <= DateTime.UtcNow)
        {
            throw new SecurityTokenException("Invalid or expired refresh token");
        }

        var user = await _userManager.FindByIdAsync(refreshToken.UserId);
        if (user == null)
        {
            throw new SecurityTokenException("User not found");
        }

        // Revoke current refresh token
        refreshToken.IsRevoked = true;
        refreshToken.RevokedDate = DateTime.UtcNow;
        refreshToken.RevokedByIp = ipAddress;

        // Generate new tokens
        var roles = await _userManager.GetRolesAsync(user);
        var newAccessToken = GenerateAccessToken(user, roles);
        var newRefreshToken = GenerateRefreshToken(user.Id, ipAddress);

        // Replace old token
        refreshToken.ReplacedByToken = newRefreshToken.Token;

        _context.RefreshTokens.Add(newRefreshToken);
        await _context.SaveChangesAsync();

        return new AuthResponse
        {
            AccessToken = newAccessToken,
            RefreshToken = newRefreshToken.Token,
            Expiration = DateTime.UtcNow.AddMinutes(_jwtSettings.ExpirationMinutes),
            User = new UserInfo
            {
                Id = user.Id,
                Email = user.Email,
                FirstName = user.FirstName,
                LastName = user.LastName,
                Roles = roles.ToList()
            }
        };
    }

    public async Task RevokeTokenAsync(string token, string ipAddress)
    {
        var refreshToken = await _context.RefreshTokens
            .FirstOrDefaultAsync(rt => rt.Token == token);

        if (refreshToken == null)
        {
            throw new SecurityTokenException("Token not found");
        }

        refreshToken.IsRevoked = true;
        refreshToken.RevokedDate = DateTime.UtcNow;
        refreshToken.RevokedByIp = ipAddress;

        await _context.SaveChangesAsync();
    }

    public async Task<bool> IsTokenValidAsync(string token)
    {
        var refreshToken = await _context.RefreshTokens
            .FirstOrDefaultAsync(rt => rt.Token == token);

        return refreshToken != null &&
               !refreshToken.IsRevoked &&
               refreshToken.ExpiryDate > DateTime.UtcNow;
    }
}

// Controllers/AuthController.cs
[ApiController]
[Route("api/[controller]")]
public class AuthController : ControllerBase
{
    private readonly IJwtService _jwtService;
    private readonly UserManager<ApplicationUser> _userManager;
    private readonly SignInManager<ApplicationUser> _signInManager;
    private readonly ILogger<AuthController> _logger;

    public AuthController(
        IJwtService jwtService,
        UserManager<ApplicationUser> userManager,
        SignInManager<ApplicationUser> signInManager,
        ILogger<AuthController> logger)
    {
        _jwtService = jwtService;
        _userManager = userManager;
        _signInManager = signInManager;
        _logger = logger;
    }

    [HttpPost("login")]
    public async Task<ActionResult<AuthResponse>> Login([FromBody] LoginRequest request)
    {
        try
        {
            var user = await _userManager.FindByEmailAsync(request.Email);
            if (user == null)
            {
                return Unauthorized(new { message = "Invalid credentials" });
            }

            var result = await _signInManager.CheckPasswordSignInAsync(user, request.Password, false);
            if (!result.Succeeded)
            {
                return Unauthorized(new { message = "Invalid credentials" });
            }

            var roles = await _userManager.GetRolesAsync(user);
            var accessToken = _jwtService.GenerateAccessToken(user, roles);
            var refreshToken = _jwtService.GenerateRefreshToken(user.Id, GetIpAddress());

            // Save refresh token to database
            var context = HttpContext.RequestServices.GetRequiredService<ApplicationDbContext>();
            context.RefreshTokens.Add(refreshToken);
            await context.SaveChangesAsync();

            var response = new AuthResponse
            {
                AccessToken = accessToken,
                RefreshToken = refreshToken.Token,
                Expiration = DateTime.UtcNow.AddMinutes(60), // Based on your JWT settings
                User = new UserInfo
                {
                    Id = user.Id,
                    Email = user.Email,
                    FirstName = user.FirstName,
                    LastName = user.LastName,
                    Roles = roles.ToList()
                }
            };

            _logger.LogInformation("User {Email} logged in successfully", user.Email);
            return Ok(response);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Login failed for {Email}", request.Email);
            return StatusCode(500, new { message = "An error occurred during login" });
        }
    }

    [HttpPost("refresh-token")]
    public async Task<ActionResult<AuthResponse>> RefreshToken([FromBody] RefreshTokenRequest request)
    {
        try
        {
            var response = await _jwtService.RefreshTokenAsync(request.RefreshToken, GetIpAddress());
            return Ok(response);
        }
        catch (SecurityTokenException ex)
        {
            return Unauthorized(new { message = ex.Message });
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Token refresh failed");
            return StatusCode(500, new { message = "An error occurred during token refresh" });
        }
    }

    [HttpPost("revoke-token")]
    [Authorize]
    public async Task<IActionResult> RevokeToken([FromBody] RevokeTokenRequest request)
    {
        try
        {
            await _jwtService.Revo// Models/JwtModels.cs
public class JwtSettings
{
    public string SecretKey { get; set; }
    public string Issuer { get; set; }
    public string Audience { get; set; }
    public int ExpirationMinutes { get; set; }
    public int RefreshTokenExpirationDays { get; set; }
}

public class LoginRequest
{
    [Required]
    [EmailAddress]
    public string Email { get; set; }

    [Required]
    public string Password { get; set; }
}

public class AuthResponse
{
    public string AccessToken { get; set; }
    public string RefreshToken { get; set; }
    public DateTime Expiration { get; set; }
    public UserInfo User { get; set; }
}

public class UserInfo
{
    public string Id { get; set; }
    public string Email { get; set; }
    public string FirstName { get; set; }
    public string LastName { get; set; }
    public List<string> Roles { get; set; }
}

public class RefreshToken
{
    public string Id { get; set; }
    public string UserId { get; set; }
    public string Token { get; set; }
    public DateTime ExpiryDate { get; set; }
    public bool IsRevoked { get; set; }
    public DateTime CreatedDate { get; set; }
    public string CreatedByIp { get; set; }
    public DateTime? RevokedDate { get; set; }
    public string RevokedByIp { get; set; }
    public string ReplacedByToken { get; set; }
}

// Services/IJwtService.cs
public interface IJwtService
{
    string GenerateAccessToken(ApplicationUser user, IList<string> roles);
    RefreshToken GenerateRefreshToken(string userId, string ipAddress);
    ClaimsPrincipal ValidateToken(string token);
    Task<AuthResponse> RefreshTokenAsync(string token, string ipAddress);
    Task RevokeTokenAsync(string token, string ipAddress);
    Task<bool> IsTokenValidAsync(string token);
}

// Services/JwtService.cs
public class JwtService : IJwtService
{
    private readonly JwtSettings _jwtSettings;
    private readonly UserManager<ApplicationUser> _userManager;
    private readonly ApplicationDbContext _context;
    private readonly ILogger<JwtService> _logger;

    public JwtService(
        IOptions<JwtSettings> jwtSettings,
        UserManager<ApplicationUser> userManager,
        ApplicationDbContext context,
        ILogger<JwtService> logger)
    {
        _jwtSettings = jwtSettings.Value;
        _userManager = userManager;
        _context = context;
        _logger = logger;
    }

    public string GenerateAccessToken(ApplicationUser user, IList<string> roles)
    {
        var claims = new List<Claim>
        {
            new Claim(JwtRegisteredClaimNames.Sub, user.Id),
            new Claim(JwtRegisteredClaimNames.Email, user.Email),
            new Claim(JwtRegisteredClaimNames.Jti, Guid.NewGuid().ToString()),
            new Claim(JwtRegisteredClaimNames.Iat,
                new DateTimeOffset(DateTime.UtcNow).ToUnixTimeSeconds().ToString(),
                ClaimValueTypes.Integer64),
            new Claim("firstName", user.FirstName ?? ""),
            new Claim("lastName", user.LastName ?? ""),
            new Claim("userId", user.Id)
        };

        // Add role claims
        foreach (var role in roles)
        {
            claims.Add(new Claim(ClaimTypes.Role, role));
        }

        var key = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(_jwtSettings.SecretKey));
        var credentials = new SigningCredentials(key, SecurityAlgorithms.HmacSha256);

        var token = new JwtSecurityToken(
            issuer: _jwtSettings.Issuer,
            audience: _jwtSettings.Audience,
            claims: claims,
            expires: DateTime.UtcNow.AddMinutes(_jwtSettings.ExpirationMinutes),
            signingCredentials: credentials
        );

        return new JwtSecurityTokenHandler().WriteToken(token);
    }

    public RefreshToken GenerateRefreshToken(string userId, string ipAddress)
    {
        using var rngCryptoServiceProvider = new RNGCryptoServiceProvider();
        var randomBytes = new byte[64];
        rngCryptoServiceProvider.GetBytes(randomBytes);

        return new RefreshToken
        {
            Id = Guid.NewGuid().ToString(),
            UserId = userId,
            Token = Convert.ToBase64String(randomBytes),
            ExpiryDate = DateTime.UtcNow.AddDays(_jwtSettings.RefreshTokenExpirationDays),
            CreatedDate = DateTime.UtcNow,
            CreatedByIp = ipAddress
        };
    }

    public ClaimsPrincipal ValidateToken(string token)
    {
        var tokenHandler = new JwtSecurityTokenHandler();
        var key = Encoding.UTF8.GetBytes(_jwtSettings.SecretKey);

        try
        {
            var principal = tokenHandler.ValidateToken(token, new TokenValidationParameters
            {
                ValidateIssuerSigningKey = true,
                IssuerSigningKey = new SymmetricSecurityKey(key),
                ValidateIssuer = true,
                ValidIssuer = _jwtSettings.Issuer,
                ValidateAudience = true,
                ValidAudience = _jwtSettings.Audience,
                ValidateLifetime = true,
                ClockSkew = TimeSpan.Zero
            }, out SecurityToken validatedToken);

            return principal;
        }
        catch
        {
            return null;
        }
    }

    public async Task<AuthResponse> RefreshTokenAsync(string token, string ipAddress)
    {
        var refreshToken = await _context.RefreshTokens
            .FirstOrDefaultAsync(rt => rt.Token == token && !rt.IsRevoked);

        if (refreshToken == null || refreshToken.ExpiryDate <= DateTime.UtcNow)
        {
            throw new SecurityTokenException("Invalid or expired refresh token");
        }

        var user = await _userManager.FindByIdAsync(refreshToken.UserId);
        if (user == null)
        {
            throw new SecurityTokenException("User not found");
        }

        // Revoke current refresh token
        refreshToken.IsRevoked = true;
        refreshToken.RevokedDate = DateTime.UtcNow;
        refreshToken.RevokedByIp = ipAddress;

        // Generate new tokens
        var roles = await _userManager.GetRolesAsync(user);
        var newAccessToken = GenerateAccessToken(user, roles);
        var newRefreshToken = GenerateRefreshToken(user.Id, ipAddress);

        // Replace old token
        refreshToken.ReplacedByToken = newRefreshToken.Token;

        _context.RefreshTokens.Add(newRefreshToken);
        await _context.SaveChangesAsync();

        return new AuthResponse
        {
            AccessToken = newAccessToken,
            RefreshToken = newRefreshToken.Token,
            Expiration = DateTime.UtcNow.AddMinutes(_jwtSettings.ExpirationMinutes),
            User = new UserInfo
            {
                Id = user.Id,
                Email = user.Email,
                FirstName = user.FirstName,
                LastName = user.LastName,
                Roles = roles.ToList()
            }
        };
    }

    public async Task RevokeTokenAsync(string token, string ipAddress)
    {
        var refreshToken = await _context.RefreshTokens
            .FirstOrDefaultAsync(rt => rt.Token == token);

        if (refreshToken == null)
        {
            throw new SecurityTokenException("Token not found");
        }

        refreshToken.IsRevoked = true;
        refreshToken.RevokedDate = DateTime.UtcNow;
        refreshToken.RevokedByIp = ipAddress;

        await _context.SaveChangesAsync();
    }

    public async Task<bool> IsTokenValidAsync(string token)
    {
        var refreshToken = await _context.RefreshTokens
            .FirstOrDefaultAsync(rt => rt.Token == token);

        return refreshToken != null &&
               !refreshToken.IsRevoked &&
               refreshToken.ExpiryDate > DateTime.UtcNow;
    }
}

// Controllers/AuthController.cs
[ApiController]
[Route("api/[controller]")]
public class AuthController : ControllerBase
{
    private readonly IJwtService _jwtService;
    private readonly UserManager<ApplicationUser> _userManager;
    private readonly SignInManager<ApplicationUser> _signInManager;
    private readonly ILogger<AuthController> _logger;

    public AuthController(
        IJwtService jwtService,
        UserManager<ApplicationUser> userManager,
        SignInManager<ApplicationUser> signInManager,
        ILogger<AuthController> logger)
    {
        _jwtService = jwtService;
        _userManager = userManager;
        _signInManager = signInManager;
        _logger = logger;
    }

    [HttpPost("login")]
    public async Task<ActionResult<AuthResponse>> Login([FromBody] LoginRequest request)
    {
        try
        {
            var user = await _userManager.FindByEmailAsync(request.Email);
            if (user == null)
            {
                return Unauthorized(new { message = "Invalid credentials" });
            }

            var result = await _signInManager.CheckPasswordSignInAsync(user, request.Password, false);
            if (!result.Succeeded)
            {
                return Unauthorized(new { message = "Invalid credentials" });
            }

            var roles = await _userManager.GetRolesAsync(user);
            var accessToken = _jwtService.GenerateAccessToken(user, roles);
            var refreshToken = _jwtService.GenerateRefreshToken(user.Id, GetIpAddress());

            // Save refresh token to database
            var context = HttpContext.RequestServices.GetRequiredService<ApplicationDbContext>();
            context.RefreshTokens.Add(refreshToken);
            await context.SaveChangesAsync();

            var response = new AuthResponse
            {
                AccessToken = accessToken,
                RefreshToken = refreshToken.Token,
                Expiration = DateTime.UtcNow.AddMinutes(60), // Based on your JWT settings
                User = new UserInfo
                {
                    Id = user.Id,
                    Email = user.Email,
                    FirstName = user.FirstName,
                    LastName = user.LastName,
                    Roles = roles.ToList()
                }
            };

            _logger.LogInformation("User {Email} logged in successfully", user.Email);
            return Ok(response);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Login failed for {Email}", request.Email);
            return StatusCode(500, new { message = "An error occurred during login" });
        }
    }

    [HttpPost("refresh-token")]
    public async Task<ActionResult<AuthResponse>> RefreshToken([FromBody] RefreshTokenRequest request)
    {
        try
        {
            var response = await _jwtService.RefreshTokenAsync(request.RefreshToken, GetIpAddress());
            return Ok(response);
        }
        catch (SecurityTokenException ex)
        {
            return Unauthorized(new { message = ex.Message });
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Token refresh failed");
            return StatusCode(500, new { message = "An error occurred during token refresh" });
        }
    }

    [HttpPost("revoke-token")]
    [Authorize]
    public async Task<IActionResult> RevokeToken([FromBody] RevokeTokenRequest request)
    {
        try
        {
            await _jwtService.Revo

Similar code found with 4 license types

sreee2001: please continue from after

 [HttpPost("refresh-token")]
    public async Task<ActionResult<AuthResponse>> RefreshToken([FromBody] RefreshTokenRequest request)
    {
        try
        {
            var response = await _jwtService.RefreshTokenAsync(request.RefreshToken, GetIpAddress());
            return Ok(response);
        }
        catch (SecurityTokenException ex)
        {
            return Unauthorized(new { message = ex.Message });
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Token refresh failed");
            return StatusCode(500, new { message = "An error occurred during token refresh" });
        }
    }

    [HttpPost("revoke-token")]
    [Authorize]
    public async Task<IActionResult> RevokeToken([FromBody] RevokeTokenRequest request)
    {
        try
        {
            await _jwtService.Revo

------------
if needed please start from after  stable sub section. You were in the middle of Sample Implementation of 50. JWT Tokens
- if you think you will run out of response limit. pause at a stable point and print --continue-- and keep the state  in mind , so when I ask you to continue, you can continue the task from that checkpoint

GitHub Copilot: I'll continue from where the JWT Tokens sample implementation left off, completing the controller and then moving to the remaining sections.

## 50. JWT Tokens (continued)

```csharp
    [HttpPost("revoke-token")]
    [Authorize]
    public async Task<IActionResult> RevokeToken([FromBody] RevokeTokenRequest request)
    {
        try
        {
            await _jwtService.RevokeTokenAsync(request.RefreshToken);
            return Ok(new { message = "Token revoked successfully" });
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Token revocation failed");
            return BadRequest(new { message = "Token revocation failed" });
        }
    }

    private string GetIpAddress()
    {
        return HttpContext.Connection.RemoteIpAddress?.MapToIPv4().ToString() ?? "Unknown";
    }
}

// Models/DTOs
public class LoginRequest
{
    public string Email { get; set; } = string.Empty;
    public string Password { get; set; } = string.Empty;
}

public class AuthResponse
{
    public string AccessToken { get; set; } = string.Empty;
    public string RefreshToken { get; set; } = string.Empty;
    public DateTime ExpiresAt { get; set; }
    public UserDto User { get; set; } = null!;
}

public class RefreshTokenRequest
{
    public string RefreshToken { get; set; } = string.Empty;
}

public class RevokeTokenRequest
{
    public string RefreshToken { get; set; } = string.Empty;
}

// JWT Configuration in appsettings.json
{
  "JwtSettings": {
    "SecretKey": "your-super-secret-key-that-is-at-least-256-bits-long",
    "Issuer": "HotelManagement.Api",
    "Audience": "HotelManagement.Client",
    "ExpirationInMinutes": 15,
    "RefreshTokenExpirationInDays": 7
  }
}

// Program.cs configuration
builder.Services.Configure<JwtSettings>(builder.Configuration.GetSection("JwtSettings"));

builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        var jwtSettings = builder.Configuration.GetSection("JwtSettings").Get<JwtSettings>();
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuerSigningKey = true,
            IssuerSigningKey = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(jwtSettings.SecretKey)),
            ValidateIssuer = true,
            ValidIssuer = jwtSettings.Issuer,
            ValidateAudience = true,
            ValidAudience = jwtSettings.Audience,
            ValidateLifetime = true,
            ClockSkew = TimeSpan.Zero
        };
    });
````

**NuGet Packages Required:**

```xml
<PackageReference Include="Microsoft.AspNetCore.Authentication.JwtBearer" Version="8.0.0" />
<PackageReference Include="System.IdentityModel.Tokens.Jwt" Version="7.0.3" />
```
