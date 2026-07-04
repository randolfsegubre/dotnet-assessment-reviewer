# Mock Exam 2 — Short Answer & Code Writing (25 Questions)

> ⏱️ **Time limit: 45 minutes** | Write answers before checking solutions

---

## Section A: Short Answer (Questions 1–15)

**Q1.** Explain the difference between `IEnumerable<T>` and `IQueryable<T>`. When would you use each?

<details>
<summary>✅ Answer</summary>

`IEnumerable<T>` — represents an in-memory collection. LINQ operations run as C# code on data already loaded into memory. Use when working with in-memory collections or after data has been materialized.

`IQueryable<T>` — represents a query that will be translated to SQL (or another query language) and executed on the data source. LINQ operations are translated by the provider (e.g., EF Core) into SQL. Use when working with EF Core/LINQ-to-SQL to avoid loading unnecessary data.

**Rule:** Keep data as `IQueryable<T>` until all filtering, sorting, and paging are applied, then call `.ToListAsync()` to materialize.
</details>

---

**Q2.** What are the three DI lifetimes in ASP.NET Core? Give a real-world example for each.

<details>
<summary>✅ Answer</summary>

- **Singleton** — one instance for the entire application. Example: `IConfiguration`, `IHttpClientFactory`, `ILogger`, in-memory cache. Shared across all requests.
- **Scoped** — one instance per HTTP request. Example: `DbContext`, `ICurrentUserService`, `IUnitOfWork`. Created at start of request, disposed at end.
- **Transient** — new instance every time it's requested. Example: form validators, lightweight stateless services.

**Key rule:** Never inject Scoped into Singleton — the Scoped service will be captured and outlive its intended lifetime.
</details>

---

**Q3.** What is the "N+1 query problem"? Write a code example showing the problem and the solution.

<details>
<summary>✅ Answer</summary>

**Problem:** Loading N parent entities and then issuing 1 additional query per entity to get related data.

```csharp
// ❌ N+1 — 1 query for accounts + N queries for transactions
var accounts = await context.Accounts.ToListAsync();
foreach (var account in accounts) // each access triggers a DB query with lazy loading
{
    Console.WriteLine(account.Transactions.Count); // query per account!
}

// ✅ Fix — eager loading with Include()
var accounts = await context.Accounts
    .Include(a => a.Transactions) // JOIN in single query
    .ToListAsync();
foreach (var account in accounts) // no additional queries
{
    Console.WriteLine(account.Transactions.Count);
}
```
</details>

---

**Q4.** Explain SOLID's Open/Closed Principle with a C# example.

<details>
<summary>✅ Answer</summary>

Classes should be **open for extension, closed for modification**. Add new behavior by adding new code, not changing existing code.

```csharp
// ❌ Violates OCP — add new report type = modify existing class
public class ReportExporter
{
    public void Export(Report r, string format)
    {
        if (format == "PDF") { /* PDF logic */ }
        else if (format == "Excel") { /* Excel logic */ }
        // Adding CSV requires modifying this class
    }
}

// ✅ OCP — add new format by adding a new class
public interface IReportFormatter { byte[] Format(Report report); }
public class PdfFormatter : IReportFormatter { ... }
public class ExcelFormatter : IReportFormatter { ... }
public class CsvFormatter : IReportFormatter { ... } // extends without modifying

public class ReportExporter
{
    public byte[] Export(Report r, IReportFormatter formatter)
        => formatter.Format(r); // unchanged when new formatters are added
}
```
</details>

---

**Q5.** What is JWT? What are the three parts of a JWT token?

<details>
<summary>✅ Answer</summary>

JWT (JSON Web Token) is a compact, URL-safe token format for representing claims securely between parties. It's stateless — the server doesn't need to store session data.

**Three parts** (separated by `.`):
1. **Header** — algorithm and token type: `{ "alg": "HS256", "typ": "JWT" }`
2. **Payload** — claims: `{ "sub": "user-id", "email": "user@example.com", "exp": 1234567890, "iss": "BudgetPH" }`
3. **Signature** — `HMACSHA256(base64(header) + "." + base64(payload), secretKey)` — proves the token wasn't tampered with

**Flow:** Client sends `Authorization: Bearer {token}` → Server validates signature → Server reads claims from payload.
</details>

---

**Q6.** What is the difference between `401 Unauthorized` and `403 Forbidden`?

<details>
<summary>✅ Answer</summary>

