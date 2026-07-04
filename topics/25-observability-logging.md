# Topic 25 — Observability & Logging

> You can't fix what you can't see. Observability is the ability to understand internal system state from external outputs. Three pillars: logs, metrics, traces.

---

## Three Pillars of Observability

| Pillar | What It Answers | Tools |
|---|---|---|
| **Logs** | "What happened?" | Serilog, NLog, Application Insights |
| **Metrics** | "How much / how fast?" | Prometheus, dotnet-counters, App Insights |
| **Traces** | "Where did time go across services?" | OpenTelemetry, Jaeger, Zipkin, App Insights |

---

## Serilog — Structured Logging

```csharp
// Install: Serilog.AspNetCore, Serilog.Sinks.Console, Serilog.Sinks.File
// Optionally: Serilog.Sinks.ApplicationInsights

// Program.cs — configure before anything else:
Log.Logger = new LoggerConfiguration()
    .MinimumLevel.Information()
    .MinimumLevel.Override("Microsoft", LogEventLevel.Warning)
    .MinimumLevel.Override("Microsoft.Hosting.Lifetime", LogEventLevel.Information)
    .MinimumLevel.Override("Microsoft.EntityFrameworkCore", LogEventLevel.Warning)
    .Enrich.FromLogContext()
    .Enrich.WithMachineName()
    .Enrich.WithEnvironmentName()
    .WriteTo.Console(outputTemplate:
        "[{Timestamp:HH:mm:ss} {Level:u3}] {Message:lj} ({SourceContext}){NewLine}{Exception}")
    .WriteTo.File(
        path: "logs/app-.txt",
        rollingInterval: RollingInterval.Day,
        retainedFileCountLimit: 14,
        outputTemplate: "[{Timestamp:yyyy-MM-dd HH:mm:ss} {Level:u3}] {Message:lj}{NewLine}{Exception}")
    .CreateLogger();

builder.Host.UseSerilog();

// Structured logging — log properties, not string concatenation:
// ❌ Wrong:
_logger.LogInformation($"User {userId} created transaction {txId} for {amount}");

// ✅ Correct — properties are searchable in log aggregation tools:
_logger.LogInformation(
    "User {UserId} created transaction {TransactionId} for {Amount} {Currency}",
    userId, txId, amount, currency);
// Results in a log event with structured properties: UserId, TransactionId, Amount, Currency

// Log with context:
using (_logger.BeginScope(new Dictionary<string, object>
{
    ["UserId"] = userId,
    ["CorrelationId"] = correlationId
}))
{
    _logger.LogInformation("Processing transaction {TransactionId}", txId);
    _logger.LogWarning("Balance {Balance} is low", balance);
    // Both entries have UserId + CorrelationId automatically
}
```

### Log Levels — When to Use Each

```csharp
_logger.LogTrace("Entering method {Method}", nameof(CreateAsync));    // Very verbose, dev only
_logger.LogDebug("Query params: {Params}", queryParams);               // Debug info, rarely in prod
_logger.LogInformation("Transaction {Id} created", transactionId);     // Normal operations
_logger.LogWarning("Low balance {Balance} for account {AccountId}", balance, accountId); // Unusual but not error
_logger.LogError(ex, "Failed to send notification for {UserId}", userId);   // Errors, always log
_logger.LogCritical(ex, "Database connection failed — app cannot start");    // System failures
```

---

## Request Logging Middleware

```csharp
// Log every HTTP request with timing:
app.UseSerilogRequestLogging(options =>
{
    options.MessageTemplate = "HTTP {RequestMethod} {RequestPath} responded {StatusCode} in {Elapsed:0.0000} ms";
    
    options.GetLevel = (ctx, elapsed, ex) =>
    {
        if (ex != null || ctx.Response.StatusCode >= 500)
            return LogEventLevel.Error;
        if (ctx.Response.StatusCode >= 400 || elapsed > 3000)
            return LogEventLevel.Warning;
        return LogEventLevel.Information;
    };
    
    options.EnrichDiagnosticContext = (diagnosticContext, httpContext) =>
    {
        diagnosticContext.Set("UserId", httpContext.User.FindFirst("sub")?.Value);
        diagnosticContext.Set("ClientIP", httpContext.Connection.RemoteIpAddress?.ToString());
        diagnosticContext.Set("UserAgent", httpContext.Request.Headers.UserAgent.ToString());
    };
});
```

---

## OpenTelemetry — Distributed Tracing

```csharp
// Install: OpenTelemetry.Extensions.Hosting, OpenTelemetry.Instrumentation.AspNetCore,
//          OpenTelemetry.Instrumentation.EntityFrameworkCore, OpenTelemetry.Exporter.Jaeger

// Program.cs:
builder.Services.AddOpenTelemetry()
    .ConfigureResource(resource => resource
        .AddService("FinanceManager.API", serviceVersion: "1.0.0"))
    .WithTracing(tracing => tracing
        .AddAspNetCoreInstrumentation(opts =>
        {
            opts.RecordException = true;
            opts.Filter = ctx => ctx.Request.Path != "/health"; // exclude health checks
        })
        .AddEntityFrameworkCoreInstrumentation(opts => opts.SetDbStatementForText = true)
        .AddHttpClientInstrumentation()
        .AddSource("FinanceManager.*")  // custom activities
        .AddJaegerExporter(opts =>
        {
            opts.AgentHost = "localhost"; // Jaeger running locally
            opts.AgentPort = 6831;
        })
    )
    .WithMetrics(metrics => metrics
        .AddAspNetCoreInstrumentation()
        .AddRuntimeInstrumentation()
        .AddPrometheusExporter() // exposes /metrics endpoint
    );

// Custom activity (span) in service:
private static readonly ActivitySource _activitySource = new("FinanceManager.TransactionService");

public async Task<Transaction> CreateAsync(CreateTransactionCommand cmd, CancellationToken ct)
{
    using var activity = _activitySource.StartActivity("CreateTransaction");
    activity?.SetTag("userId", cmd.UserId.ToString());
    activity?.SetTag("amount", cmd.Amount.ToString());
    activity?.SetTag("type", cmd.Type.ToString());
    
    try
    {
        var tx = await _repo.CreateAsync(cmd, ct);
        activity?.SetTag("transactionId", tx.Id.ToString());
        return tx;
    }
    catch (Exception ex)
    {
        activity?.RecordException(ex);
        activity?.SetStatus(ActivityStatusCode.Error, ex.Message);
        throw;
    }
}
```

