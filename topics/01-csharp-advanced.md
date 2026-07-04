# 01 — C# Advanced Features

> 🔗 **BudgetPH reference**: [BaseApiController.cs](https://github.com/randolfsegubre/BudgetPH/blob/master/src/Presentation/FinanceManager.API/Controllers/BaseApiController.cs)

---

## Table of Contents
1. [Generics & Constraints](#1-generics--constraints)
2. [Delegates, Func, Action, Predicate](#2-delegates-func-action-predicate)
3. [Events](#3-events)
4. [LINQ — Deep Dive](#4-linq--deep-dive)
5. [async / await & Task](#5-async--await--task)
6. [Pattern Matching](#6-pattern-matching)
7. [Records, Structs, and Classes](#7-records-structs-and-classes)
8. [Nullable Reference Types](#8-nullable-reference-types)
9. [Extension Methods](#9-extension-methods)
10. [IDisposable & Memory Management](#10-idisposable--memory-management)
11. [Common Interview Questions](#11-common-interview-questions)

---

## 1. Generics & Constraints

### What are Generics?
Generics allow you to write **type-safe, reusable code** without sacrificing performance. They avoid boxing/unboxing for value types and eliminate the need for casting.

```csharp
// Without generics — requires boxing for value types
public object GetFirst(object[] items) => items[0];

// With generics — type-safe, no boxing
public T GetFirst<T>(T[] items) => items[0];
```

### Generic Constraints

| Constraint | Meaning |
|---|---|
| `where T : class` | T must be a reference type |
| `where T : struct` | T must be a value type (non-nullable) |
| `where T : new()` | T must have a parameterless constructor |
| `where T : SomeBaseClass` | T must inherit from SomeBaseClass |
| `where T : ISomeInterface` | T must implement ISomeInterface |
| `where T : notnull` | T must be a non-nullable type |
| `where T : unmanaged` | T must be an unmanaged type (int, double, etc.) |

```csharp
// Multiple constraints
public class Repository<T> where T : class, new()
{
    public T CreateNew() => new T();
}

// Combining constraints
public T FindOrCreate<T>(IEnumerable<T> items, Func<T, bool> predicate)
    where T : class, new()
{
    return items.FirstOrDefault(predicate) ?? new T();
}
```

### 🔗 BudgetPH Example
The `Result<T>` class in the Application layer uses generics:
```csharp
// src/Core/FinanceManager.Application/Common/Models/Result.cs
public class Result<T>
{
    public bool IsSuccess { get; }
    public T? Value { get; }
    public string? Error { get; }
    
    private Result(bool isSuccess, T? value, string? error)
    {
        IsSuccess = isSuccess; Value = value; Error = error;
    }
    
    public static Result<T> Success(T value) => new(true, value, null);
    public static Result<T> Failure(string error) => new(false, default, error);
}
```

### Covariance and Contravariance
- **Covariance** (`out`): A generic type parameter is **more derived** than specified. Used in `IEnumerable<out T>`.
- **Contravariance** (`in`): A generic type parameter is **less derived** than specified. Used in `Action<in T>`.

```csharp
// Covariance: IEnumerable<string> can be assigned to IEnumerable<object>
IEnumerable<string> strings = new List<string>();
IEnumerable<object> objects = strings; // ✅ covariant

// Contravariance: Action<object> can be assigned to Action<string>
Action<object> printObject = obj => Console.WriteLine(obj);
Action<string> printString = printObject; // ✅ contravariant

// Your own covariant interface
public interface IProducer<out T>
{
    T Produce();
}

// Your own contravariant interface
public interface IConsumer<in T>
{
    void Consume(T item);
}
```

---

## 2. Delegates, Func, Action, Predicate

### Delegates — The Foundation
A delegate is a **type-safe function pointer** — a variable that holds a reference to a method.

```csharp
// Declare delegate type
public delegate int MathOperation(int a, int b);

// Create instance
MathOperation add = (a, b) => a + b;
MathOperation multiply = (a, b) => a * b;

Console.WriteLine(add(3, 4));       // 7
Console.WriteLine(multiply(3, 4));  // 12
```

### Func, Action, Predicate — Built-in Delegates

| Delegate | Signature | Purpose |
|---|---|---|
| `Func<T, TResult>` | Has return value | Transform a value |
| `Action<T>` | Returns void | Perform side effect |
| `Predicate<T>` | Returns bool | Test a condition |

```csharp
// Func<TIn, TOut> — takes input, returns output
Func<string, int> getLength = s => s.Length;
Func<int, int, int> add = (a, b) => a + b;

// Action<T> — takes input, returns nothing
Action<string> print = s => Console.WriteLine(s);
Action<string, int> printN = (s, n) => Console.WriteLine($"{s} {n}");

// Predicate<T> — takes input, returns bool
Predicate<int> isEven = n => n % 2 == 0;

// Using with LINQ
var numbers = new[] { 1, 2, 3, 4, 5 };
var evens = numbers.Where(n => isEven(n)); // [2, 4]
```

### Multicast Delegates
```csharp
Action<string> log = Console.WriteLine;
log += s => File.AppendAllText("log.txt", s); // add another handler

log("Hello"); // calls BOTH handlers
log -= Console.WriteLine; // remove one
```

---

## 3. Events

Events are built on top of delegates, following the **publish-subscribe** pattern.

```csharp
public class BudgetTracker
{
    // Declare event using EventHandler<T>
    public event EventHandler<BudgetExceededArgs>? BudgetExceeded;

    private decimal _spent;
    private decimal _limit;

    public void AddExpense(decimal amount)
    {
        _spent += amount;
        if (_spent > _limit)
        {
            // Raise the event
            BudgetExceeded?.Invoke(this, new BudgetExceededArgs(_spent, _limit));
        }
    }
}

public class BudgetExceededArgs : EventArgs
{
    public decimal Spent { get; }
    public decimal Limit { get; }
    public BudgetExceededArgs(decimal spent, decimal limit) 
    { Spent = spent; Limit = limit; }
}

// Subscribe
var tracker = new BudgetTracker();
tracker.BudgetExceeded += (sender, e) => 
    Console.WriteLine($"Budget exceeded! Spent: {e.Spent}, Limit: {e.Limit}");
```

**Events vs Delegates:**
- Events can only be invoked from within the declaring class
- Events enforce the publish-subscribe pattern (callers can only `+=` or `-=`)
- Delegates can be invoked by anyone holding a reference

---

## 4. LINQ — Deep Dive

### Deferred vs Immediate Execution

This is **critical** to understand and frequently tested.

```csharp
var numbers = new List<int> { 1, 2, 3, 4, 5 };

// DEFERRED — query is not executed yet
IEnumerable<int> query = numbers.Where(n => n > 2); // ← no execution here

numbers.Add(6); // adding AFTER the query definition

// Execution happens HERE (foreach, ToList, Count, etc.)
foreach (var n in query) Console.WriteLine(n); // prints 3, 4, 5, 6 ← includes 6!

// IMMEDIATE — executes now
List<int> result = numbers.Where(n => n > 2).ToList(); // executes immediately
numbers.Add(7);
// result does NOT include 7
```

**Methods that force immediate execution:** `ToList()`, `ToArray()`, `ToDictionary()`, `Count()`, `Sum()`, `First()`, `Single()`, `Any()`, `All()`

### Method Syntax vs Query Syntax

```csharp
// Method syntax (lambda-based) — most commonly used
var result = transactions
    .Where(t => t.Amount > 1000)
    .OrderByDescending(t => t.TransactionDate)
    .Select(t => new { t.Description, t.Amount })
    .Take(10);

// Query syntax (SQL-like)
var result = from t in transactions
             where t.Amount > 1000
             orderby t.TransactionDate descending
             select new { t.Description, t.Amount };
```

### Essential LINQ Methods

```csharp
var items = new[] { 1, 2, 3, 4, 5 };

// Filtering
items.Where(x => x > 2)                // [3, 4, 5]

// Projection
items.Select(x => x * 2)              // [2, 4, 6, 8, 10]
items.SelectMany(x => new[] { x, x })  // [1,1,2,2,3,3,4,4,5,5] — flattens

// Ordering
items.OrderBy(x => x)
items.OrderByDescending(x => x)
items.ThenBy(x => ...)   // secondary sort

// Aggregation
items.Count()             // 5
items.Sum()               // 15
items.Average()           // 3.0
items.Min()               // 1
items.Max()               // 5
items.Aggregate((a, b) => a + b)  // 15 (custom reduce)

// Element retrieval
items.First()             // 1 (throws if empty)
items.FirstOrDefault()    // 1 (returns default if empty)
items.Single()            // throws if not exactly one
items.ElementAt(2)        // 3

// Quantifiers
items.Any(x => x > 4)     // true
items.All(x => x > 0)     // true
items.Contains(3)         // true

// Partitioning
items.Take(3)             // [1, 2, 3]
items.Skip(2)             // [3, 4, 5]
items.TakeLast(2)         // [4, 5]
items.SkipLast(2)         // [1, 2, 3]

// Grouping
items.GroupBy(x => x % 2 == 0 ? "even" : "odd")

// Set operations
var a = new[] { 1, 2, 3 };
var b = new[] { 2, 3, 4 };
a.Intersect(b)  // [2, 3]
a.Union(b)      // [1, 2, 3, 4]
a.Except(b)     // [1]
a.Distinct()    // removes duplicates

// Joining
var orders = orders.Join(customers,
    o => o.CustomerId,
    c => c.Id,
    (o, c) => new { o.Amount, c.Name });
```

### 🔗 BudgetPH LINQ Example

From [ApplicationDbContext.cs](https://github.com/randolfsegubre/BudgetPH/blob/master/src/Infrastructure/FinanceManager.Infrastructure/Data/ApplicationDbContext.cs):
```csharp
// Grouping and aggregation in EF LINQ
var summary = await context.Transactions
    .Where(t => t.UserId == userId && t.TransactionDate.Year == year)
    .GroupBy(t => t.TransactionDate.Month)
    .Select(g => new MonthlySummary
    {
        Month = g.Key,
        TotalIncome = g.Where(t => t.TransactionType == TransactionType.Income).Sum(t => t.Amount),
        TotalExpense = g.Where(t => t.TransactionType == TransactionType.Expense).Sum(t => t.Amount)
    })
    .ToListAsync();
```

---

## 5. async / await & Task

### The Mental Model
`async`/`await` does **not** create new threads by default. It releases the calling thread while waiting for I/O, then resumes when the I/O completes. The thread pool handles the continuation.

```csharp
// WRONG — synchronously blocks thread
public string GetData()
{
    var result = httpClient.GetStringAsync(url).Result; // ⚠️ DEADLOCK RISK
    return result;
}

// RIGHT — async all the way
public async Task<string> GetDataAsync()
{
    var result = await httpClient.GetStringAsync(url); // releases thread while waiting
    return result;
}
```

### Task vs ValueTask

| | `Task<T>` | `ValueTask<T>` |
|---|---|---|
| Allocation | Always allocates heap object | Stack allocation when result is synchronous |
| Use case | Most async methods | Hot paths where result is often synchronous (caching) |
| Can await multiple times | ✅ Yes | ❌ No (unless reset) |

```csharp
// ValueTask is optimal when result is often cached
public ValueTask<User> GetUserAsync(int id)
{
    if (_cache.TryGetValue(id, out var user))
        return ValueTask.FromResult(user); // no allocation!
    
    return new ValueTask<User>(FetchFromDbAsync(id));
}
```

### ConfigureAwait(false)

In library code (non-UI), use `ConfigureAwait(false)` to avoid capturing the synchronization context:

```csharp
// Library/infrastructure code
public async Task<string> FetchDataAsync()
{
    var data = await httpClient.GetStringAsync(url).ConfigureAwait(false);
    // ^ continues on thread pool, not original sync context
    return Process(data);
}
```

**Rule:** Use `ConfigureAwait(false)` in library/infrastructure code. In ASP.NET Core controllers (no sync context), it's not needed but also doesn't hurt.

### Parallel Async Operations

```csharp
// SEQUENTIAL — slow (waits for each before starting next)
var a = await GetAAsync();
var b = await GetBAsync();

// PARALLEL — both start simultaneously
var taskA = GetAAsync();
var taskB = GetBAsync();
await Task.WhenAll(taskA, taskB);
var a = taskA.Result; // already done
var b = taskB.Result; // already done

// Get first result
var firstResult = await Task.WhenAny(taskA, taskB);
```

### CancellationToken

Always propagate `CancellationToken` in ASP.NET Core controllers — it's provided by the framework when a request is cancelled:

```csharp
// 🔗 Pattern used throughout BudgetPH controllers
[HttpGet]
public async Task<IActionResult> GetAccounts(CancellationToken ct)
{
    var accounts = await context.Accounts
        .Where(a => a.UserId == appUser.Id)
        .ToListAsync(ct); // ← pass CancellationToken here
    return Ok(accounts);
}
```

### Common Async Pitfalls

```csharp
// ❌ DEADLOCK — .Result or .Wait() in synchronous code that has a SyncContext
var result = GetDataAsync().Result;    // deadlock in ASP.NET (old), WPF, WinForms
var result = GetDataAsync().GetAwaiter().GetResult(); // still a potential deadlock

// ❌ ASYNC VOID — swallows exceptions
public async void ProcessData() { ... } // don't do this except for event handlers

// ✅ CORRECT
public async Task ProcessDataAsync() { ... }

// ❌ Fire-and-forget without error handling
_ = ProcessInBackgroundAsync(); // exceptions silently swallowed

// ✅ Better fire-and-forget
_ = ProcessInBackgroundAsync().ContinueWith(t => 
    logger.LogError(t.Exception, "Background task failed"), 
    TaskContinuationOptions.OnlyOnFaulted);
```

---

## 6. Pattern Matching

C# 8–12 introduced powerful pattern matching constructs:

```csharp
// Type patterns
object obj = "hello";
if (obj is string s) Console.WriteLine(s.Length); // s is scoped to if block

// Switch expressions (C# 8+)
string category = accountType switch
{
    AccountType.Checking or AccountType.Savings => "Bank",
    AccountType.CreditCard => "Credit",
    AccountType.Investment => "Investment",
    AccountType.Loan => "Debt",
    _ => "Other"
};

// Property patterns
bool isHighValue = transaction switch
{
    { Amount: > 10000, TransactionType: TransactionType.Expense } => true,
    { IsRecurring: true, Amount: > 5000 } => true,
    _ => false
};

// Positional patterns (with Deconstruct)
public record Point(int X, int Y);
string quadrant = point switch
{
    (> 0, > 0) => "Q1",
    (< 0, > 0) => "Q2",
    (< 0, < 0) => "Q3",
    (> 0, < 0) => "Q4",
    _ => "Origin"
};

// List patterns (C# 11+)
int[] nums = { 1, 2, 3 };
bool matches = nums is [1, 2, ..]; // starts with 1, 2
bool single = nums is [var only]; // exactly one element

// Guard clauses in patterns
string describe = n switch
{
    int x when x < 0 => "negative",
    0 => "zero",
    int x when x > 0 => "positive"
};
```

---

## 7. Records, Structs, and Classes

### When to Use Each

| Type | Heap/Stack | Equality | Immutable by default | Use For |
|------|-----------|----------|----------------------|---------|
| `class` | Heap | Reference | No | Entities, services |
| `struct` | Stack (usually) | Value | No | Small value types (Point, Color) |
| `record class` | Heap | Value | Yes (init-only) | DTOs, immutable models |
| `record struct` | Stack | Value | No (mutable) | Performance-critical value objects |

```csharp
// record — value equality, immutability, with-expressions
public record TransactionDto(Guid Id, string Description, decimal Amount, DateTime Date);

var dto1 = new TransactionDto(Guid.NewGuid(), "Salary", 50000, DateTime.Now);
var dto2 = dto1 with { Amount = 55000 }; // creates a copy with one change
Console.WriteLine(dto1 == dto2); // false (different Amount)

// Non-destructive mutation
var updated = original with { Description = "Updated" };

// record with additional members
public record AccountDto(Guid Id, string Name, decimal Balance)
{
    public string DisplayBalance => $"₱{Balance:N2}";
}
```

### Init-only Properties

```csharp
public class Config
{
    public string ConnectionString { get; init; } = ""; // can only be set during initialization
}

var config = new Config { ConnectionString = "Server=..." }; // ✅
config.ConnectionString = "other"; // ❌ Compile error
```

---

## 8. Nullable Reference Types

Enabled in all modern .NET projects via `<Nullable>enable</Nullable>` in .csproj.

```csharp
// Without NRT — runtime NullReferenceException possible
string name = GetName();  // could be null

// With NRT enabled
string name = GetName();   // compiler promises NOT null
string? name = GetName();  // compiler warns: could be null

// Null-forgiving operator (use sparingly)
string name = GetName()!;  // tells compiler "I know it's not null"

// Null-coalescing
string display = name ?? "Anonymous";
string display = name ?? throw new ArgumentNullException(nameof(name));

// Null-conditional
int? length = name?.Length;          // null if name is null
string? first = list?.FirstOrDefault();

// Null-coalescing assignment
name ??= "Default"; // assign only if null
```

### Nullability Annotations for APIs

```csharp
// 🔗 Pattern used in BudgetPH DTOs
public class AccountDto
{
    public Guid Id { get; set; }
    public string Name { get; set; } = "";          // non-nullable, must initialize
    public string? AccountNumber { get; set; }      // nullable — can be null
    public string? Notes { get; set; }
    public CreditCardSummaryDto? CreditCard { get; set; } // null for non-CC accounts
}
```

---

## 9. Extension Methods

Extension methods allow you to add methods to existing types without modifying them.

```csharp
// Define in a static class
public static class StringExtensions
{
    public static bool IsValidEmail(this string str)
    {
        return str.Contains('@') && str.Contains('.');
    }
    
    public static string ToTitleCase(this string str)
    {
        return CultureInfo.CurrentCulture.TextInfo.ToTitleCase(str.ToLower());
    }
}

// Usage — looks like an instance method
bool valid = "user@example.com".IsValidEmail(); // true
string title = "hello world".ToTitleCase(); // "Hello World"

// Extension on IQueryable (EF Core pattern)
public static class QueryableExtensions
{
    public static IQueryable<T> ApplyPaging<T>(this IQueryable<T> query, int page, int pageSize)
        => query.Skip((page - 1) * pageSize).Take(pageSize);
}

// Usage
var transactions = context.Transactions.ApplyPaging(page: 2, pageSize: 20);
```

---

## 10. IDisposable & Memory Management

### IDisposable Pattern

```csharp
// Implement when class holds unmanaged resources (file handles, connections, etc.)
public class DatabaseConnection : IDisposable
{
    private SqlConnection? _connection;
    private bool _disposed = false;

    public DatabaseConnection(string connStr)
    {
        _connection = new SqlConnection(connStr);
        _connection.Open();
    }

    public void Dispose()
    {
        Dispose(true);
        GC.SuppressFinalize(this); // tell GC finalizer not needed
    }

    protected virtual void Dispose(bool disposing)
    {
        if (_disposed) return;
        
        if (disposing)
        {
            // Dispose managed resources
            _connection?.Dispose();
            _connection = null;
        }
        
        _disposed = true;
    }

    ~DatabaseConnection() // finalizer — last resort if Dispose not called
    {
        Dispose(false);
    }
}

// Usage — always use 'using'
using var conn = new DatabaseConnection(connStr); // auto-disposed at end of scope
// OR
using (var conn = new DatabaseConnection(connStr))
{
    // use conn
} // conn.Dispose() called here
```

### IAsyncDisposable (modern .NET)

```csharp
public class AsyncResource : IAsyncDisposable
{
    private Stream? _stream;

    public async ValueTask DisposeAsync()
    {
        if (_stream != null)
        {
            await _stream.DisposeAsync();
            _stream = null;
        }
    }
}

await using var resource = new AsyncResource(); // async disposal
```

---

## 11. Common Interview Questions

**Q1: What is the difference between `IEnumerable<T>` and `IQueryable<T>`?**

> `IEnumerable<T>` executes queries **in-memory** (after data is retrieved from DB). `IQueryable<T>` translates LINQ to SQL and executes **on the database server**. Always use `IQueryable<T>` with EF Core to get efficient SQL generation; convert to `IEnumerable<T>` only after all filtering is applied.

**Q2: What does `async`/`await` actually do under the hood?**

> The C# compiler transforms `async` methods into a state machine. When `await` is encountered on an incomplete `Task`, the method suspends, returning control to the caller. When the awaited task completes, the state machine resumes from where it left off (potentially on a different thread). No new threads are created for I/O operations.

**Q3: What is the difference between `Task.WhenAll` and `Task.WhenAny`?**

> `Task.WhenAll` waits for **all** tasks to complete. `Task.WhenAny` completes as soon as **any one** task completes. Use `WhenAll` for parallel independent operations; use `WhenAny` for timeout patterns or "first successful result" patterns.

**Q4: Explain covariance and contravariance with examples.**

> Covariance (`out`): You can use a more-derived type where a base is expected. `IEnumerable<string>` can be assigned to `IEnumerable<object>`. Contravariance (`in`): You can use a less-derived type where a more-derived is expected. An `Action<Animal>` can be used as `Action<Dog>`.

**Q5: When would you use `ValueTask` over `Task`?**

> Use `ValueTask<T>` in hot paths where the result is **often already available synchronously** (e.g., reading from a cache). It avoids heap allocation when the result doesn't require async execution. Don't use it for fire-and-forget or when the value will be awaited multiple times.

**Q6: What is the difference between `==` and `Equals()` for value types and reference types?**

> For **reference types**: `==` compares references by default (same object in memory). `Equals()` can be overridden to compare values. For **value types** and `record` types: `==` and `Equals()` compare values. String is a reference type that overrides `==` to compare values.

**Q7: What happens if you don't call `Dispose()` on a `DbContext`?**

> The `DbContext` holds a database connection, change tracker, and other resources. If not disposed, these linger until garbage collection. In ASP.NET Core, `DbContext` registered as `Scoped` is automatically disposed at the end of each HTTP request — which is why you should NOT use `Singleton` lifetime for `DbContext`.

**Q8: What is the difference between `where T : class` and `where T : notnull`?**

> `where T : class` restricts T to reference types (allows nullable reference types). `where T : notnull` restricts T to non-nullable types — either non-nullable reference types OR value types (includes structs). This is useful with nullable reference types enabled.

**Q9: Can you explain LINQ's `SelectMany`?**

> `SelectMany` is used to **flatten** a collection of collections into a single collection. Example: given `IEnumerable<Order>` where each `Order` has `IEnumerable<OrderItem>`, `orders.SelectMany(o => o.Items)` returns a flat `IEnumerable<OrderItem>`.

**Q10: What is the `record` type and when should you use it?**

> A `record` is a reference type with **value-based equality** (equality based on property values, not reference). Records auto-generate `Equals`, `GetHashCode`, `ToString`, and support `with`-expressions for non-destructive mutation. Use records for **DTOs, immutable data models, and Value Objects** — exactly as used in BudgetPH's DTO layer.