- **401 Unauthorized** — the request lacks valid authentication credentials. The user has NOT proven who they are. In practice: no token, expired token, invalid token signature. Response should include `WWW-Authenticate` header.

- **403 Forbidden** — the user IS authenticated (valid token) but does NOT have permission to access the resource. The server understood the request and knows who you are, but you're not allowed.

"401 = you need to log in; 403 = you're logged in but can't do that"
</details>

---

**Q7.** Explain what `async`/`await` does under the hood in C#.

<details>
<summary>✅ Answer</summary>

The C# compiler transforms `async` methods into a **state machine**. When `await` is encountered:
1. If the awaited task is already complete → continues synchronously
2. If not complete → the method **suspends** (returns control to the caller without blocking the thread)
3. The thread is released back to the thread pool
4. When the task completes, the continuation is scheduled to run (possibly on a different thread)

Key points:
- `async`/`await` does NOT create new threads for I/O operations
- It enables concurrency on a single thread by releasing it during waits
- The compiler generates a state machine class with fields for local variables and the current state
</details>

---

**Q8.** What is the difference between SQL's `WHERE` and `HAVING` clauses?

<details>
<summary>✅ Answer</summary>

- `WHERE` — filters **individual rows** before grouping. Operates on raw column values.
- `HAVING` — filters **groups** after `GROUP BY`. Operates on aggregate functions.

```sql
-- WHERE: filter rows before grouping
SELECT CategoryId, SUM(Amount) AS Total
FROM Transactions
WHERE TransactionType = 2  -- ← filters rows
GROUP BY CategoryId

-- HAVING: filter groups after aggregating
SELECT CategoryId, SUM(Amount) AS Total
FROM Transactions
GROUP BY CategoryId
HAVING SUM(Amount) > 1000  -- ← filters groups
```
</details>

---

**Q9.** What is database normalization? Briefly explain 1NF, 2NF, and 3NF.

<details>
<summary>✅ Answer</summary>

Normalization reduces data redundancy and improves integrity.

- **1NF (First Normal Form)**: Atomic values only (no arrays, no repeating groups). Every row is unique (has a primary key).
- **2NF (Second Normal Form)**: 1NF + every non-key attribute depends on the **entire** primary key (no partial dependencies). Applies only when the key is composite.
- **3NF (Third Normal Form)**: 2NF + no transitive dependencies (non-key column A depends on non-key column B which depends on the key). Example: ZipCode → City, State should be a separate table.
</details>

---

**Q10.** What is the Repository pattern and what problem does it solve?

<details>
<summary>✅ Answer</summary>

The Repository pattern provides an abstraction layer over data access. It:
1. **Decouples** business logic from persistence technology (EF Core, Dapper, etc.)
2. Makes business logic **testable** (can inject a fake/mock repository)
3. Provides a consistent, domain-focused API for data operations

```csharp
// Business logic depends on interface, not EF Core directly
public class AccountService
{
    private readonly IAccountRepository _repo; // interface
    
    // In tests: inject FakeAccountRepository
    // In production: inject EfAccountRepository
}
```
</details>

---

**Q11.** What is the purpose of `CancellationToken` in ASP.NET Core? How does it improve performance?

<details>
<summary>✅ Answer</summary>

`CancellationToken` allows an operation to be cancelled cooperatively. In ASP.NET Core:
- The framework provides a `CancellationToken` that is triggered when the client disconnects or the request is cancelled
- Passing this to EF Core queries (`ToListAsync(ct)`) and `HttpClient` calls (`GetAsync(url, ct)`) causes them to cancel the operation immediately
- This frees database connections, threads, and resources that would otherwise finish unnecessary work
- Improves server throughput — cancelled requests release resources instead of completing silently

```csharp
[HttpGet]
public async Task<IActionResult> GetReports(CancellationToken ct) // injected by framework
{
    var data = await context.Transactions.ToListAsync(ct); // cancels if client disconnects
    return Ok(data);
}
```
</details>

---

**Q12.** Explain the Strategy design pattern with an example relevant to a financial application.

<details>
<summary>✅ Answer</summary>

The Strategy pattern defines a family of algorithms, encapsulates each one, and makes them interchangeable.

