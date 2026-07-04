# 11 — Performance & Caching

---

## Table of Contents
1. [Caching Strategies](#1-caching-strategies)
2. [IMemoryCache](#2-imemorycache)
3. [IDistributedCache & Redis](#3-idistributedcache--redis)
4. [Response Caching & Output Caching](#4-response-caching--output-caching)
5. [Async Performance Patterns](#5-async-performance-patterns)
6. [EF Core Performance](#6-ef-core-performance)
7. [Database Connection Pooling](#7-database-connection-pooling)
8. [Common Interview Questions](#8-common-interview-questions)

---

## 1. Caching Strategies

| Strategy | Scope | Speed | Shared | Use When |
|---|---|---|---|---|
| **In-Memory** | Single server | Fastest | ❌ | Single-server apps, frequently-read static data |
| **Distributed (Redis)** | Multiple servers | Fast | ✅ | Multi-server apps, session data, rate limiting |
| **Response Cache** | HTTP client | Network save | Per client | Public APIs, rarely-changing data |
| **Output Cache** | Server-side HTTP | Fast | Per config | Server-side rendered pages, API responses |
| **CDN** | Edge | Fastest | Global | Static assets, public content |

### Cache-Aside Pattern (Most Common)
```
1. Check cache → hit? return cached
2. Cache miss → fetch from DB
3. Store in cache
4. Return data
```

---

## 2. IMemoryCache

```csharp
// Register
builder.Services.AddMemoryCache();

// Inject and use
public class DashboardService
{
    private readonly IMemoryCache _cache;
    private readonly ApplicationDbContext _context;

    public DashboardService(IMemoryCache cache, ApplicationDbContext context)
    {
        _cache = cache; _context = context;
    }

    public async Task<DashboardSummaryDto> GetSummaryAsync(Guid userId, CancellationToken ct)
    {
        var cacheKey = $"dashboard-summary:{userId}";

        // GetOrCreateAsync — thread-safe cache-aside pattern
        return await _cache.GetOrCreateAsync(cacheKey, async entry =>
        {
            entry.AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(5);
            entry.SlidingExpiration = TimeSpan.FromMinutes(2); // reset timer on each access
            entry.Priority = CacheItemPriority.Normal;

            // Called only on cache miss
            return await FetchDashboardFromDbAsync(userId, ct);
        }) ?? throw new InvalidOperationException("Cache returned null");
    }

    // Invalidate cache when data changes
    public async Task InvalidateDashboardCacheAsync(Guid userId)
    {
        _cache.Remove($"dashboard-summary:{userId}");
    }
}

// Advanced: cache with cancellation token support
_cache.Set(key, value, new MemoryCacheEntryOptions
{
    AbsoluteExpiration = DateTimeOffset.UtcNow.AddHours(1),
    Size = 1, // for size-limited cache
    PostEvictionCallbacks = { new PostEvictionCallbackRegistration
    {
        EvictionCallback = (key, value, reason, state) =>
            Console.WriteLine($"Cache evicted: {key} due to {reason}")
    }}
});
```

### Memory Cache Sizing

```csharp
// Limit total cache size (prevents memory explosion)
builder.Services.AddMemoryCache(options =>
{
    options.SizeLimit = 1024; // 1024 "units"
    options.CompactionPercentage = 0.25; // remove 25% when limit hit
    options.ExpirationScanFrequency = TimeSpan.FromMinutes(1);
});

// Each entry must declare its size
_cache.Set(key, value, new MemoryCacheEntryOptions { Size = 1 });
```

---

## 3. IDistributedCache & Redis

```csharp
// Register Redis distributed cache
builder.Services.AddStackExchangeRedisCache(options =>
{
    options.Configuration = builder.Configuration.GetConnectionString("Redis");
    options.InstanceName = "BudgetPH:";
});

// IDistributedCache — string/byte based interface
public class CachedAccountService
{
    private readonly IDistributedCache _cache;
    private readonly ApplicationDbContext _context;
    private static readonly JsonSerializerOptions _jsonOptions = new() { PropertyNamingPolicy = JsonNamingPolicy.CamelCase };

    public async Task<AccountDto?> GetByIdAsync(Guid id, CancellationToken ct)
    {
        var cacheKey = $"account:{id}";
        
        // Try cache first
        var cached = await _cache.GetStringAsync(cacheKey, ct);
        if (cached is not null)
            return JsonSerializer.Deserialize<AccountDto>(cached, _jsonOptions);
        
        // Fetch from DB
        var account = await _context.Accounts.FindAsync(new object[] { id }, ct);
        if (account is null) return null;
        
        var dto = Map(account);
        
        // Store in cache
        await _cache.SetStringAsync(
            cacheKey,
            JsonSerializer.Serialize(dto, _jsonOptions),
            new DistributedCacheEntryOptions
            {
                AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(10),
                SlidingExpiration = TimeSpan.FromMinutes(5)
            },
            ct);
        
        return dto;
    }

    public async Task InvalidateAsync(Guid id, CancellationToken ct)
        => await _cache.RemoveAsync($"account:{id}", ct);
}
```

### Redis Use Cases

```csharp
// 1. Distributed rate limiting
// 2. Session storage
// 3. Pub/Sub messaging
// 4. Distributed locks
// 5. Leaderboards (sorted sets)
// 6. Cache invalidation across servers

// Distributed lock example (prevent duplicate processing)
var lockKey = $"processing-lock:transaction:{transactionId}";
var lockValue = Guid.NewGuid().ToString();

// Acquire lock (SET key value NX PX milliseconds)
var acquired = await redisDb.StringSetAsync(lockKey, lockValue, 
    TimeSpan.FromSeconds(30), When.NotExists);

if (!acquired)
    return; // another instance is processing this

try
{
    await ProcessTransactionAsync(transactionId);
}
finally
{
    // Release only if we own the lock (LUA script for atomicity)
    await redisDb.ScriptEvaluateAsync(
        "if redis.call('get', KEYS[1]) == ARGV[1] then return redis.call('del', KEYS[1]) else return 0 end",
        new RedisKey[] { lockKey }, new RedisValue[] { lockValue });
}
```

---

## 4. Response Caching & Output Caching

### Response Caching (HTTP Cache-Control headers)

```csharp
// Register
builder.Services.AddResponseCaching();
app.UseResponseCaching(); // must be early in pipeline

// On controller action
[HttpGet]
[ResponseCache(Duration = 60, VaryByHeader = "Accept-Language")]
public async Task<IActionResult> GetCategories()
{
    // Response will be cached by client/proxy for 60 seconds
    var categories = await context.Categories
        .Where(c => c.IsSystem)
        .AsNoTracking()
        .ToListAsync();
    return Ok(categories);
}

// On controller class (sets no-cache for all actions)
[ResponseCache(NoStore = true, Location = ResponseCacheLocation.None)]
public class TransactionsController : BaseApiController { }
```

### Output Caching (.NET 7+ — server-side)

```csharp
// Register
builder.Services.AddOutputCache(options =>
{
    options.AddPolicy("CategoriesPolicy", policy =>
        policy.Expire(TimeSpan.FromHours(1))
              .Tag("categories"));
    
    options.AddPolicy("UserPolicy", policy =>
        policy.Expire(TimeSpan.FromMinutes(5))
              .VaryByRouteValue("id")
              .VaryByHeader("Authorization"));
});

app.UseOutputCache(); // before UseRouting

// Apply on endpoint
[HttpGet]
[OutputCache(PolicyName = "CategoriesPolicy")]
public async Task<IActionResult> GetCategories(CancellationToken ct)
{
    // First request: hits DB, caches result
    // Subsequent requests within 1 hour: served from cache
    return Ok(await context.Categories.AsNoTracking().ToListAsync(ct));
}

// Invalidate output cache by tag (when data changes)
[HttpPost]
public async Task<IActionResult> CreateCategory(
    [FromBody] CreateCategoryRequest req,
    [FromServices] IOutputCacheStore cacheStore,
    CancellationToken ct)
{
    // ... create category
    await cacheStore.EvictByTagAsync("categories", ct); // invalidate cached categories
    return CreatedAtAction(nameof(GetById), ...);
}
```

---

## 5. Async Performance Patterns

### Parallel Operations

```csharp
// ✅ Parallel independent operations
var dashboardTask = context.Budgets.Where(b => b.UserId == userId)
    .AsNoTracking().ToListAsync(ct);
var accountsTask = context.Accounts.Where(a => a.UserId == userId)
    .AsNoTracking().ToListAsync(ct);
var savingsTask = context.SavingsGoals.Where(g => g.UserId == userId)
    .AsNoTracking().ToListAsync(ct);

await Task.WhenAll(dashboardTask, accountsTask, savingsTask);

var budgets = dashboardTask.Result;
var accounts = accountsTask.Result;
var savings = savingsTask.Result;
// ↑ 3 DB queries in parallel vs sequential = 3x faster

// ❌ Sequential (slow)
var budgets = await context.Budgets.ToListAsync(ct);
var accounts = await context.Accounts.ToListAsync(ct);
var savings = await context.SavingsGoals.ToListAsync(ct);
```

### Cancellation for Performance

```csharp
// Propagate CancellationToken everywhere — cancels DB queries when HTTP request is cancelled
[HttpGet]
public async Task<IActionResult> GetReports(CancellationToken ct)
{
    // If client disconnects, EF Core cancels the DB query automatically
    var reports = await context.Transactions
        .Where(t => t.UserId == userId)
        .OrderByDescending(t => t.TransactionDate)
        .ToListAsync(ct); // ← passes cancellation token
    
    return Ok(reports);
}
```

### ValueTask for Hot Paths

```csharp
// Use ValueTask when result is frequently available synchronously
public ValueTask<ExchangeRate> GetRateAsync(string from, string to)
{
    var key = $"{from}:{to}";
    
    if (_rateCache.TryGetValue(key, out var cachedRate))
        return ValueTask.FromResult(cachedRate); // no allocation!
    
    return new ValueTask<ExchangeRate>(FetchRateFromApiAsync(from, to));
}
```

---

## 6. EF Core Performance

### AsNoTracking for Read Operations

```csharp
// ✅ Read-only queries — disable change tracking
var accounts = await context.Accounts
    .AsNoTracking()
    .Where(a => a.UserId == userId)
    .ToListAsync(ct);

// Even better — no tracking with identity resolution
var accounts = await context.Accounts
    .AsNoTrackingWithIdentityResolution()
    .Include(a => a.Transactions)
    .ToListAsync(ct);
```

### Select Only Needed Columns

```csharp
// ❌ Over-fetching — loads all 30+ properties
var accounts = await context.Accounts.ToListAsync(ct);
var summary = accounts.Select(a => new { a.Id, a.Name, a.Balance });

// ✅ Projection — only fetches 3 columns from DB
var summary = await context.Accounts
    .Select(a => new AccountSummaryDto { Id = a.Id, Name = a.Name, Balance = a.Balance })
    .ToListAsync(ct);
```

### Compiled Queries

```csharp
// Compiled queries avoid re-translating LINQ on every call
private static readonly Func<ApplicationDbContext, Guid, Task<List<TransactionDto>>>
    GetUserTransactionsCompiled = EF.CompileAsyncQuery(
        (ApplicationDbContext ctx, Guid userId) =>
            ctx.Transactions
               .Where(t => t.UserId == userId)
               .OrderByDescending(t => t.TransactionDate)
               .Select(t => new TransactionDto { Id = t.Id, Amount = t.Amount }));

// Usage — faster on repeated calls
var transactions = await GetUserTransactionsCompiled(context, userId);
```

### Bulk Operations (EF Core 7+)

```csharp
// ✅ Bulk update without loading entities
await context.Transactions
    .Where(t => t.TransactionDate < DateTime.UtcNow.AddYears(-5))
    .ExecuteUpdateAsync(s => s.SetProperty(t => t.IsArchived, true), ct);

// ✅ Bulk delete
await context.Transactions
    .Where(t => t.IsArchived && t.TransactionDate < DateTime.UtcNow.AddYears(-10))
    .ExecuteDeleteAsync(ct);
```

### Split Queries for Multiple Includes

```csharp
// ❌ Cartesian explosion — one query with JOINs can return huge result set
var accounts = await context.Accounts
    .Include(a => a.Transactions) // 1000 transactions
    .Include(a => a.Tags)          // 50 tags
    .ToListAsync(ct);
// SQL result set: 1000 × 50 = 50,000 rows!

// ✅ Split queries — multiple smaller queries
var accounts = await context.Accounts
    .Include(a => a.Transactions)
    .Include(a => a.Tags)
    .AsSplitQuery() // 3 separate SQL queries
    .ToListAsync(ct);
```

---

## 7. Database Connection Pooling

SQL Server connections are expensive to create. EF Core uses ADO.NET connection pooling by default.

```csharp
// SQL Server connection string with pool settings
"Server=.;Database=BudgetPH_Dev;Trusted_Connection=True;
 Min Pool Size=5;Max Pool Size=100;Connection Timeout=30;"

// For high-concurrency apps — increase pool size
builder.Services.AddDbContext<ApplicationDbContext>(options =>
    options.UseSqlServer(
        connectionString,
        sqlOptions => sqlOptions
            .EnableRetryOnFailure(maxRetryCount: 3, maxRetryDelay: TimeSpan.FromSeconds(5), null)
            .CommandTimeout(30)));
```

---

## 8. Common Interview Questions

**Q1: What is the difference between `IMemoryCache` and `IDistributedCache`?**

> `IMemoryCache` — in-process cache, stores objects in application memory. Fast, no serialization needed, but NOT shared between server instances. Lost on app restart. `IDistributedCache` — external cache (Redis, SQL Server). Shared across all server instances (essential for load-balanced apps). Requires serialization (byte array/string). Slightly slower due to network I/O. Use `IMemoryCache` for single-server apps; `IDistributedCache` for multi-server.

**Q2: What is cache stampede (thundering herd) and how do you prevent it?**

> Cache stampede occurs when a cache key expires and many concurrent requests all miss the cache at the same time, all hitting the database simultaneously. Prevention: (1) **Locking** — use `SemaphoreSlim` to allow only one thread to populate the cache; (2) **Cache jitter** — add random TTL variation to prevent simultaneous expiry; (3) **Background refresh** — refresh cache before it expires; (4) `GetOrCreateAsync` in .NET's `IMemoryCache` includes factory locking for single-instance apps.

**Q3: Explain `AbsoluteExpiration` vs `SlidingExpiration`.**

> `AbsoluteExpiration` — cache entry expires at a specific time (e.g., 1 hour from creation). Good for time-sensitive data. `SlidingExpiration` — cache entry expires after N time of inactivity. Each access resets the timer. Good for user session data. You can combine both — the entry expires on whichever comes first. Always set an `AbsoluteExpiration` even with sliding, to prevent entries from living forever.

**Q4: Why is `Task.WhenAll` better than sequential awaiting for independent operations?**

> Sequential awaiting blocks: total time = T1 + T2 + T3. `Task.WhenAll` runs in parallel: total time = max(T1, T2, T3). For a dashboard with 3 independent DB queries each taking 100ms, sequential = 300ms, parallel = ~100ms. Use `Task.WhenAll` when operations don't depend on each other's results.

**Q5: What is query splitting in EF Core and when should you use it?**

> `AsSplitQuery()` tells EF Core to execute a query with multiple collection `Include()`s as separate SQL `SELECT` statements instead of one big `JOIN`. This prevents Cartesian product explosion (e.g., 1000 transactions × 50 tags = 50,000 rows from a JOIN vs two queries of 1000 + 50 rows). Use when including multiple collections on the same entity. Downside: transactions are not atomic across split queries.

**Q6: What is the N+1 problem and how does caching help?**

> N+1 generates N+1 database queries instead of 1. Caching can mitigate it by caching individual entities (so the N lookups hit cache instead of DB), but the real fix is using `Include()` in EF Core. Caching at the service level (cache the entire result set) is often more efficient than caching individual entities.
