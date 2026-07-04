# 02 — .NET Core & ASP.NET Core

> 🔗 **BudgetPH reference**: [Program.cs](https://github.com/randolfsegubre/BudgetPH/blob/master/src/Presentation/FinanceManager.API/Program.cs) | [BaseApiController.cs](https://github.com/randolfsegubre/BudgetPH/blob/master/src/Presentation/FinanceManager.API/Controllers/BaseApiController.cs)

---

## Table of Contents
1. [Dependency Injection — Lifetimes](#1-dependency-injection--lifetimes)
2. [Middleware Pipeline](#2-middleware-pipeline)
3. [Configuration System](#3-configuration-system)
4. [Filters (Action, Exception, Authorization)](#4-filters)
5. [Routing](#5-routing)
6. [Model Binding & Validation](#6-model-binding--validation)
7. [Minimal APIs vs Controllers](#7-minimal-apis-vs-controllers)
8. [Hosted Services & Background Tasks](#8-hosted-services--background-tasks)
9. [Logging with Serilog](#9-logging-with-serilog)
10. [Health Checks & Rate Limiting](#10-health-checks--rate-limiting)
11. [Common Interview Questions](#11-common-interview-questions)

---

## 1. Dependency Injection — Lifetimes

This is the **most frequently tested** topic in .NET interviews. Get this right.

### Three Lifetimes

| Lifetime | Created | Destroyed | Per |
|---|---|---|---|
| `Transient` | Every time requested | End of scope | Request per injection |
| `Scoped` | Once per scope (HTTP request) | End of scope | HTTP request |
| `Singleton` | First time requested | App shutdown | Application |

```csharp
// Registration
builder.Services.AddTransient<IEmailService, SmtpEmailService>();
builder.Services.AddScoped<ICurrentUserService, CurrentUserService>();
builder.Services.AddSingleton<ICacheService, InMemoryCacheService>();

// RULE: Never inject a shorter-lived service into a longer-lived one!
// ❌ CAPTIVE DEPENDENCY — Singleton holds a Scoped service
builder.Services.AddSingleton<IMyService, MyService>(); // holds Scoped!
public class MyService
{
    private readonly IDbContext _db; // ❌ Scoped injected into Singleton
    // _db will be the same instance forever — stale, thread-unsafe
}
```

### The Captive Dependency Problem

```
❌ WRONG:          ✅ CORRECT:
Singleton          Singleton
  └─ Scoped          └─ IServiceScopeFactory
       └─ OK                └─ create scope manually
                              └─ resolve Scoped
```

```csharp
// ✅ Correct way to use Scoped in Singleton — use IServiceScopeFactory
public class BackgroundProcessor
{
    private readonly IServiceScopeFactory _scopeFactory;
    
    public BackgroundProcessor(IServiceScopeFactory scopeFactory)
    {
        _scopeFactory = scopeFactory;
    }
    
    public async Task ProcessAsync()
    {
        using var scope = _scopeFactory.CreateScope();
        var dbContext = scope.ServiceProvider.GetRequiredService<ApplicationDbContext>();
        // use dbContext safely
    }
}
```

### 🔗 BudgetPH DI Registration

From [Application/DependencyInjection.cs](https://github.com/randolfsegubre/BudgetPH/blob/master/src/Core/FinanceManager.Application/DependencyInjection.cs):
```csharp
public static IServiceCollection AddApplication(this IServiceCollection services)
{
    services.AddAutoMapper(typeof(MappingProfile));
    services.AddFluentValidationAutoValidation();
    services.AddValidatorsFromAssembly(Assembly.GetExecutingAssembly());
    return services;
}
```

From [Infrastructure/DependencyInjection.cs](https://github.com/randolfsegubre/BudgetPH/blob/master/src/Infrastructure/FinanceManager.Infrastructure/DependencyInjection.cs):
```csharp
public static IServiceCollection AddInfrastructure(this IServiceCollection services, IConfiguration config)
{
    services.AddDbContext<ApplicationDbContext>(options =>
        options.UseSqlServer(config.GetConnectionString("DefaultConnection")));
    
    services.AddIdentity<IdentityUser, IdentityRole>()
        .AddEntityFrameworkStores<ApplicationDbContext>();
    
    services.AddScoped<IAuthService, AuthService>(); // Scoped ✅
    return services;
}
```

### Registering with Factory

```csharp
// Factory method for complex construction
builder.Services.AddScoped<IReportGenerator>(sp =>
{
    var config = sp.GetRequiredService<IConfiguration>();
    var logger = sp.GetRequiredService<ILogger<ReportGenerator>>();
    return new ReportGenerator(config["ReportPath"], logger);
});

// Keyed services (.NET 8+)
builder.Services.AddKeyedScoped<IPaymentGateway, PaymayaGateway>("paymaya");
builder.Services.AddKeyedScoped<IPaymentGateway, GCashGateway>("gcash");

// Resolve keyed service
public class PaymentService([FromKeyedServices("gcash")] IPaymentGateway gateway) {}
```

---

## 2. Middleware Pipeline

ASP.NET Core processes HTTP requests through a **pipeline of middleware components**. Order matters critically.

```
Request → [Middleware 1] → [Middleware 2] → [Middleware N] → Controller
Response ← [Middleware 1] ← [Middleware 2] ← [Middleware N] ←
```

### Correct Middleware Order (from BudgetPH)

```csharp
// 🔗 From Program.cs
app.UseHttpsRedirection();      // 1. Force HTTPS
app.UseCors("AllowWeb");        // 2. CORS headers (before auth)
app.UseSerilogRequestLogging(); // 3. Logging
app.UseAuthentication();        // 4. Who are you? (sets User)
app.UseAuthorization();         // 5. Are you allowed? (checks User)
app.MapControllers();           // 6. Route to controllers
```

**Critical Order Rules:**
- `UseCors` must come BEFORE `UseAuthentication`
- `UseAuthentication` must come BEFORE `UseAuthorization`
- `UseRouting` must come BEFORE `UseAuthorization` (implicit in modern .NET)
- Exception/error middleware should be FIRST (to catch errors from subsequent middleware)

### Creating Custom Middleware

```csharp
// Option 1: Inline with Use
app.Use(async (context, next) =>
{
    // Before next middleware
    var stopwatch = Stopwatch.StartNew();
    
    await next(context); // call next middleware
    
    // After next middleware (response is being sent)
    stopwatch.Stop();
    context.Response.Headers["X-Response-Time"] = $"{stopwatch.ElapsedMilliseconds}ms";
});

// Option 2: Class-based (recommended for non-trivial middleware)
public class RequestTimingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<RequestTimingMiddleware> _logger;

    public RequestTimingMiddleware(RequestDelegate next, ILogger<RequestTimingMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        var sw = Stopwatch.StartNew();
        await _next(context);
        sw.Stop();
        _logger.LogInformation("Request {Path} took {Ms}ms", 
            context.Request.Path, sw.ElapsedMilliseconds);
    }
}

// Register
app.UseMiddleware<RequestTimingMiddleware>();
// or via extension method:
public static IApplicationBuilder UseRequestTiming(this IApplicationBuilder app)
    => app.UseMiddleware<RequestTimingMiddleware>();
```

### `Run` vs `Use` vs `Map`

```csharp
// Use — calls next middleware
app.Use(async (ctx, next) => { ... await next(ctx); ... });

// Run — terminal (does NOT call next)
app.Run(async ctx => await ctx.Response.WriteAsync("Hello"));

// Map — branches pipeline based on path
app.Map("/api", apiApp =>
{
    apiApp.UseAuthentication();
    apiApp.UseAuthorization();
    apiApp.MapControllers();
});

// MapWhen — branches based on condition
app.MapWhen(ctx => ctx.Request.Headers.ContainsKey("X-Special"), specialApp =>
{
    specialApp.UseMiddleware<SpecialMiddleware>();
});
```

---

## 3. Configuration System

### Configuration Sources (in priority order, last wins)
1. `appsettings.json`
2. `appsettings.{Environment}.json`
3. User Secrets (Development only)
4. Environment Variables
5. Command-line arguments

```csharp
// Reading values
var connStr = configuration.GetConnectionString("DefaultConnection");
var jwtSecret = configuration["JwtSettings:SecretKey"];
var port = configuration.GetValue<int>("AppSettings:Port", defaultValue: 5000);
```

### Options Pattern (Strongly-typed configuration)

```csharp
// Define options class
public class JwtSettings
{
    public string SecretKey { get; set; } = "";
    public string Issuer { get; set; } = "";
    public string Audience { get; set; } = "";
    public int ExpiryMinutes { get; set; } = 60;
}

// Register
builder.Services.Configure<JwtSettings>(builder.Configuration.GetSection("JwtSettings"));

// Inject — three flavors:
// IOptions<T> — Singleton, reads once at startup
public class TokenService(IOptions<JwtSettings> jwtOptions)
{
    private readonly JwtSettings _jwt = jwtOptions.Value;
}

// IOptionsSnapshot<T> — Scoped, re-reads on each request (supports hot reload)
public class TokenService(IOptionsSnapshot<JwtSettings> jwtOptions)
{
    private readonly JwtSettings _jwt = jwtOptions.Value; // fresh each request
}

// IOptionsMonitor<T> — Singleton, notifies on change
public class TokenService(IOptionsMonitor<JwtSettings> jwtMonitor)
{
    void Setup()
    {
        jwtMonitor.OnChange(settings => 
            Console.WriteLine($"JWT config changed: {settings.SecretKey}"));
    }
}
```

### 🔗 BudgetPH JWT Config Example

From [Program.cs](https://github.com/randolfsegubre/BudgetPH/blob/master/src/Presentation/FinanceManager.API/Program.cs):
```csharp
var jwt = builder.Configuration.GetSection("JwtSettings");
builder.Services.AddAuthentication(...)
    .AddJwtBearer(options =>
    {
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidIssuer = jwt["Issuer"],
            ValidAudience = jwt["Audience"],
            IssuerSigningKey = new SymmetricSecurityKey(
                Encoding.UTF8.GetBytes(jwt["SecretKey"]!))
        };
    });
```

---

## 4. Filters

Filters run code **before/after** specific stages in the action invocation pipeline.

### Filter Types and Execution Order

```
Authorization Filter → Resource Filter → [Model Binding] → Action Filter → Controller Action
                                                                          ↓
                    ← Result Filter ← Exception Filter (if error) ← Action Result
```

| Filter | Interface | Use For |
|---|---|---|
| Authorization | `IAuthorizationFilter` | Custom auth logic |
| Resource | `IResourceFilter` | Caching, short-circuit |
| Action | `IActionFilter` | Logging, input manipulation |
| Exception | `IExceptionFilter` | Global error handling |
| Result | `IResultFilter` | Response manipulation |
| Always | `IAlwaysRunResultFilter` | Runs even if short-circuited |

```csharp
// Action Filter — logs method execution time
public class LogExecutionTimeFilter : IActionFilter
{
    private Stopwatch? _stopwatch;

    public void OnActionExecuting(ActionExecutingContext context)
    {
        _stopwatch = Stopwatch.StartNew();
        Console.WriteLine($"Executing: {context.ActionDescriptor.DisplayName}");
    }

    public void OnActionExecuted(ActionExecutedContext context)
    {
        _stopwatch?.Stop();
        Console.WriteLine($"Completed in {_stopwatch?.ElapsedMilliseconds}ms");
    }
}

// Exception Filter — global error handler
public class ApiExceptionFilter : IExceptionFilter
{
    private readonly ILogger<ApiExceptionFilter> _logger;

    public ApiExceptionFilter(ILogger<ApiExceptionFilter> logger)
    {
        _logger = logger;
    }

    public void OnException(ExceptionContext context)
    {
        _logger.LogError(context.Exception, "Unhandled exception");
        
        context.Result = context.Exception switch
        {
            NotFoundException ex => new NotFoundObjectResult(new { error = ex.Message }),
            UnauthorizedException => new UnauthorizedResult(),
            ValidationException ex => new BadRequestObjectResult(new { errors = ex.Errors }),
            _ => new ObjectResult(new { error = "Internal server error" }) { StatusCode = 500 }
        };
        
        context.ExceptionHandled = true;
    }
}

// Register globally
builder.Services.AddControllers(options =>
{
    options.Filters.Add<ApiExceptionFilter>();
    options.Filters.Add<LogExecutionTimeFilter>();
});

// Register on specific controller or action
[ServiceFilter(typeof(LogExecutionTimeFilter))]
[HttpGet("{id}")]
public async Task<IActionResult> GetById(Guid id, CancellationToken ct) { ... }
```

---

## 5. Routing

### Attribute Routing (Used in BudgetPH)

```csharp
// 🔗 From BaseApiController.cs
[ApiController]
[Route("api/[controller]")]  // api/accounts, api/transactions, etc.
public abstract class BaseApiController : ControllerBase { }

// Route templates
[Route("api/v{version:apiVersion}/[controller]")]  // API versioning
[HttpGet]                           // GET /api/accounts
[HttpGet("{id:guid}")]              // GET /api/accounts/{guid}
[HttpGet("{id:guid}/details")]      // GET /api/accounts/{guid}/details
[HttpPost]                          // POST /api/accounts
[HttpPut("{id:guid}")]              // PUT /api/accounts/{guid}
[HttpDelete("{id:guid}")]           // DELETE /api/accounts/{guid}

// Route constraints
[HttpGet("{id:int:min(1)}")]        // id must be int >= 1
[HttpGet("{slug:alpha}")]           // slug must be alphabetic
[HttpGet("{date:datetime}")]        // must be a valid datetime
```

### Route Parameters vs Query Parameters vs Body

```csharp
// Route parameter — part of URL
[HttpGet("{id:guid}")]
public async Task<IActionResult> GetById([FromRoute] Guid id) { }

// Query parameter — ?page=1&pageSize=20
[HttpGet]
public async Task<IActionResult> GetAll(
    [FromQuery] int page = 1,
    [FromQuery] int pageSize = 20,
    [FromQuery] string? search = null) { }

// Request body — JSON payload
[HttpPost]
public async Task<IActionResult> Create([FromBody] CreateAccountRequest request) { }

// Header
[HttpGet]
public IActionResult Get([FromHeader(Name = "X-Api-Key")] string apiKey) { }
```

---

## 6. Model Binding & Validation

### Data Annotations

```csharp
public class CreateTransactionRequest
{
    [Required]
    public Guid AccountId { get; set; }
    
    [Required, StringLength(200)]
    public string Description { get; set; } = "";
    
    [Range(0.01, double.MaxValue, ErrorMessage = "Amount must be positive")]
    public decimal Amount { get; set; }
    
    [EnumDataType(typeof(TransactionType))]
    public TransactionType TransactionType { get; set; }
    
    [DataType(DataType.Date)]
    public DateTime TransactionDate { get; set; }
}
```

### FluentValidation (Used in BudgetPH)

```csharp
// Validator class
public class CreateTransactionRequestValidator : AbstractValidator<CreateTransactionRequest>
{
    public CreateTransactionRequestValidator()
    {
        RuleFor(x => x.AccountId)
            .NotEmpty().WithMessage("Account is required");
        
        RuleFor(x => x.Amount)
            .GreaterThan(0).WithMessage("Amount must be positive")
            .LessThanOrEqualTo(10_000_000).WithMessage("Amount too large");
        
        RuleFor(x => x.Description)
            .NotEmpty()
            .MaximumLength(200)
            .When(x => x.TransactionType != TransactionType.Transfer);
        
        RuleFor(x => x.TransactionDate)
            .NotEmpty()
            .LessThanOrEqualTo(DateTime.UtcNow.AddDays(1))
            .WithMessage("Transaction date cannot be in the future");
    }
}

// Automatic validation via DI (registered in Application/DependencyInjection.cs)
services.AddFluentValidationAutoValidation();
services.AddValidatorsFromAssembly(Assembly.GetExecutingAssembly());
// ↑ Returns 400 BadRequest automatically on validation failure
```

---

## 7. Minimal APIs vs Controllers

### Controllers (Used in BudgetPH)

```csharp
// Full MVC controller — more structure, filters, routing, model binding
[ApiController]
[Route("api/[controller]")]
[Authorize]
public class AccountsController : BaseApiController
{
    [HttpGet]
    public async Task<IActionResult> GetAll(CancellationToken ct) { ... }
}
```

### Minimal APIs (.NET 6+)

```csharp
// Concise, no controller class needed
var app = builder.Build();

app.MapGet("/api/accounts", async (ApplicationDbContext db, ClaimsPrincipal user, CancellationToken ct) =>
{
    var accounts = await db.Accounts
        .Where(a => a.UserId == GetUserId(user))
        .ToListAsync(ct);
    return Results.Ok(accounts);
}).RequireAuthorization();

app.MapPost("/api/accounts", async (CreateAccountRequest req, ApplicationDbContext db) =>
{
    var account = new Account { Name = req.Name, Balance = req.Balance };
    db.Accounts.Add(account);
    await db.SaveChangesAsync();
    return Results.Created($"/api/accounts/{account.Id}", account);
}).RequireAuthorization();

// Route groups (organize like controllers)
var accountsGroup = app.MapGroup("/api/accounts").RequireAuthorization();
accountsGroup.MapGet("/", GetAllAccounts);
accountsGroup.MapGet("/{id:guid}", GetById);
accountsGroup.MapPost("/", CreateAccount);
```

**When to choose Minimal APIs:**
- Simple CRUD, microservices, serverless functions
- Performance-critical endpoints (slightly less overhead)

**When to choose Controllers:**
- Complex apps with many endpoints, filters, versioning
- Teams familiar with MVC pattern (like BudgetPH)

---

## 8. Hosted Services & Background Tasks

```csharp
// IHostedService — manual control
public class MyStartupService : IHostedService
{
    public Task StartAsync(CancellationToken cancellationToken)
    {
        Console.WriteLine("App started — initializing...");
        return Task.CompletedTask;
    }

    public Task StopAsync(CancellationToken cancellationToken)
    {
        Console.WriteLine("App stopping — cleaning up...");
        return Task.CompletedTask;
    }
}

// BackgroundService — easier for long-running loops
public class NotificationBackgroundService : BackgroundService
{
    private readonly IServiceScopeFactory _scopeFactory;
    private readonly ILogger<NotificationBackgroundService> _logger;

    public NotificationBackgroundService(
        IServiceScopeFactory scopeFactory,
        ILogger<NotificationBackgroundService> logger)
    {
        _scopeFactory = scopeFactory;
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                using var scope = _scopeFactory.CreateScope();
                var db = scope.ServiceProvider.GetRequiredService<ApplicationDbContext>();
                
                // Check for budget alerts, due payments, etc.
                await CheckBudgetAlertsAsync(db, stoppingToken);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error in notification service");
            }

            await Task.Delay(TimeSpan.FromMinutes(15), stoppingToken); // run every 15 min
        }
    }
}

// Register
builder.Services.AddHostedService<NotificationBackgroundService>();
```

---

## 9. Logging with Serilog

### 🔗 BudgetPH Serilog Setup ([Program.cs](https://github.com/randolfsegubre/BudgetPH/blob/master/src/Presentation/FinanceManager.API/Program.cs))

```csharp
// Bootstrap logger (before builder)
Log.Logger = new LoggerConfiguration().WriteTo.Console().CreateBootstrapLogger();

builder.Host.UseSerilog((ctx, lc) => lc
    .ReadFrom.Configuration(ctx.Configuration)
    .WriteTo.Console()
    .WriteTo.File("logs/api-.txt", rollingInterval: RollingInterval.Day));

// In middleware
app.UseSerilogRequestLogging(); // logs each HTTP request
```

### Structured Logging

```csharp
// ❌ String interpolation — loses structure, can't query
logger.LogInformation($"User {userId} created transaction {transactionId}");

// ✅ Structured logging — queryable properties
logger.LogInformation("User {UserId} created transaction {TransactionId}", userId, transactionId);

// Log levels
logger.LogTrace("Verbose debug");       // level 0
logger.LogDebug("Debug info");          // level 1
logger.LogInformation("Normal flow");   // level 2
logger.LogWarning("Unexpected event");  // level 3
logger.LogError(ex, "Error occurred");  // level 4
logger.LogCritical("System failure");   // level 5
```

---

## 10. Health Checks & Rate Limiting

### Health Checks

```csharp
builder.Services.AddHealthChecks()
    .AddSqlServer(connectionString, name: "database")
    .AddUrlGroup(new Uri("https://api.example.com"), name: "external-api");

app.MapHealthChecks("/health");
app.MapHealthChecks("/health/ready", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("ready")
});
```

### Rate Limiting (.NET 7+)

```csharp
builder.Services.AddRateLimiter(options =>
{
    options.AddFixedWindowLimiter("fixed", o =>
    {
        o.PermitLimit = 10;
        o.Window = TimeSpan.FromSeconds(10);
        o.QueueLimit = 5;
    });
    
    options.AddSlidingWindowLimiter("sliding", o =>
    {
        o.PermitLimit = 100;
        o.Window = TimeSpan.FromMinutes(1);
        o.SegmentsPerWindow = 6;
    });
    
    options.RejectionStatusCode = StatusCodes.Status429TooManyRequests;
});

app.UseRateLimiter();

// Apply to controller/endpoint
[EnableRateLimiting("fixed")]
[HttpPost("login")]
public async Task<IActionResult> Login(LoginRequest request) { ... }
```

---

## 11. Common Interview Questions

**Q1: What is the difference between Scoped, Transient, and Singleton lifetimes?**

> **Transient**: new instance every time it is requested — good for stateless, lightweight services. **Scoped**: one instance per HTTP request — good for `DbContext`, `ICurrentUserService`. **Singleton**: one instance for the application's lifetime — good for caches, configuration, stateless services. Key rule: never inject Scoped into Singleton.

**Q2: Explain the ASP.NET Core middleware pipeline.**

> Middleware is a chain of components that process HTTP requests sequentially. Each component can perform work before and after calling the next component. The order of `app.Use*()` calls determines execution order. Critical: `UseAuthentication` must come before `UseAuthorization`; `UseCors` must come before `UseAuthentication`.

**Q3: What is the Options pattern and its three variants?**

> The Options pattern provides strongly-typed access to configuration. `IOptions<T>` — Singleton, read once; `IOptionsSnapshot<T>` — Scoped, refreshes per request; `IOptionsMonitor<T>` — Singleton with change notification callbacks.

**Q4: How does `[ApiController]` differ from `[Controller]`?**

> `[ApiController]` adds several behaviors: automatic 400 responses from model validation, binding source inference (params from route/query/body based on type), problem details for error responses, and multipart/form-data validation. It's specifically for API controllers that return data, not views.

**Q5: What is the difference between `IActionFilter` and middleware?**

> Middleware runs for ALL requests including static files and non-controller routes. Action filters only run for controller actions. Use middleware for cross-cutting concerns that apply broadly (CORS, auth, logging). Use filters for concerns specific to MVC actions (logging action timing, validating a specific request type).

**Q6: How would you implement global exception handling in ASP.NET Core?**

> Three approaches: (1) Exception middleware (`app.UseExceptionHandler`), (2) Global `IExceptionFilter`, (3) `IProblemDetailsService`. BudgetPH uses an exception filter. The middleware approach is more inclusive (catches errors in middleware, not just controller actions). Best practice: use `UseExceptionHandler` for a centralized fallback plus domain-specific exception filters.

**Q7: What is `IServiceScopeFactory` and when do you need it?**

> Used when a Singleton service needs to resolve Scoped services. Since Scoped services can't be directly injected into Singletons (captive dependency), `IServiceScopeFactory` lets you create a new scope manually and resolve the Scoped service within it. Common in `BackgroundService` implementations.

**Q8: Explain `ConfigureServices` vs `Configure` in old-style Startup.**

> `ConfigureServices` registers services in the DI container. `Configure` builds the middleware pipeline using the `IApplicationBuilder`. In .NET 6+ `Program.cs` (minimal hosting model), these are merged — `builder.Services.*` replaces `ConfigureServices`, and `app.Use*()` replaces `Configure`.