```csharp
public interface IInterestCalculationStrategy
{
    decimal Calculate(decimal principal, decimal annualRate, int months);
}

public class SimpleInterest : IInterestCalculationStrategy
{
    public decimal Calculate(decimal p, decimal r, int m) => p * r * m / 12;
}

public class CompoundInterest : IInterestCalculationStrategy
{
    public decimal Calculate(decimal p, decimal r, int m)
        => p * (decimal)Math.Pow((double)(1 + r / 12), m) - p;
}

public class LoanCalculator
{
    private IInterestCalculationStrategy _strategy;
    public void SetStrategy(IInterestCalculationStrategy s) => _strategy = s;
    public decimal CalculateInterest(decimal p, decimal r, int m) => _strategy.Calculate(p, r, m);
}
```

The calculator doesn't need to change when new interest calculation methods are added.
</details>

---

**Q13.** What is a clustered index vs a non-clustered index in SQL Server?

<details>
<summary>✅ Answer</summary>

- **Clustered index**: Determines the **physical order** of rows in the table. There can be only **1** per table. The leaf level IS the actual data. Usually the Primary Key. When you `SELECT * FROM Accounts WHERE Id = @id`, the clustered index is used to locate and return the row directly.

- **Non-clustered index**: A separate structure that contains the indexed column(s) and a pointer (row locator) back to the actual data row. There can be up to 999 per table. Used for columns frequently used in `WHERE`, `JOIN`, or `ORDER BY` clauses.

A **covering index** is a non-clustered index that includes all columns needed by a query (via `INCLUDE`) to avoid the extra lookup to the clustered index.
</details>

---

**Q14.** What is the difference between `Task`, `Task<T>`, and `ValueTask<T>`?

<details>
<summary>✅ Answer</summary>

- **`Task`** — represents an async operation with no result. Used for `async void`-style operations.
- **`Task<T>`** — represents an async operation that returns a value of type T. Always allocates a heap object.
- **`ValueTask<T>`** — a struct that avoids heap allocation when the result is synchronously available (e.g., cache hit). Cannot be awaited multiple times. Use in hot paths where the result is frequently synchronous.

```csharp
// ValueTask is ideal here — often returns cached value without async
public ValueTask<ExchangeRate> GetRateAsync(string pair)
{
    if (_cache.TryGetValue(pair, out var rate))
        return ValueTask.FromResult(rate); // no heap allocation
    return new ValueTask<ExchangeRate>(FetchFromApiAsync(pair));
}
```
</details>

---

**Q15.** What is React's `useCallback` hook and when would you use it?

<details>
<summary>✅ Answer</summary>

`useCallback` returns a **memoized function** that only changes if one of its dependencies changes. It prevents creating a new function reference on every render.

**Use when:**
- Passing a callback function as a prop to a `React.memo` child component (prevents unnecessary re-renders of the child)
- The function is a dependency of another hook (like `useEffect`) and you want to control when the effect re-runs

```jsx
// Without useCallback — new function on every render → child always re-renders
const handleDelete = (id) => deleteAccount(id);

// With useCallback — stable reference
const handleDelete = useCallback((id) => {
    deleteAccount(id);
}, []); // no deps = created once

<AccountCard onDelete={handleDelete} /> // React.memo child won't re-render
```
</details>

---

## Section B: Code Writing (Questions 16–25)

**Q16.** Write a LINQ query that groups transactions by month, calculates total income and total expenses per month, and returns months where expenses exceeded income.

<details>
<summary>✅ Answer</summary>

```csharp
var excessMonths = transactions
    .GroupBy(t => new { t.TransactionDate.Year, t.TransactionDate.Month })
    .Select(g => new
    {
        g.Key.Year,
        g.Key.Month,
        TotalIncome = g.Where(t => t.TransactionType == TransactionType.Income).Sum(t => t.Amount),
        TotalExpenses = g.Where(t => t.TransactionType == TransactionType.Expense).Sum(t => t.Amount)
    })
    .Where(m => m.TotalExpenses > m.TotalIncome)
    .OrderByDescending(m => m.Year)
    .ThenByDescending(m => m.Month)
    .ToList();
```
</details>

---

**Q17.** Implement a thread-safe Singleton class for an exchange rate cache.

<details>
<summary>✅ Answer</summary>

