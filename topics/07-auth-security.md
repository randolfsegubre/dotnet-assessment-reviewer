# 07 — Authentication, Authorization & Security

> 🔗 **BudgetPH reference**: [AuthController.cs](https://github.com/randolfsegubre/BudgetPH/blob/master/src/Presentation/FinanceManager.API/Controllers/AuthController.cs) | [Program.cs](https://github.com/randolfsegubre/BudgetPH/blob/master/src/Presentation/FinanceManager.API/Program.cs)

---

## Table of Contents
1. [JWT — JSON Web Tokens](#1-jwt--json-web-tokens)
2. [OAuth2 & OpenID Connect](#2-oauth2--openid-connect)
3. [ASP.NET Core Identity](#3-aspnet-core-identity)
4. [Claims-Based Authorization](#4-claims-based-authorization)
5. [Policy-Based Authorization](#5-policy-based-authorization)
6. [OWASP Top 10](#6-owasp-top-10)
7. [Input Validation & Sanitization](#7-input-validation--sanitization)
8. [CORS](#8-cors)
9. [Secrets Management](#9-secrets-management)
10. [Common Interview Questions](#10-common-interview-questions)

---

## 1. JWT — JSON Web Tokens

### JWT Structure
A JWT has 3 base64url-encoded parts separated by dots:

```
Header.Payload.Signature

eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9       ← Header (algorithm + type)
.eyJzdWIiOiJ1c2VyLWlkLWhlcmUiLCJlbWFp...   ← Payload (claims)
.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c  ← Signature (HMAC/RSA)
```

**Header:**
```json
{ "alg": "HS256", "typ": "JWT" }
```

**Payload (claims):**
```json
{
  "sub": "user-guid-here",       // Subject (user ID)
  "email": "user@example.com",
  "role": "Admin",
  "iss": "BudgetPH",             // Issuer
  "aud": "BudgetPH",             // Audience
  "exp": 1751000000,             // Expiry (Unix timestamp)
  "iat": 1750996400              // Issued at
}
```

**Signature:** `HMACSHA256(base64(header) + "." + base64(payload), secretKey)`

### 🔗 BudgetPH JWT Setup

From [Program.cs](https://github.com/randolfsegubre/BudgetPH/blob/master/src/Presentation/FinanceManager.API/Program.cs):
```csharp
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
.AddJwtBearer(options =>
{
    options.TokenValidationParameters = new TokenValidationParameters
    {
        ValidateIssuer = true,
        ValidateAudience = true,
        ValidateLifetime = true,              // check exp claim
        ValidateIssuerSigningKey = true,       // verify signature
        ValidIssuer = jwt["Issuer"],
        ValidAudience = jwt["Audience"],
        IssuerSigningKey = new SymmetricSecurityKey(
            Encoding.UTF8.GetBytes(jwt["SecretKey"]!)),
        ClockSkew = TimeSpan.Zero              // no tolerance for expired tokens
    };
});
```

### Generating a JWT

```csharp
public string GenerateJwtToken(ApplicationUser user, IList<string> roles)
{
    var claims = new List<Claim>
    {
        new("sub", user.Id),
        new(ClaimTypes.Email, user.Email!),
        new(JwtRegisteredClaimNames.Jti, Guid.NewGuid().ToString()),
    };
    
    foreach (var role in roles)
        claims.Add(new Claim(ClaimTypes.Role, role));

    var key = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(_jwtSettings.SecretKey));
    var creds = new SigningCredentials(key, SecurityAlgorithms.HmacSha256);

    var token = new JwtSecurityToken(
        issuer: _jwtSettings.Issuer,
        audience: _jwtSettings.Audience,
        claims: claims,
        expires: DateTime.UtcNow.AddMinutes(_jwtSettings.ExpiryMinutes),
        signingCredentials: creds
    );

    return new JwtSecurityTokenHandler().WriteToken(token);
}
```

### Refresh Token Pattern

```csharp
// Refresh token is a long-lived opaque token stored in DB
public async Task<AuthResponse> RefreshAsync(string refreshToken)
{
    // 1. Find the refresh token in DB
    var stored = await context.RefreshTokens
        .Include(r => r.User)
        .FirstOrDefaultAsync(r => r.Token == refreshToken);

    // 2. Validate: exists, not expired, not revoked
    if (stored is null || stored.ExpiresAt < DateTime.UtcNow || stored.IsRevoked)
        throw new UnauthorizedException("Invalid refresh token");

    // 3. Revoke old token (rotation strategy)
    stored.IsRevoked = true;
    stored.RevokedAt = DateTime.UtcNow;

    // 4. Generate new access token + new refresh token
    var newAccessToken = GenerateJwtToken(stored.User, await GetRolesAsync(stored.User));
    var newRefreshToken = GenerateRefreshToken();

    context.RefreshTokens.Add(new RefreshToken
    {
        Token = newRefreshToken,
        UserId = stored.UserId,
        ExpiresAt = DateTime.UtcNow.AddDays(30),
        CreatedAt = DateTime.UtcNow
    });

    await context.SaveChangesAsync();
    return new AuthResponse(newAccessToken, newRefreshToken);
}

private string GenerateRefreshToken()
{
    var bytes = new byte[64];
    RandomNumberGenerator.Fill(bytes); // cryptographically secure
    return Convert.ToBase64String(bytes);
}
```

---

## 2. OAuth2 & OpenID Connect

### OAuth2 Flows

| Flow | Use Case |
|---|---|
| **Authorization Code + PKCE** | Web apps, mobile apps (most secure) |
| **Client Credentials** | Machine-to-machine (no user involved) |
| **Implicit** | ❌ Deprecated — tokens in URL fragment |
| **Resource Owner Password** | ❌ Legacy — sends username/password to client |

### Authorization Code + PKCE Flow

```
1. User clicks "Login with Google"
2. App generates code_verifier (random) and code_challenge = SHA256(code_verifier)
3. App redirects to: https://accounts.google.com/oauth/authorize?
      client_id=...&redirect_uri=...&response_type=code
      &code_challenge=...&code_challenge_method=S256
4. User authenticates at Google
5. Google redirects back: https://yourapp.com/callback?code=AUTH_CODE
6. App exchanges: POST https://oauth2.googleapis.com/token
      { code: AUTH_CODE, code_verifier: CODE_VERIFIER, ... }
7. Google returns: access_token + id_token + refresh_token
8. App verifies id_token (JWT) → extracts user identity
```

**PKCE** (Proof Key for Code Exchange) prevents authorization code interception attacks — the `code_verifier` proves the entity that started the flow is the same one completing it.

### OpenID Connect (OIDC)
OIDC is an **identity layer on top of OAuth2**. OAuth2 handles *authorization* (what can you do?); OIDC handles *authentication* (who are you?). OIDC adds the `id_token` (a JWT) and the `/userinfo` endpoint.

---

## 3. ASP.NET Core Identity

Identity manages users, passwords, roles, claims, and tokens.

```csharp
// Setup
builder.Services.AddIdentity<IdentityUser, IdentityRole>(options =>
{
    options.Password.RequiredLength = 8;
    options.Password.RequireUppercase = true;
    options.Password.RequireDigit = true;
    options.Password.RequireNonAlphanumeric = false;
    options.User.RequireUniqueEmail = true;
    options.Lockout.MaxFailedAccessAttempts = 5;
    options.Lockout.DefaultLockoutTimeSpan = TimeSpan.FromMinutes(15);
})
.AddEntityFrameworkStores<ApplicationDbContext>()
.AddDefaultTokenProviders();

// Registration
public async Task<AuthResponse> RegisterAsync(RegisterRequest request)
{
    var identityUser = new IdentityUser
    {
        UserName = request.Email,
        Email = request.Email
    };

    var result = await _userManager.CreateAsync(identityUser, request.Password);
    
    if (!result.Succeeded)
        throw new ValidationException(result.Errors.Select(e => e.Description));

    await _userManager.AddToRoleAsync(identityUser, "User");
    
    // Also create ApplicationUser (domain entity)
    var appUser = new ApplicationUser
    {
        IdentityUserId = identityUser.Id,
        FirstName = request.FirstName,
        LastName = request.LastName,
        Email = request.Email
    };
    context.AppUsers.Add(appUser);
    await context.SaveChangesAsync();

    return GenerateAuthResponse(identityUser, appUser);
}

// Login
public async Task<AuthResponse> LoginAsync(LoginRequest request)
{
    var identityUser = await _userManager.FindByEmailAsync(request.Email);
    
    if (identityUser is null || !await _userManager.CheckPasswordAsync(identityUser, request.Password))
        throw new UnauthorizedException("Invalid credentials");

    if (await _userManager.IsLockedOutAsync(identityUser))
        throw new UnauthorizedException("Account is locked");

    await _userManager.ResetAccessFailedCountAsync(identityUser);
    
    var roles = await _userManager.GetRolesAsync(identityUser);
    return GenerateAuthResponse(identityUser, roles);
}
```

---

## 4. Claims-Based Authorization

### 🔗 BudgetPH BaseApiController Pattern

From [BaseApiController.cs](https://github.com/randolfsegubre/BudgetPH/blob/master/src/Presentation/FinanceManager.API/Controllers/BaseApiController.cs):
```csharp
protected string CurrentUserId =>
    User.FindFirstValue("sub") ?? User.FindFirstValue(ClaimTypes.NameIdentifier)
        ?? throw new UnauthorizedException();
```

### Working with Claims

```csharp
// Reading claims from JWT
var userId = User.FindFirstValue("sub");
var email = User.FindFirstValue(ClaimTypes.Email);
var roles = User.FindAll(ClaimTypes.Role).Select(c => c.Value);
var isAdmin = User.IsInRole("Admin");

// Custom claim authorization
[Authorize]
[HttpGet("premium-report")]
public async Task<IActionResult> GetPremiumReport()
{
    var tier = User.FindFirstValue("subscription_tier");
    if (tier != "Premium")
        return Forbid(); // 403
    
    // proceed
}
```

---

## 5. Policy-Based Authorization

```csharp
// Define policies in Program.cs
builder.Services.AddAuthorization(options =>
{
    // Simple role policy
    options.AddPolicy("AdminOnly", policy => 
        policy.RequireRole("Admin"));
    
    // Multiple roles
    options.AddPolicy("ModeratorOrAdmin", policy => 
        policy.RequireRole("Admin", "Moderator"));
    
    // Custom claim requirement
    options.AddPolicy("PremiumUser", policy => 
        policy.RequireClaim("subscription_tier", "Premium"));
    
    // Complex custom requirement
    options.AddPolicy("CanDeleteAccount", policy =>
        policy.Requirements.Add(new AccountOwnerRequirement()));
});

// Custom requirement
public class AccountOwnerRequirement : IAuthorizationRequirement { }

// Custom handler
public class AccountOwnerHandler : AuthorizationHandler<AccountOwnerRequirement>
{
    private readonly ApplicationDbContext _context;
    private readonly IHttpContextAccessor _httpContext;

    protected override async Task HandleRequirementAsync(
        AuthorizationHandlerContext context,
        AccountOwnerRequirement requirement)
    {
        var userId = context.User.FindFirstValue("sub");
        var routeData = _httpContext.HttpContext!.GetRouteData();
        var accountId = Guid.Parse(routeData.Values["id"]?.ToString() ?? "");

        var account = await _context.Accounts
            .AsNoTracking()
            .FirstOrDefaultAsync(a => a.Id == accountId);

        if (account?.UserId.ToString() == userId)
            context.Succeed(requirement);
        else
            context.Fail(); // 403
    }
}

// Usage on controller
[Authorize(Policy = "AdminOnly")]
[HttpDelete("{id:guid}")]
[Authorize(Policy = "CanDeleteAccount")]
public async Task<IActionResult> Delete(Guid id, CancellationToken ct) { ... }
```

---

## 6. OWASP Top 10

### A01: Broken Access Control
**Prevention:** Check authorization on every request. Never trust client-supplied IDs.
```csharp
// ❌ VULNERABLE — trusts client-supplied userId
[HttpGet("accounts")]
public async Task<IActionResult> GetAccounts([FromQuery] Guid userId)
    => Ok(await context.Accounts.Where(a => a.UserId == userId).ToListAsync());

// ✅ SECURE — use authenticated user's ID from JWT
[Authorize]
[HttpGet("accounts")]
public async Task<IActionResult> GetAccounts(CancellationToken ct)
{
    var appUser = await GetAppUserAsync(ct);
    return Ok(await context.Accounts.Where(a => a.UserId == appUser!.Id).ToListAsync(ct));
}
```

### A02: Cryptographic Failures
```csharp
// ❌ VULNERABLE — MD5/SHA1 for passwords
var hash = MD5.HashData(Encoding.UTF8.GetBytes(password));

// ✅ SECURE — bcrypt via ASP.NET Core Identity (handles salt + work factor)
await _userManager.CreateAsync(user, password); // uses PasswordHasher<T> internally

// ❌ VULNERABLE — weak JWT secret
"SecretKey": "abc123"

// ✅ SECURE — minimum 256-bit key (32 bytes for HS256)
"SecretKey": "xK8mP3qR9nV2wA5jH7bT1yL4cF6dE0iG" // cryptographically random, 32+ chars
```

### A03: Injection (SQL Injection)
```csharp
// ❌ VULNERABLE — string concatenation
var sql = $"SELECT * FROM Users WHERE Email = '{email}'";
// email = "'; DROP TABLE Users; --"

// ✅ SECURE — parameterized queries
var user = await context.Users
    .Where(u => u.Email == email)  // EF Core — always parameterized
    .FirstOrDefaultAsync();

// ✅ SECURE — raw SQL with parameters
var user = await context.Users
    .FromSqlInterpolated($"SELECT * FROM Users WHERE Email = {email}")
    .FirstOrDefaultAsync();
```

### A04: Insecure Design
- Use threat modeling during design phase
- Implement rate limiting on authentication endpoints
- Use CAPTCHA for brute force prevention

### A05: Security Misconfiguration
```csharp
// ✅ Hide sensitive error details in production
if (!app.Environment.IsDevelopment())
{
    app.UseExceptionHandler("/error"); // generic error page
    app.UseHsts();
}

// ✅ Remove server header
builder.WebHost.ConfigureKestrel(options =>
    options.AddServerHeader = false);

// ✅ Security headers
app.Use(async (context, next) =>
{
    context.Response.Headers["X-Content-Type-Options"] = "nosniff";
    context.Response.Headers["X-Frame-Options"] = "DENY";
    context.Response.Headers["X-XSS-Protection"] = "1; mode=block";
    await next();
});
```

### A06: Vulnerable and Outdated Components
```powershell
# Audit NuGet packages regularly
dotnet list package --vulnerable
dotnet list package --outdated
```

### A07: Identification and Authentication Failures
```csharp
// ✅ Account lockout
options.Lockout.MaxFailedAccessAttempts = 5;
options.Lockout.DefaultLockoutTimeSpan = TimeSpan.FromMinutes(15);

// ✅ Rate limiting on login
[EnableRateLimiting("login-policy")]
[HttpPost("login")]
public async Task<IActionResult> Login(LoginRequest request) { ... }

// ✅ Strong password requirements
options.Password.RequiredLength = 12;
options.Password.RequireUppercase = true;
options.Password.RequireLowercase = true;
options.Password.RequireDigit = true;
```

### A08: Software and Data Integrity Failures
- Use SRI (Subresource Integrity) for CDN-loaded scripts
- Validate webhook signatures (e.g., Stripe uses HMAC)
```csharp
// Validate Stripe webhook signature
var webhookSecret = _config["Stripe:WebhookSecret"];
var stripeEvent = EventUtility.ConstructEvent(
    json, Request.Headers["Stripe-Signature"], webhookSecret);
```

### A09: Security Logging and Monitoring Failures
```csharp
// ✅ Log security events
logger.LogWarning("Failed login attempt for {Email} from {IP}", 
    email, context.Connection.RemoteIpAddress);

logger.LogWarning("Account locked for {UserId}", userId);
logger.LogInformation("User {UserId} logged in successfully", userId);
```

### A10: Server-Side Request Forgery (SSRF)
```csharp
// ❌ VULNERABLE — user can supply any URL
var url = request.ExternalUrl;
var data = await httpClient.GetStringAsync(url); // could be internal IP!

// ✅ SECURE — whitelist allowed domains
var allowedDomains = new[] { "api.exchangerate-api.com", "api.openweathermap.org" };
var uri = new Uri(url);
if (!allowedDomains.Contains(uri.Host))
    return BadRequest("URL not allowed");
```

---

## 7. Input Validation & Sanitization

```csharp
// FluentValidation — validate at API boundary
public class CreateTransactionValidator : AbstractValidator<CreateTransactionRequest>
{
    public CreateTransactionValidator()
    {
        RuleFor(x => x.Amount)
            .GreaterThan(0).WithMessage("Amount must be positive")
            .LessThanOrEqualTo(10_000_000).WithMessage("Amount too large");

        RuleFor(x => x.Description)
            .NotEmpty()
            .MaximumLength(200)
            .Matches(@"^[a-zA-Z0-9\s\-\.,#\/]+$") // whitelist characters
            .WithMessage("Description contains invalid characters");

        RuleFor(x => x.TransactionDate)
            .LessThanOrEqualTo(DateTime.UtcNow.Date.AddDays(1))
            .WithMessage("Transaction date cannot be in the future");
    }
}

// HTML encoding for any output rendered in HTML (anti-XSS)
using HtmlAgilityPack;
var encoded = HtmlEncoder.Default.Encode(userInput);

// Never trust user-supplied file paths
var fileName = Path.GetFileName(uploadedFileName); // strips directory traversal
var safePath = Path.Combine(_uploadDir, fileName);
if (!safePath.StartsWith(_uploadDir)) throw new SecurityException("Path traversal");
```

---

## 8. CORS

```csharp
// 🔗 BudgetPH CORS setup (Program.cs)
builder.Services.AddCors(options =>
    options.AddPolicy("AllowWeb", policy =>
        policy.WithOrigins(
            "http://localhost:3000",    // React dev server
            "http://localhost:5173")    // Vite default
        .AllowAnyHeader()
        .AllowAnyMethod()
        .AllowCredentials())); // required when sending cookies/auth headers

app.UseCors("AllowWeb"); // must be BEFORE UseAuthentication

// ❌ NEVER in production
policy.AllowAnyOrigin().AllowCredentials(); // CORS spec prohibits this combination!
```

---

## 9. Secrets Management

```csharp
// ❌ NEVER store secrets in code or appsettings.json committed to git
"JwtSettings": { "SecretKey": "my-secret" } // DO NOT COMMIT THIS!

// ✅ User Secrets (development only)
dotnet user-secrets set "JwtSettings:SecretKey" "my-secret-here"
// Stored in %APPDATA%\Microsoft\UserSecrets\ — never in project folder

// ✅ Environment variables (staging/production)
// On the server:
set JWTSECRETSKEY=your-production-secret
// In ASP.NET Core, env vars override appsettings.json

// ✅ Azure Key Vault / AWS Secrets Manager (production recommended)
builder.Configuration.AddAzureKeyVault(
    new Uri($"https://{vaultName}.vault.azure.net/"),
    new DefaultAzureCredential());
```

---

## 10. Common Interview Questions

**Q1: What is the difference between Authentication and Authorization?**

> **Authentication** = Who are you? (verifying identity). **Authorization** = What can you do? (verifying permissions). JWT handles both: the token proves identity (authentication), and the claims/roles within it determine access (authorization). `UseAuthentication()` must come before `UseAuthorization()` in the pipeline.

**Q2: Explain JWT — what's inside and why is it stateless?**

> A JWT has 3 parts: Header (algorithm), Payload (claims — user ID, roles, expiry), Signature (proves the token wasn't tampered with). It's stateless because the server doesn't store session data — all needed information is in the token itself. The server just validates the signature using its secret key. The downside: you can't invalidate a JWT before it expires (use short expiry + refresh tokens).

**Q3: What is the difference between HTTP 401 and 403?**

> **401 Unauthorized** = you are not authenticated (no token, expired token, invalid signature). **403 Forbidden** = you are authenticated but not authorized (valid token, but insufficient permissions). ASP.NET Core returns 401 when authentication fails, 403 when authorization fails.

**Q4: How do you store JWT tokens securely in a browser?**

> **HttpOnly cookie** (recommended) — inaccessible to JavaScript, prevents XSS theft, but requires CSRF protection. **localStorage** — simple but vulnerable to XSS attacks (any JS on the page can read it). **Memory (React state)** — most secure (not persisted), but lost on page refresh. Best practice: use HttpOnly cookies with SameSite=Strict and CSRF tokens.

**Q5: What is CSRF and how do you prevent it?**

> CSRF (Cross-Site Request Forgery) tricks an authenticated user's browser into making a request to your server from a malicious site. Prevention: (1) `SameSite=Strict` cookie attribute, (2) Anti-forgery tokens (`[ValidateAntiForgeryToken]` in MVC), (3) Check `Origin`/`Referer` headers, (4) Custom request headers (e.g., `X-Requested-With`) — cross-origin forms can't set custom headers.

**Q6: Why should you use `ClockSkew = TimeSpan.Zero` in JWT validation?**

> By default, JWT validation has a 5-minute clock skew tolerance, meaning a token remains valid for 5 minutes after its `exp` claim. Setting `ClockSkew = TimeSpan.Zero` makes expiry exact. This is important for security-sensitive applications where you want tokens to expire precisely when they say they do.