---

## Application Insights Integration

```csharp
// Install: Microsoft.ApplicationInsights.AspNetCore

// Program.cs:
builder.Services.AddApplicationInsightsTelemetry(opts =>
    opts.ConnectionString = builder.Configuration["ApplicationInsights:ConnectionString"]);

// Custom telemetry:
public class TransactionCommandHandler
{
    private readonly TelemetryClient _telemetry;
    
    public async Task<Result> Handle(CreateTransactionCommand cmd, CancellationToken ct)
    {
        var sw = Stopwatch.StartNew();
        
        try
        {
            var tx = await CreateTransactionAsync(cmd, ct);
            
            // Track business event:
            _telemetry.TrackEvent("TransactionCreated", new Dictionary<string, string>
            {
                ["type"] = cmd.Type.ToString(),
                ["categoryId"] = cmd.CategoryId.ToString()
            }, new Dictionary<string, double>
            {
                ["amount"] = (double)cmd.Amount
            });
            
            return Result.Success(tx.Id);
        }
        catch (Exception ex)
        {
            _telemetry.TrackException(ex, new Dictionary<string, string>
            {
                ["command"] = nameof(CreateTransactionCommand),
                ["userId"] = cmd.UserId.ToString()
            });
            throw;
        }
        finally
        {
            // Track operation duration:
            _telemetry.TrackMetric("TransactionCreate.Duration", sw.ElapsedMilliseconds);
        }
    }
}
```

---

## Health Checks

```csharp
// Install: Microsoft.AspNetCore.Diagnostics.HealthChecks, AspNetCore.HealthChecks.SqlServer

// Program.cs:
builder.Services.AddHealthChecks()
    .AddSqlServer(
        connectionString: builder.Configuration.GetConnectionString("DefaultConnection")!,
        name: "sqlserver",
        tags: ["database", "ready"])
    .AddCheck("disk", () =>
    {
        var free = new DriveInfo("C").AvailableFreeSpace;
        return free > 1_000_000_000 // 1 GB
            ? HealthCheckResult.Healthy($"Disk free: {free / 1e9:N1} GB")
            : HealthCheckResult.Degraded($"Low disk: {free / 1e6:N0} MB");
    }, tags: ["system"]);

// Map endpoints:
app.MapHealthChecks("/health", new HealthCheckOptions { Predicate = _ => true });
app.MapHealthChecks("/health/ready", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("ready"),
    ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse // JSON response
});
app.MapHealthChecks("/health/live", new HealthCheckOptions
{
    Predicate = check => !check.Tags.Contains("database") // app is alive even if DB is slow
});
```

---

## Correlation IDs

```csharp
// Middleware to propagate correlation ID across requests:
public class CorrelationIdMiddleware : IMiddleware
{
    private const string CorrelationIdHeader = "X-Correlation-ID";
    
    public async Task InvokeAsync(HttpContext context, RequestDelegate next)
    {
        var correlationId = context.Request.Headers[CorrelationIdHeader].FirstOrDefault()
            ?? Guid.NewGuid().ToString();
        
        context.Response.Headers.Append(CorrelationIdHeader, correlationId);
        
        using (LogContext.PushProperty("CorrelationId", correlationId))
        {
            await next(context);
        }
    }
}
```

---

## Interview Q&A

**Q1: What is the difference between logging and tracing?**
> Logging records discrete events (what happened, when, severity). Distributed tracing follows a request across multiple services — shows the full path with timing for each span. A trace contains multiple spans; a log message is part of a span. OpenTelemetry correlates logs and traces via TraceId/SpanId.

**Q2: What is structured logging and why is it better than string concatenation?**
> Structured logging emits log events as key-value pairs, not plain strings. `Log.Information("User {UserId} logged in", userId)` stores `UserId` as a searchable property, not part of the message text. In Kibana, Seq, or Application Insights you can filter: `UserId = "123"` instead of regex matching on message strings. Enables powerful querying and alerting.

**Q3: What is the difference between liveness and readiness health checks?**
> **Liveness** — is the app alive? (not deadlocked/crashed). Failing = kill and restart the container. Should check minimal things (memory, etc.). **Readiness** — is the app ready to serve traffic? Failing = remove from load balancer but don't restart. Checks dependencies (DB, external services). Kubernetes uses both probes; App Service uses health check path (which acts as liveness).

**Q4: How do you find N+1 query problems in production?**
> Enable EF Core query logging in dev (`optionsBuilder.LogTo(Console.WriteLine)`). In prod: check Application Insights dependencies or Serilog + Seq for many similar SQL queries per request. A sign: 1 request triggers `SELECT * FROM Transactions WHERE Id = @id` 50 times. Fix with `Include()`, `Select()` projection, or batch loading.