```csharp
public sealed class ExchangeRateCache
{
    // Lazy<T> is thread-safe by default
    private static readonly Lazy<ExchangeRateCache> _instance = 
        new(() => new ExchangeRateCache(), LazyThreadSafetyMode.ExecutionAndPublication);

    public static ExchangeRateCache Instance => _instance.Value;

    private readonly ConcurrentDictionary<string, (decimal Rate, DateTime CachedAt)> _rates = new();
    private readonly TimeSpan _expiry = TimeSpan.FromMinutes(15);

    private ExchangeRateCache() { }

    public bool TryGetRate(string pair, out decimal rate)
    {
        rate = 0;
        if (!_rates.TryGetValue(pair, out var entry)) return false;
        if (DateTime.UtcNow - entry.CachedAt > _expiry) return false;
        rate = entry.Rate;
        return true;
    }

    public void SetRate(string pair, decimal rate)
        => _rates[pair] = (rate, DateTime.UtcNow);
}

// Better in ASP.NET Core — use IMemoryCache injected as Singleton
```
</details>

---

**Q18.** Write a generic `Result<T>` class that represents either a success with a value or a failure with an error message.

<details>
<summary>✅ Answer</summary>

```csharp
public class Result<T>
{
    public bool IsSuccess { get; }
    public bool IsFailure => !IsSuccess;
    public T? Value { get; }
    public string? Error { get; }

    private Result(bool isSuccess, T? value, string? error)
    {
        IsSuccess = isSuccess;
        Value = value;
        Error = error;
    }

    public static Result<T> Success(T value) 
        => new(true, value, null);
    
    public static Result<T> Failure(string error) 
        => new(false, default, error);

    // Functional map — transform value if success
    public Result<TOut> Map<TOut>(Func<T, TOut> mapper)
        => IsSuccess ? Result<TOut>.Success(mapper(Value!)) : Result<TOut>.Failure(Error!);

    // Implicit conversions for ergonomics
    public static implicit operator Result<T>(T value) => Success(value);
}

// Non-generic version
public class Result
{
    public bool IsSuccess { get; }
    public string? Error { get; }
    
    private Result(bool isSuccess, string? error) { IsSuccess = isSuccess; Error = error; }
    
    public static Result Success() => new(true, null);
    public static Result Failure(string error) => new(false, error);
}
```
</details>

---

**Q19.** Write an ASP.NET Core action filter that logs the execution time of every controller action.

<details>
<summary>✅ Answer</summary>

```csharp
public class ExecutionTimeFilter : IActionFilter
{
    private readonly ILogger<ExecutionTimeFilter> _logger;
    private Stopwatch? _stopwatch;

    public ExecutionTimeFilter(ILogger<ExecutionTimeFilter> logger)
        => _logger = logger;

    public void OnActionExecuting(ActionExecutingContext context)
    {
        _stopwatch = Stopwatch.StartNew();
        _logger.LogInformation("Starting {Action}", context.ActionDescriptor.DisplayName);
    }

    public void OnActionExecuted(ActionExecutedContext context)
    {
        _stopwatch?.Stop();
        _logger.LogInformation(
            "Completed {Action} in {ElapsedMs}ms | Status: {Status}",
            context.ActionDescriptor.DisplayName,
            _stopwatch?.ElapsedMilliseconds,
            context.HttpContext.Response.StatusCode);
    }
}

// Register globally
builder.Services.AddControllers(options =>
    options.Filters.Add<ExecutionTimeFilter>());
```
</details>

---

**Q20.** Write a SQL query using window functions to find the top 3 expense categories per user for the current month.

<details>
<summary>✅ Answer</summary>

```sql
WITH CategoryTotals AS (
    SELECT 
        t.UserId,
        c.Name AS CategoryName,
        SUM(t.Amount) AS TotalSpent,
        ROW_NUMBER() OVER (
            PARTITION BY t.UserId 
            ORDER BY SUM(t.Amount) DESC
        ) AS Rank
    FROM Transactions t
    INNER JOIN Categories c ON t.CategoryId = c.Id
    WHERE 
        t.TransactionType = 2  -- Expense
        AND YEAR(t.TransactionDate) = YEAR(GETUTCDATE())
        AND MONTH(t.TransactionDate) = MONTH(GETUTCDATE())
    GROUP BY t.UserId, c.Name
)
SELECT UserId, CategoryName, TotalSpent, Rank
FROM CategoryTotals
WHERE Rank <= 3
ORDER BY UserId, Rank;
```
</details>

---

**Q21.** Implement a simple circuit breaker in C# without using Polly.

<details>
<summary>✅ Answer</summary>

```csharp
public enum CircuitState { Closed, Open, HalfOpen }

public class CircuitBreaker
{
    private CircuitState _state = CircuitState.Closed;
    private int _failureCount = 0;
    private DateTime _openedAt = DateTime.MinValue;
    
    private readonly int _failureThreshold;
    private readonly TimeSpan _openDuration;
    private readonly object _lock = new();

    public CircuitBreaker(int failureThreshold = 5, int openDurationSeconds = 30)
    {
        _failureThreshold = failureThreshold;
        _openDuration = TimeSpan.FromSeconds(openDurationSeconds);
    }

    public async Task<T> ExecuteAsync<T>(Func<Task<T>> action)
    {
        lock (_lock)
        {
            if (_state == CircuitState.Open)
            {
                if (DateTime.UtcNow - _openedAt >= _openDuration)
                    _state = CircuitState.HalfOpen; // try again
                else
                    throw new InvalidOperationException("Circuit is OPEN");
            }
        }

        try
        {
            var result = await action();
            OnSuccess();
            return result;
        }
        catch
        {
            OnFailure();
            throw;
        }
    }

    private void OnSuccess()
    {
        lock (_lock)
        {
            _failureCount = 0;
            _state = CircuitState.Closed;
        }
    }

    private void OnFailure()
    {
        lock (_lock)
        {
            _failureCount++;
            if (_failureCount >= _failureThreshold)
            {
                _state = CircuitState.Open;
                _openedAt = DateTime.UtcNow;
            }
        }
    }
}
```
</details>

---

**Q22.** Write a React custom hook `useDebounce` that delays updating a value until a specified delay has passed.

<details>
<summary>✅ Answer</summary>

```jsx
import { useState, useEffect } from 'react';

function useDebounce(value, delay = 300) {
    const [debouncedValue, setDebouncedValue] = useState(value);

    useEffect(() => {
        // Set a timer to update the debounced value
        const timer = setTimeout(() => {
            setDebouncedValue(value);
        }, delay);

        // Cleanup: cancel timer if value changes before delay expires
        return () => clearTimeout(timer);
    }, [value, delay]);

    return debouncedValue;
}

// Usage in search
function TransactionSearch() {
    const [search, setSearch] = useState('');
    const debouncedSearch = useDebounce(search, 400); // 400ms delay

    // Only fires query after user stops typing for 400ms
    const { data } = useQuery({
        queryKey: ['transactions', debouncedSearch],
        queryFn: () => transactionsService.getAll({ search: debouncedSearch }),
        enabled: debouncedSearch.length >= 2,
    });

    return <input value={search} onChange={e => setSearch(e.target.value)} />;
}
```
</details>

---

**Q23.** Implement a middleware in ASP.NET Core that adds security response headers to every response.

<details>
<summary>✅ Answer</summary>

```csharp
public class SecurityHeadersMiddleware
{
    private readonly RequestDelegate _next;

    public SecurityHeadersMiddleware(RequestDelegate next)
        => _next = next;

    public async Task InvokeAsync(HttpContext context)
    {
        // Add headers before the response starts
        context.Response.OnStarting(() =>
        {
            var headers = context.Response.Headers;
            
            headers["X-Content-Type-Options"] = "nosniff";
            headers["X-Frame-Options"] = "DENY";
            headers["X-XSS-Protection"] = "1; mode=block";
            headers["Referrer-Policy"] = "strict-origin-when-cross-origin";
            headers["Permissions-Policy"] = "camera=(), microphone=(), geolocation=()";
            headers["Content-Security-Policy"] = 
                "default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline';";
            
            // Remove server info headers
            headers.Remove("Server");
            headers.Remove("X-Powered-By");
            
            return Task.CompletedTask;
        });

        await _next(context);
    }
}

// Extension method
public static IApplicationBuilder UseSecurityHeaders(this IApplicationBuilder app)
    => app.UseMiddleware<SecurityHeadersMiddleware>();

// Register
app.UseSecurityHeaders(); // before UseAuthentication
```
</details>

---

**Q24.** Write a unit test for a `BudgetService.AddExpense()` method that verifies:
1. The spent amount is updated correctly
2. A `BudgetExceededEvent` is published when limit is exceeded

<details>
<summary>✅ Answer</summary>

```csharp
using Xunit;
using Moq;
using FluentAssertions;

public class BudgetServiceTests
{
    private readonly Mock<IBudgetRepository> _mockRepo;
    private readonly Mock<IEventPublisher> _mockPublisher;
    private readonly BudgetService _service;

    public BudgetServiceTests()
    {
        _mockRepo = new Mock<IBudgetRepository>();
        _mockPublisher = new Mock<IEventPublisher>();
        _service = new BudgetService(_mockRepo.Object, _mockPublisher.Object);
    }

    [Fact]
    public async Task AddExpense_ShouldUpdateSpentAmount()
    {
        // Arrange
        var budgetItem = new BudgetItem
        {
            Id = Guid.NewGuid(),
            AllocatedAmount = 5000m,
            SpentAmount = 1000m
        };
        
        _mockRepo.Setup(r => r.GetBudgetItemAsync(It.IsAny<Guid>(), It.IsAny<CancellationToken>()))
            .ReturnsAsync(budgetItem);

        // Act
        await _service.AddExpenseAsync(budgetItem.Id, 2000m, CancellationToken.None);

        // Assert
        budgetItem.SpentAmount.Should().Be(3000m);
        _mockRepo.Verify(r => r.SaveChangesAsync(It.IsAny<CancellationToken>()), Times.Once);
    }

    [Fact]
    public async Task AddExpense_WhenBudgetExceeded_ShouldPublishEvent()
    {
        // Arrange
        var budgetItem = new BudgetItem
        {
            Id = Guid.NewGuid(),
            AllocatedAmount = 5000m,
            SpentAmount = 4500m // already 4500 of 5000
        };
        
        _mockRepo.Setup(r => r.GetBudgetItemAsync(It.IsAny<Guid>(), It.IsAny<CancellationToken>()))
            .ReturnsAsync(budgetItem);

        // Act
        await _service.AddExpenseAsync(budgetItem.Id, 600m, CancellationToken.None); // exceeds by 100

        // Assert
        budgetItem.SpentAmount.Should().Be(5100m);
        _mockPublisher.Verify(p => p.PublishAsync(
            It.Is<BudgetExceededEvent>(e => 
                e.BudgetItemId == budgetItem.Id && 
                e.ExceededBy == 100m),
            It.IsAny<CancellationToken>()), Times.Once);
    }
}
```
</details>

---

**Q25.** Given a REST API endpoint `GET /api/transactions`, add proper pagination, filtering by type and date range, and sorting. Show both the controller action signature and the response shape.

<details>
<summary>✅ Answer</summary>

```csharp
// Query parameters model
public class TransactionQueryParams
{
    public int Page { get; set; } = 1;
    public int PageSize { get; set; } = 20;
    public TransactionType? Type { get; set; }
    public DateTime? StartDate { get; set; }
    public DateTime? EndDate { get; set; }
    public string? Search { get; set; }
    public string SortBy { get; set; } = "transactionDate";
    public string SortDir { get; set; } = "desc";
}

// Controller action
[HttpGet]
public async Task<IActionResult> GetAll([FromQuery] TransactionQueryParams query, CancellationToken ct)
{
    var appUser = await GetAppUserAsync(ct);

    var q = context.Transactions
        .AsNoTracking()
        .Where(t => t.UserId == appUser!.Id);

    if (query.Type.HasValue) q = q.Where(t => t.TransactionType == query.Type.Value);
    if (query.StartDate.HasValue) q = q.Where(t => t.TransactionDate >= query.StartDate.Value);
    if (query.EndDate.HasValue) q = q.Where(t => t.TransactionDate <= query.EndDate.Value);
    if (!string.IsNullOrWhiteSpace(query.Search))
        q = q.Where(t => t.Description.Contains(query.Search));

    q = query.SortBy.ToLower() switch
    {
        "amount" => query.SortDir == "asc" ? q.OrderBy(t => t.Amount) : q.OrderByDescending(t => t.Amount),
        _ => query.SortDir == "asc" ? q.OrderBy(t => t.TransactionDate) : q.OrderByDescending(t => t.TransactionDate)
    };

    var total = await q.CountAsync(ct);
    var items = await q
        .Skip((query.Page - 1) * query.PageSize)
        .Take(query.PageSize)
        .Select(t => new TransactionDto { Id = t.Id, Description = t.Description, Amount = t.Amount })
        .ToListAsync(ct);

    return Ok(new
    {
        data = items,
        pagination = new
        {
            page = query.Page,
            pageSize = query.PageSize,
            total,
            totalPages = (int)Math.Ceiling((double)total / query.PageSize),
            hasNextPage = query.Page * query.PageSize < total,
            hasPreviousPage = query.Page > 1
        }
    });
}

// Response shape:
// {
//   "data": [...],
//   "pagination": { "page": 1, "pageSize": 20, "total": 150, "totalPages": 8, ... }
// }
```
</details>
