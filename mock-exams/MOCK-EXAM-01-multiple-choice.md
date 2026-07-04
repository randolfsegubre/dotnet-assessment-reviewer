# Mock Exam 1 — Multiple Choice (50 Questions)

> ⏱️ **Suggested time: 40 minutes** | Each question shows the answer and explanation below it.  
> Try to answer mentally before reading the solution.

---

## C# Language (Questions 1–15)

---

**Q1.** What is the output of the following code?
```csharp
int a = 5;
int b = a++;
Console.WriteLine($"{a} {b}");
```
- A) `5 5`
- B) `6 5`
- C) `5 6`
- D) `6 6`

✅ **Answer: B — `6 5`**

> `a++` is **post-increment**: it returns the current value first, then increments.  
> So `b` gets `5` (the current value of `a`), then `a` becomes `6`.
> ```csharp
> int a = 5;
> int b = a++;        // b = 5, THEN a becomes 6
> // vs pre-increment:
> int c = ++a;        // a becomes 6 first, THEN c = 6
> Console.WriteLine($"{a} {b}"); // "6 5"
> ```

---

**Q2.** Which correctly declares a generic method where `T` must be a reference type with a parameterless constructor?
- A) `public T Create<T>() where T : new()`
- B) `public T Create<T>() where T : class`
- C) `public T Create<T>() where T : class, new()`
- D) `public T Create<T>() where T : struct, new()`

✅ **Answer: C — `where T : class, new()`**

> You need **both** constraints. `class` = reference type; `new()` = has a public parameterless constructor. Order matters: `class` must come before `new()`.
> ```csharp
> public T Create<T>() where T : class, new()
> {
>     return new T(); // allowed because of new() constraint
> }
> 
> var user = Create<ApplicationUser>(); // works
> var num  = Create<int>();             // compile error — int is struct
> ```

---

**Q3.** What is the difference between `Task.WhenAll` and `Task.WhenAny`?
- A) `WhenAll` runs tasks sequentially; `WhenAny` runs in parallel
- B) `WhenAll` completes when all tasks complete; `WhenAny` completes when the first task completes
- C) `WhenAll` completes when the first task completes; `WhenAny` completes when all complete
- D) There is no difference

✅ **Answer: B**

> Both start tasks **in parallel**. The difference is when they return:
> ```csharp
> var t1 = FetchAccountsAsync();   // 2 seconds
> var t2 = FetchBudgetsAsync();    // 3 seconds
> var t3 = FetchGoalsAsync();      // 1 second
> 
> await Task.WhenAll(t1, t2, t3);  // waits 3s — all must finish
> await Task.WhenAny(t1, t2, t3);  // waits 1s — returns when t3 finishes
> 
> // Timeout pattern with WhenAny:
> var completed = await Task.WhenAny(actualWork, Task.Delay(5000));
> if (completed != actualWork) throw new TimeoutException();
> ```

---

**Q4.** Which LINQ method forces immediate execution?
- A) `Where()`
- B) `Select()`
- C) `OrderBy()`
- D) `ToList()`

✅ **Answer: D — `ToList()`**

> `Where()`, `Select()`, `OrderBy()` are **deferred** — they build an expression tree and only execute when enumerated. `ToList()`, `ToArray()`, `Count()`, `First()`, `Any()` force immediate execution.
> ```csharp
> // Deferred — nothing runs yet:
> IEnumerable<int> query = numbers.Where(n => n > 0).Select(n => n * 2);
> 
> // Immediate — evaluates now:
> List<int> result = query.ToList();
> int count  = query.Count();   // runs the query AGAIN
> bool any   = query.Any();     // runs the query AGAIN
> ```

---

**Q5.** What is a `record` type in C#?
- A) A class that supports inheritance and reference equality
- B) A reference type with value-based equality and immutability support
- C) A value type stored on the stack
- D) An alias for `struct`

✅ **Answer: B**

> `record` (C# 9+) is a **reference type** (heap) with **value-based equality** — two records with the same property values are considered equal. Supports non-destructive mutation via `with`.
> ```csharp
> record Money(decimal Amount, string Currency);
> 
> var a = new Money(100, "PHP");
> var b = new Money(100, "PHP");
> 
> Console.WriteLine(a == b);               // true  (value equality)
> Console.WriteLine(ReferenceEquals(a, b)); // false (different objects on heap)
> 
> var c = a with { Amount = 200 };  // non-destructive copy — a is unchanged
> // c = Money(200, "PHP"), a = Money(100, "PHP")
> ```

---

**Q6.** What does `ConfigureAwait(false)` do?
- A) Prevents the async method from being awaited
- B) Configures the method to not capture the current synchronization context
- C) Runs the method synchronously
- D) Disables cancellation support

✅ **Answer: B**

> By default, `await` resumes on the original SynchronizationContext (UI thread, ASP.NET request context). `ConfigureAwait(false)` skips context capture — use this in library code.
> ```csharp
> // Library code — use ConfigureAwait(false) to avoid deadlocks:
> public async Task<string> GetDataAsync()
> {
>     var result = await httpClient.GetStringAsync(url).ConfigureAwait(false);
>     // continues on any thread pool thread — no context captured
>     return result;
> }
> 
> // ASP.NET Core controllers — no need (no SyncContext in .NET Core):
> public async Task<IActionResult> GetUser()
> {
>     var user = await _service.GetAsync(); // fine without ConfigureAwait
>     return Ok(user);
> }
> ```

---

**Q7.** What is the output?
```csharp
var numbers = new List<int> { 1, 2, 3 };
IEnumerable<int> query = numbers.Where(n => n > 1);
numbers.Add(4);
Console.WriteLine(query.Count());
```
- A) `2`
- B) `3`
- C) `4`
- D) `1`

✅ **Answer: B — `3`**

> `Where()` is **deferred** — the query doesn't run at declaration. It runs when `Count()` is called, by which time `numbers` is `[1, 2, 3, 4]`. Items > 1 are: 2, 3, 4 → **3**.
> ```csharp
> // To capture state immediately:
> List<int> snapshot = numbers.Where(n => n > 1).ToList(); // [2, 3]
> numbers.Add(4);
> Console.WriteLine(snapshot.Count()); // 2 — captured at ToList() time
> ```

---

**Q8.** What is the correct way to raise an event safely?
- A) `MyEvent.Invoke(this, EventArgs.Empty)`
- B) `MyEvent?.Invoke(this, EventArgs.Empty)`
- C) `if (MyEvent != null) MyEvent(this, EventArgs.Empty)`
- D) Both B and C are correct (though B is preferred)

✅ **Answer: D — Both B and C, but B is preferred**

> Option C has a **race condition** in multithreaded code: the event could become null between the null check and the invocation. `?.Invoke` atomically checks and invokes.
> ```csharp
> public event EventHandler? TransactionAdded;
> 
> // ✅ Thread-safe, preferred:
> TransactionAdded?.Invoke(this, EventArgs.Empty);
> 
> // ⚠️ Race condition risk in multithreaded code:
> if (TransactionAdded != null)
>     TransactionAdded(this, EventArgs.Empty); // could be null here
> ```

---

**Q9.** What is `SelectMany` used for?
- A) Selecting multiple properties from an object
- B) Flattening a collection of collections into a single collection
- C) Selecting the top N items
- D) Performing multiple projections

✅ **Answer: B — Flattening collections**

> `SelectMany` projects each element to a sequence, then **flattens** all into one.
> ```csharp
> List<Account> accounts = GetAccounts();
> 
> // Select gives nested collections:
> IEnumerable<IEnumerable<Transaction>> nested = accounts.Select(a => a.Transactions);
> 
> // SelectMany flattens into one list:
> IEnumerable<Transaction> all = accounts.SelectMany(a => a.Transactions);
> 
> // Real use — all expense transactions across all accounts:
> var expenses = accounts
>     .SelectMany(a => a.Transactions)
>     .Where(t => t.Type == TransactionType.Expense)
>     .Sum(t => t.Amount); // total expenses across all accounts
> ```

---

**Q10.** Which statement about `async void` is true?
- A) It's equivalent to `async Task`
- B) Exceptions from `async void` cannot be caught by the caller and will crash the app
- C) It should be used for all async methods that don't return a value
- D) It is the preferred pattern for event handlers that call async code

✅ **Answer: B**

> `async void` **cannot be awaited**. Exceptions propagate to the SynchronizationContext, often crashing the app. Use `async Task` everywhere except event handlers.
> ```csharp
> // ❌ Never do this:
> public async void SaveData()
> {
>     await _repo.SaveAsync(); // if this throws, app can crash
> }
> try { SaveData(); } catch { } // exception is NOT caught here!
> 
> // ✅ Always return Task:
> public async Task SaveDataAsync()
> {
>     await _repo.SaveAsync(); // caller can await and catch
> }
> 
> // ✅ Only acceptable async void — event handlers:
> private async void SaveButton_Click(object sender, EventArgs e)
> {
>     try { await SaveDataAsync(); }
>     catch (Exception ex) { ShowError(ex); }
> }
> ```

---

**Q11.** Is this valid? `IEnumerable<object> objects = strings;` (where `strings` is `IEnumerable<string>`)
- A) No — this always fails
- B) Yes — because `IEnumerable<T>` is covariant with `out T`
- C) Yes — because `List<T>` is covariant
- D) No — covariance only works with arrays

✅ **Answer: B — Yes, `IEnumerable<T>` is covariant**

> **Covariance** (`out T`) allows using a more-derived type. `IEnumerable<out T>` is declared covariant. `List<T>` is NOT covariant (invariant).
> ```csharp
> IEnumerable<string> strings = new List<string> { "PHP", "USD" };
> IEnumerable<object> objects = strings;  // ✅ covariance works
> 
> List<string> strList = new();
> List<object> objList = strList;         // ❌ compile error — List<T> is invariant
> 
> // IEnumerable<out T> — T only appears in output (return positions)
> // IComparer<in T>    — T only appears in input (parameter positions) = contravariance
> ```

---

**Q12.** What does the `??=` operator do?
- A) Compares two nullable values
- B) Assigns the right operand to the left only if the left is `null`
- C) Returns the first non-null value between two operands
- D) Converts null to the default value

✅ **Answer: B — Null-coalescing assignment**

> `??=` assigns only when the left side is null. `??` (no `=`) returns the right if left is null but doesn't assign.
> ```csharp
> string? name = null;
> name ??= "Default";       // assigns "Default" because name is null
> name ??= "Other";         // does NOT assign — name is "Default"
> Console.WriteLine(name);  // "Default"
> 
> // Common pattern — lazy initialization:
> private List<string>? _cache;
> public List<string> GetCache() => _cache ??= LoadFromDb();
> //                                            ^ only called when _cache is null
> ```

---

**Q13.** What is the difference between `==` and `Equals()` for a `record` type?
- A) `==` compares references; `Equals()` compares values
- B) Both compare values — `record` overrides both to be value-based
- C) `==` compares values; `Equals()` compares references
- D) There is no difference for any type

✅ **Answer: B — Both are value-based for records**

> `record` auto-generates `==`, `!=`, `Equals()`, and `GetHashCode()` based on all properties.
> ```csharp
> record Person(string Name, int Age);
> 
> var p1 = new Person("Randolf", 30);
> var p2 = new Person("Randolf", 30);
> 
> p1 == p2                 // true  — value equality
> p1.Equals(p2)            // true  — value equality
> ReferenceEquals(p1, p2)  // false — different objects on heap
> 
> // Compare with class:
> class PersonClass { public string Name { get; set; } public int Age { get; set; } }
> var c1 = new PersonClass { Name = "Randolf", Age = 30 };
> var c2 = new PersonClass { Name = "Randolf", Age = 30 };
> c1 == c2  // false — reference equality (different objects)
> ```

---

**Q14.** Which statement is true about `IDisposable`?
- A) It automatically garbage-collects the object when out of scope
- B) Implementing `IDisposable` guarantees the finalizer will be called
- C) `using` statement calls `Dispose()` at the end of the block
- D) `Dispose()` is called automatically by the GC

✅ **Answer: C**

> The `using` statement is syntactic sugar for `try/finally { obj.Dispose() }`. The GC does NOT call `Dispose()` — it calls the **finalizer** (`~ClassName()`), which is a different mechanism.
> ```csharp
> // using statement:
> using var conn = new SqlConnection(connStr);
> // Dispose() called automatically at end of scope
> 
> // Equivalent:
> var conn = new SqlConnection(connStr);
> try { /* use */ }
> finally { conn.Dispose(); }
> 
> // Correct IDisposable implementation:
> public class DbService : IDisposable
> {
>     private bool _disposed = false;
>     
>     public void Dispose()
>     {
>         if (!_disposed)
>         {
>             _connection?.Dispose(); // free unmanaged resources
>             _disposed = true;
>         }
>         GC.SuppressFinalize(this); // tell GC: finalizer not needed
>     }
> }
> ```

---

**Q15.** What is the purpose of the `init` keyword?
```csharp
public string Name { get; init; }
```
- A) The property can only be set in the class constructor
- B) The property can only be set during object initialization (constructor or initializer), never after
- C) The property is read-only
- D) The property is initialized with its default value

✅ **Answer: B**

> `init` is like `set` but restricted to construction time (constructor OR object initializer syntax). This enables immutable-but-constructable objects.
> ```csharp
> public class Transaction
> {
>     public Guid Id { get; init; } = Guid.NewGuid();
>     public decimal Amount { get; init; }
> }
> 
> // ✅ Object initializer — allowed:
> var tx = new Transaction { Amount = 500 };
> 
> // ❌ After construction — compile error:
> tx.Amount = 600;
> 
> // Used with records for non-destructive mutation:
> var tx2 = tx with { Amount = 600 }; // new object, tx unchanged
> ```

---

## .NET Core & ASP.NET Core (Questions 16–25)

---

**Q16.** Which DI lifetime creates a new instance every time it is requested?
- A) Singleton
- B) Scoped
- C) Transient
- D) Per-Request

✅ **Answer: C — Transient**

> | Lifetime | New instance | Shared within |
> |---|---|---|
> | **Singleton** | Once per app lifetime | Everything — always same instance |
> | **Scoped** | Once per HTTP request | Same request — different requests get different instances |
> | **Transient** | Every `GetService<T>()` call | Never shared |
>
> ```csharp
> builder.Services.AddSingleton<IConfigService, ConfigService>();   // 1 instance ever
> builder.Services.AddScoped<ICurrentUserService, UserService>();   // 1 per request
> builder.Services.AddTransient<IEmailSender, EmailSender>();       // new every call
> ```

---

**Q17.** What is the "captive dependency" problem?
- A) A Singleton service depends on another Singleton
- B) A Transient service depends on a Scoped service
- C) A Singleton service captures a Scoped service, causing incorrect shared state
- D) A Scoped service depends on a Transient service

✅ **Answer: C**

> A **Singleton** that takes a **Scoped** dependency traps the Scoped instance forever — it never gets a new instance per request, breaking Scoped's contract.
> ```csharp
> // ❌ Captive dependency — AppDbContext is Scoped, CacheService is Singleton:
> public class CacheService  // registered as Singleton
> {
>     public CacheService(AppDbContext db) { } // db is Scoped — WRONG!
>     // This DbContext will never be disposed — lives forever with the Singleton
> }
> 
> // ✅ Fix — use IServiceScopeFactory to create scopes on demand:
> public class CacheService
> {
>     private readonly IServiceScopeFactory _scopeFactory;
>     public CacheService(IServiceScopeFactory f) => _scopeFactory = f;
> 
>     public async Task RefreshAsync()
>     {
>         using var scope = _scopeFactory.CreateScope();
>         var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
>         // db is properly scoped — disposed when scope is disposed
>     }
> }
> ```

---

**Q18.** In which order must middleware be registered in `Program.cs`?
- A) `UseAuthentication` → `UseAuthorization` → `UseCors`
- B) `UseCors` → `UseRouting` → `UseAuthentication` → `UseAuthorization`
- C) `UseAuthorization` → `UseAuthentication` → `UseCors`
- D) Order doesn't matter

✅ **Answer: B**

> Middleware order is **critical**. The correct sequence:
> ```csharp
> app.UseExceptionHandler();   // 1. catch all exceptions first
> app.UseHttpsRedirection();   // 2. redirect to HTTPS
> app.UseStaticFiles();        // 3. serve files without auth
> app.UseCors();               // 4. CORS headers BEFORE auth
> app.UseRouting();            // 5. match routes
> app.UseAuthentication();     // 6. who are you? (sets HttpContext.User)
> app.UseAuthorization();      // 7. what can you do? (reads HttpContext.User)
> app.MapControllers();        // 8. execute the matched endpoint
> ```
> Wrong order = silent failures. `UseAuthorization` before `UseAuthentication` = user always anonymous.

---

**Q19.** What does `[ApiController]` attribute add over `[Controller]`?
- A) Nothing — they are the same
- B) Automatic 400 responses from model validation, binding source inference, Problem Details support
- C) Automatic database access
- D) Swagger documentation

✅ **Answer: B**

> `[ApiController]` provides automatic behaviors for Web APIs:
> ```csharp
> [ApiController]
> [Route("api/[controller]")]
> public class TransactionsController : ControllerBase
> {
>     // ✅ Auto 400 if Amount is missing — no ModelState.IsValid check needed:
>     [HttpPost]
>     public IActionResult Create(CreateTransactionRequest request) { ... }
>     
>     // ✅ Binding source inference — Guid id from route, no [FromRoute] needed:
>     [HttpGet("{id}")]
>     public IActionResult Get(Guid id) { ... }
>     
>     // ✅ Problem Details format for errors automatically
> }
> ```
> Without `[ApiController]`: model validation errors are silently ignored, you'd need `if (!ModelState.IsValid) return BadRequest()` manually.

---

**Q20.** What is the difference between `IOptions<T>`, `IOptionsSnapshot<T>`, and `IOptionsMonitor<T>`?
- A) They are interchangeable
- B) `IOptions<T>` is Singleton; `IOptionsSnapshot<T>` is Scoped (reloads per request); `IOptionsMonitor<T>` is Singleton with change notification
- C) All three are Scoped
- D) `IOptions<T>` is the newest

✅ **Answer: B**

> ```csharp
> // IOptions<T> — Singleton, reads config once at startup, never updates:
> public class MyService(IOptions<JwtSettings> opts)
> {
>     var secret = opts.Value.SecretKey; // same value forever, even if appsettings changes
> }
> 
> // IOptionsSnapshot<T> — Scoped, reloads config each HTTP request:
> public class MyController(IOptionsSnapshot<FeatureFlags> flags)
> {
>     var enabled = flags.Value.NewDashboard; // fresh per request
> }
> 
> // IOptionsMonitor<T> — Singleton, live change notification:
> public class Worker(IOptionsMonitor<WorkerSettings> monitor)
> {
>     monitor.OnChange(settings =>
>         _interval = settings.IntervalSeconds); // called when config file changes
> }
> ```

---

**Q21.** What does `app.Run()` do in the middleware pipeline?
- A) Starts the web application
- B) Adds a terminal middleware that does NOT call the next middleware
- C) Runs all middleware in parallel
- D) Adds middleware that always calls the next middleware

✅ **Answer: B — Terminal middleware**

> | Method | Calls next? | Purpose |
> |---|---|---|
> | `app.Use()` | ✅ Yes | Regular middleware |
> | `app.Run()` | ❌ No | Terminal — ends the pipeline |
> | `app.Map()` | Branches | Conditional branching |
>
> ```csharp
> app.Use(async (ctx, next) =>
> {
>     Console.WriteLine("Before");
>     await next();             // calls next middleware
>     Console.WriteLine("After"); // runs on the way back
> });
> 
> app.Run(async ctx =>
> {
>     await ctx.Response.WriteAsync("Done"); // nothing after this runs
> });
> ```

---

**Q22.** What is `IServiceScopeFactory` used for?
- A) Registering scoped services
- B) Creating new DI scopes from a Singleton service to safely resolve Scoped services
- C) Listing all registered services
- D) Configuring service lifetimes

✅ **Answer: B**

> ```csharp
> // Singleton BackgroundService needs a Scoped DbContext:
> public class TokenCleanupService : BackgroundService
> {
>     private readonly IServiceScopeFactory _scopeFactory;
>     
>     protected override async Task ExecuteAsync(CancellationToken ct)
>     {
>         while (!ct.IsCancellationRequested)
>         {
>             using (var scope = _scopeFactory.CreateScope())
>             {
>                 var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
>                 await db.RefreshTokens
>                     .Where(t => t.ExpiresAt < DateTime.UtcNow)
>                     .ExecuteDeleteAsync(ct);
>             } // scope disposed here — DbContext properly cleaned up
>             
>             await Task.Delay(TimeSpan.FromHours(1), ct);
>         }
>     }
> }
> ```

---

**Q23.** Which is the correct way to read nested config value `"JwtSettings:SecretKey"`?
- A) `config.GetSection("JwtSettings:SecretKey")`
- B) `config["JwtSettings:SecretKey"]`
- C) `config.GetValue("JwtSettings").GetValue("SecretKey")`
- D) `config.Get<JwtSettings>().SecretKey`

✅ **Answer: B — `config["JwtSettings:SecretKey"]`**

> The indexer with `:` separator is the standard way. Both B and a variation of D work.
> ```csharp
> // ✅ Indexer with colon separator:
> string secret = config["JwtSettings:SecretKey"]!;
> 
> // ✅ GetSection then indexer:
> string secret = config.GetSection("JwtSettings")["SecretKey"]!;
> 
> // ✅ Best practice — bind to typed options:
> builder.Services.Configure<JwtSettings>(builder.Configuration.GetSection("JwtSettings"));
> // Then inject IOptions<JwtSettings> in constructors
> ```

---

**Q24.** What HTTP status code does ASP.NET Core return when model validation fails (with `[ApiController]`)?
- A) 500
- B) 422
- C) 400
- D) 404

✅ **Answer: C — 400 Bad Request**

> `[ApiController]` automatically returns `400 Bad Request` with a `ProblemDetails` response when model validation fails.
> ```json
> {
>   "type": "https://tools.ietf.org/html/rfc9110#section-15.5.1",
>   "title": "One or more validation errors occurred.",
>   "status": 400,
>   "errors": {
>     "Amount": ["The Amount field is required.", "Amount must be greater than 0."]
>   }
> }
> ```
> ```csharp
> // With [ApiController] — no manual check needed:
> [HttpPost]
> public IActionResult Create(CreateTransactionDto dto)
> {
>     // if dto fails validation, 400 is returned before reaching here
>     return Ok();
> }
> ```

---

**Q25.** Which is the correct way to create a background service that loops?
- A) Implement `IHostedService.StartAsync` with an infinite loop
- B) Extend `BackgroundService` and override `ExecuteAsync` with a `while (!stoppingToken.IsCancellationRequested)` loop
- C) Use `Task.Run` in `Startup.Configure`
- D) Use `Thread.Sleep` in a Singleton service constructor

✅ **Answer: B**

> ```csharp
> public class DailyReportService : BackgroundService
> {
>     private readonly IServiceScopeFactory _scopeFactory;
>     private readonly ILogger<DailyReportService> _logger;
> 
>     protected override async Task ExecuteAsync(CancellationToken stoppingToken)
>     {
>         while (!stoppingToken.IsCancellationRequested)
>         {
>             try
>             {
>                 using var scope = _scopeFactory.CreateScope();
>                 var service = scope.ServiceProvider.GetRequiredService<IReportService>();
>                 await service.GenerateDailyAsync(stoppingToken);
>             }
>             catch (Exception ex) when (ex is not OperationCanceledException)
>             {
>                 _logger.LogError(ex, "Daily report failed");
>             }
>             
>             await Task.Delay(TimeSpan.FromHours(24), stoppingToken);
>         }
>     }
> }
> 
> // Register:
> builder.Services.AddHostedService<DailyReportService>();
> ```

---

## Entity Framework Core (Questions 26–33)

---

**Q26.** What is the N+1 problem in EF Core?
- A) A query that returns N+1 columns
- B) Loading N parent entities then issuing 1 query per entity to load related data
- C) Having N+1 migrations applied
- D) An index with N+1 columns

✅ **Answer: B**

> N+1 = 1 query for the list + N extra queries (one per item). Fix with `Include()`.
> ```csharp
> // ❌ N+1 — 101 queries if 100 accounts:
> var accounts = await db.Accounts.ToListAsync();  // 1 query
> foreach (var a in accounts)
>     var count = a.Transactions.Count;            // 1 query per account!
> 
> // ✅ Fix — eager loading (1 query with JOIN):
> var accounts = await db.Accounts
>     .Include(a => a.Transactions)
>     .ToListAsync();
> 
> // ✅ Best — project only what you need:
> var summaries = await db.Accounts
>     .Select(a => new { a.Name, TxCount = a.Transactions.Count })
>     .ToListAsync();
> ```

---

**Q27.** What does `AsNoTracking()` do?
- A) Prevents the entity from being saved
- B) Disables change tracking — faster for read-only queries
- C) Makes the query execute asynchronously
- D) Prevents lazy loading

✅ **Answer: B**

> EF Core tracks entities in a `ChangeTracker` to detect updates. For read-only scenarios, skip tracking for 30–50% better performance.
> ```csharp
> // ❌ Tracked — overhead for display-only data:
> var transactions = await db.Transactions.ToListAsync();
> 
> // ✅ No tracking — faster for read-only:
> var transactions = await db.Transactions
>     .AsNoTracking()
>     .Where(t => t.UserId == userId && t.Date >= startDate)
>     .OrderByDescending(t => t.Date)
>     .Take(50)
>     .ToListAsync();
> ```

---

**Q28.** What is the difference between `Find()` and `FirstOrDefault()`?
- A) `Find()` is async; `FirstOrDefault()` is sync
- B) `Find()` checks the ChangeTracker before querying the DB; `FirstOrDefault()` always queries
- C) `Find()` throws if not found; `FirstOrDefault()` returns null
- D) There is no difference

✅ **Answer: B**

> ```csharp
> // FindAsync — checks ChangeTracker first (memory), then DB if needed:
> var account = await db.Accounts.FindAsync(id);
> // Fast if the entity was already loaded in this request
> 
> // FirstOrDefaultAsync — ALWAYS hits the database:
> var account = await db.Accounts.FirstOrDefaultAsync(a => a.Id == id);
> 
> // Use Find() when: updating entities, same entity accessed multiple times per request
> // Use FirstOrDefault() when: filtering by non-PK, need Include(), need AsNoTracking()
> ```

---

**Q29.** Which EF Core feature loads related entities in the same query?
- A) Lazy Loading
- B) Explicit Loading
- C) Eager Loading (`Include()`)
- D) Deferred Loading

✅ **Answer: C — Eager Loading**

> | Strategy | When loaded | SQL queries | Best for |
> |---|---|---|---|
> | **Eager** (`Include`) | Same query | 1 JOIN | Always-needed related data |
> | **Lazy** (proxy) | On first access | N extra queries | Rare — causes N+1 |
> | **Explicit** (`Load()`) | Manually | Extra when called | Conditionally-needed data |
>
> ```csharp
> // Eager loading with nested includes:
> var account = await db.Accounts
>     .Include(a => a.Transactions)
>         .ThenInclude(t => t.Category)
>     .FirstOrDefaultAsync(a => a.Id == id);
> // Single SQL query with JOINs
> ```

---

**Q30.** When should you use `AsSplitQuery()`?
- A) When you want to run queries on a separate thread
- B) When including multiple collection navigation properties to avoid Cartesian explosion
- C) When splitting a transaction across multiple databases
- D) When ordering by multiple columns

✅ **Answer: B**

> Without `AsSplitQuery()`, multiple collection `Include()`s create a JOIN that multiplies rows (Cartesian product).
> ```csharp
> // ❌ Single query — Cartesian explosion:
> // 1000 transactions × 50 budgets = 50,000 rows returned for just 1 account!
> var account = await db.Accounts
>     .Include(a => a.Transactions)
>     .Include(a => a.Budgets)
>     .FirstOrDefaultAsync(a => a.Id == id);
> 
> // ✅ Split query — 3 separate SELECTs, no row multiplication:
> var account = await db.Accounts
>     .Include(a => a.Transactions)
>     .Include(a => a.Budgets)
>     .AsSplitQuery()
>     .FirstOrDefaultAsync(a => a.Id == id);
> ```

---

**Q31.** What happens if you call `SaveChanges()` and one operation fails?
- A) Only the successful operations are saved
- B) EF Core automatically retries
- C) All operations are rolled back (single transaction)
- D) The exception is swallowed

✅ **Answer: C — All rolled back**

> `SaveChanges()` wraps all pending changes in a **single DB transaction**. If any SQL fails, everything rolls back.
> ```csharp
> try
> {
>     db.Transactions.Add(newTx);       // debit
>     db.Accounts.Update(account);       // update balance
>     db.AuditLogs.Add(auditEntry);      // audit trail
>     
>     await db.SaveChangesAsync();       // all 3 in one transaction
>     // If account update fails → ALL THREE roll back → consistency maintained
> }
> catch (DbUpdateException ex)
> {
>     _logger.LogError(ex, "Save failed — all changes rolled back");
> }
> ```

---

**Q32.** What is a migration in EF Core?
- A) Moving data from one table to another
- B) A code representation of incremental database schema changes, with `Up()` and `Down()` methods
- C) A tool for exporting data to CSV
- D) A type of index

✅ **Answer: B**

> Migrations track schema changes as code. `Up()` applies, `Down()` reverts.
> ```csharp
> // After adding Currency to Transaction entity, run:
> // dotnet ef migrations add AddCurrencyToTransaction
> 
> // Generated code:
> public partial class AddCurrencyToTransaction : Migration
> {
>     protected override void Up(MigrationBuilder mb)
>     {
>         mb.AddColumn<string>("Currency", "Transactions", defaultValue: "PHP");
>     }
>     protected override void Down(MigrationBuilder mb)
>     {
>         mb.DropColumn("Currency", "Transactions");
>     }
> }
> 
> // Apply:  dotnet ef database update
> // Revert: dotnet ef database update PreviousMigrationName
> ```

---

**Q33.** Which method executes a bulk DELETE without loading entities into memory?
- A) `RemoveRange()` followed by `SaveChanges()`
- B) `ExecuteDeleteAsync()` (EF Core 7+)
- C) `DeleteMany()`
- D) `BulkDelete()`

✅ **Answer: B — `ExecuteDeleteAsync()`**

> `RemoveRange` loads all entities into memory first. `ExecuteDeleteAsync` generates a direct `DELETE` SQL statement.
> ```csharp
> // ❌ Slow — loads all expired tokens into memory:
> var expired = await db.RefreshTokens.Where(t => t.ExpiresAt < DateTime.UtcNow).ToListAsync();
> db.RemoveRange(expired);
> await db.SaveChangesAsync();
> 
> // ✅ Fast — single DELETE SQL, no memory overhead (EF Core 7+):
> await db.RefreshTokens
>     .Where(t => t.ExpiresAt < DateTime.UtcNow)
>     .ExecuteDeleteAsync();
> // SQL: DELETE FROM RefreshTokens WHERE ExpiresAt < GETUTCDATE()
> 
> // Similarly, ExecuteUpdateAsync for bulk updates:
> await db.Transactions
>     .Where(t => t.Status == "Pending" && t.CreatedAt < cutoff)
>     .ExecuteUpdateAsync(t => t.SetProperty(x => x.Status, "Cancelled"));
> ```

---

## Design Patterns & Architecture (Questions 34–42)

---

**Q34.** Which SOLID principle does this violate?
```csharp
public class UserService {
    public User GetUser(int id) { ... }
    public void SendWelcomeEmail(User user) { ... }
    public string GeneratePdfReport(User user) { ... }
}
```
- A) Open/Closed
- B) Single Responsibility
- C) Liskov Substitution
- D) Interface Segregation

✅ **Answer: B — Single Responsibility Principle**

> A class should have **one reason to change**. `UserService` has three responsibilities — data access, email, reporting.
> ```csharp
> // ✅ SRP-compliant — one class, one job:
> public class UserRepository   { public User GetUser(int id) { ... } }
> public class EmailService     { public void SendWelcomeEmail(User u) { ... } }
> public class ReportService    { public string GeneratePdfReport(User u) { ... } }
> 
> // Now:
> // - UserRepository changes when DB schema changes
> // - EmailService changes when email templates change
> // - ReportService changes when report format changes
> // Each class has exactly ONE reason to change
> ```

---

**Q35.** What is the purpose of the Repository pattern?
- A) Store data in multiple databases
- B) Abstract data access behind an interface, decoupling business logic from persistence
- C) Cache frequently-accessed data
- D) Implement the Command pattern

✅ **Answer: B**

> ```csharp
> // Interface in Domain/Application layer — no EF Core references:
> public interface ITransactionRepository
> {
>     Task<Transaction?> GetByIdAsync(Guid id, CancellationToken ct);
>     Task<List<Transaction>> GetByAccountAsync(Guid accountId, CancellationToken ct);
>     Task AddAsync(Transaction tx, CancellationToken ct);
> }
> 
> // Implementation in Infrastructure layer — EF Core here:
> public class EfTransactionRepository : ITransactionRepository
> {
>     private readonly AppDbContext _db;
>     public async Task<Transaction?> GetByIdAsync(Guid id, CancellationToken ct)
>         => await _db.Transactions.FindAsync([id], ct);
> }
> 
> // Business logic uses the interface — no EF Core knowledge:
> public class TransactionService(ITransactionRepository repo) { }
> 
> // Benefits:
> // - Swap EF for Dapper without changing business logic
> // - Unit test with FakeTransactionRepository — no DB needed
> ```

---

**Q36.** What is CQRS?
- A) A security pattern for validating commands
- B) A framework for building microservices
- C) Separating read (Query) operations from write (Command) operations into different models
- D) A type of database transaction

✅ **Answer: C**

> ```csharp
> // COMMAND — changes state:
> public record CreateTransactionCommand(Guid AccountId, decimal Amount)
>     : IRequest<Guid>;
> 
> public class CreateTransactionHandler : IRequestHandler<CreateTransactionCommand, Guid>
> {
>     public async Task<Guid> Handle(CreateTransactionCommand cmd, CancellationToken ct)
>     {
>         var tx = new Transaction { Amount = cmd.Amount };
>         _db.Transactions.Add(tx);
>         await _db.SaveChangesAsync(ct);
>         return tx.Id;
>     }
> }
> 
> // QUERY — reads state, never modifies:
> public record GetDashboardQuery(Guid UserId) : IRequest<DashboardDto>;
> 
> public class GetDashboardHandler : IRequestHandler<GetDashboardQuery, DashboardDto>
> {
>     public async Task<DashboardDto> Handle(GetDashboardQuery q, CancellationToken ct)
>         => await _db.Transactions.AsNoTracking() // read-optimized
>                 .Where(t => t.UserId == q.UserId)
>                 .GroupBy(t => t.Type)
>                 .Select(g => /* ... */)
>                 .FirstAsync(ct);
> }
> ```

---

**Q37.** Which Design Pattern does ASP.NET Core's middleware pipeline use?
- A) Observer
- B) Decorator
- C) Chain of Responsibility
- D) Strategy

✅ **Answer: C — Chain of Responsibility**

> Each middleware is a handler that processes the request or passes it to the next handler. The chain terminates when a handler doesn't call `next()`.
> ```csharp
> // Classic Chain of Responsibility in middleware:
> app.UseExceptionHandler();    // handler 1
> app.UseCors();                // handler 2
> app.UseAuthentication();      // handler 3
> app.UseAuthorization();       // handler 4
> app.MapControllers();         // handler 5 — terminal
> 
> // Custom middleware = explicit chain link:
> app.Use(async (context, next) =>
> {
>     // pre-processing
>     await next(context);   // pass to next handler in chain
>     // post-processing (response flowing back)
> });
> ```

---

**Q38.** What does the Decorator pattern do?
- A) Creates a new instance by cloning
- B) Adds new behavior to an object by wrapping it, without modifying the original class
- C) Provides a simplified interface to a complex subsystem
- D) Ensures only one instance exists

✅ **Answer: B**

> ```csharp
> public interface IAccountRepository
> {
>     Task<Account?> GetByIdAsync(Guid id);
> }
> 
> // Original:
> public class EfAccountRepository : IAccountRepository { /* EF Core */ }
> 
> // Decorator — wraps original, adds caching:
> public class CachedAccountRepository : IAccountRepository
> {
>     private readonly IAccountRepository _inner;  // wraps the original
>     private readonly IMemoryCache _cache;
> 
>     public async Task<Account?> GetByIdAsync(Guid id)
>     {
>         return await _cache.GetOrCreateAsync($"account:{id}", async entry =>
>         {
>             entry.AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(5);
>             return await _inner.GetByIdAsync(id); // delegates to EfAccountRepository
>         });
>     }
> }
> 
> // EfAccountRepository is never modified — Open/Closed Principle respected
> ```

---

**Q39.** In Clean Architecture, which layer should NOT depend on any other layer?
- A) Application
- B) Infrastructure
- C) Presentation
- D) Domain

✅ **Answer: D — Domain**

> The **Dependency Rule**: all dependencies point inward toward Domain. Domain knows nothing about databases, HTTP, or frameworks.
> ```
> Presentation  →  Application  →  Domain    ✅
> Infrastructure               →  Domain    ✅
> Domain         →  Infrastructure           ❌ violation!
> 
> // Domain entity — pure C#, zero external dependencies:
> public class Transaction
> {
>     public Guid Id { get; private set; }
>     public decimal Amount { get; private set; }
>     
>     public void ApplyRefund(decimal refundAmount)
>     {
>         if (refundAmount > Amount)
>             throw new DomainException("Refund exceeds transaction amount");
>         Amount -= refundAmount;
>         AddDomainEvent(new RefundAppliedEvent(Id, refundAmount));
>     }
> }
> // No [Table], no DbSet, no EF attributes, no HttpContext — pure business logic
> ```

---

**Q40.** What is a MediatR Pipeline Behavior?
- A) A way to route HTTP requests to controllers
- B) Cross-cutting concern middleware for IRequest handlers (validation, logging, caching)
- C) A message queue implementation
- D) A caching strategy

✅ **Answer: B**

> Pipeline Behaviors wrap every command/query handler — like middleware, but for MediatR.
> ```csharp
> // Validation behavior — runs before EVERY handler:
> public class ValidationBehavior<TRequest, TResponse>
>     : IPipelineBehavior<TRequest, TResponse>
>     where TRequest : IRequest<TResponse>
> {
>     private readonly IEnumerable<IValidator<TRequest>> _validators;
> 
>     public async Task<TResponse> Handle(
>         TRequest request, RequestHandlerDelegate<TResponse> next, CancellationToken ct)
>     {
>         var errors = _validators
>             .SelectMany(v => v.Validate(request).Errors)
>             .Where(e => e is not null)
>             .ToList();
> 
>         if (errors.Any()) throw new ValidationException(errors);
> 
>         return await next(); // call the actual handler
>     }
> }
> 
> // Pipeline:
> // Request → LoggingBehavior → ValidationBehavior → ActualHandler → Response
> ```

---

**Q41.** Which SOLID principle does dependency injection enforce?
- A) Single Responsibility
- B) Open/Closed
- C) Dependency Inversion
- D) Interface Segregation

✅ **Answer: C — Dependency Inversion Principle**

> DIP: High-level modules should depend on **abstractions**, not concrete implementations. DI wires the concrete implementation at runtime.
> ```csharp
> // ❌ Violates DIP:
> public class TransactionService
> {
>     private readonly EfTransactionRepository _repo = new(); // concrete!
> }
> 
> // ✅ Follows DIP:
> public class TransactionService(ITransactionRepository repo) // abstraction
> {
>     // repo could be EF, Dapper, MongoDB, or a mock in tests
> }
> 
> // DI container resolves the concrete type:
> builder.Services.AddScoped<ITransactionRepository, EfTransactionRepository>();
> // To test: inject new FakeTransactionRepository() — no DB needed
> ```

---

**Q42.** What is the difference between Factory Method and Abstract Factory patterns?
- A) Factory Method creates single objects; Abstract Factory creates families of related objects
- B) Abstract Factory is a simplified Factory Method
- C) They are the same pattern
- D) Factory Method uses composition; Abstract Factory uses inheritance

✅ **Answer: A**

> ```csharp
> // Factory Method — one product, subclass decides concrete type:
> public abstract class NotificationSender
> {
>     public abstract INotification Create(); // factory method
>     public void Send(string msg) => Create().Deliver(msg);
> }
> public class SmsNotificationSender : NotificationSender
> {
>     public override INotification Create() => new SmsNotification();
> }
> 
> // Abstract Factory — FAMILY of related products:
> public interface IPhilippineFinanceFactory
> {
>     ICurrencyFormatter CreateFormatter();   // PHP formatter
>     ITaxCalculator CreateTaxCalculator();   // 12% VAT
>     IReportTemplate CreateReport();         // BIR-compliant
> }
> // One factory creates an entire consistent family of objects for a context
> ```

---

## Security, REST & Databases (Questions 43–50)

---

**Q43.** What is the correct HTTP status code when a user is authenticated but lacks permission?
- A) 401 Unauthorized
- B) 403 Forbidden
- C) 404 Not Found
- D) 400 Bad Request

✅ **Answer: B — 403 Forbidden**

> | Code | Meaning | Scenario |
> |---|---|---|
> | **401** | Not authenticated — who are you? | Missing or invalid JWT |
> | **403** | Authenticated but not allowed | User accesses another user's account |
> | **404** | Resource not found (or deliberately hidden) | Sensitive resource you shouldn't know exists |
>
> ```csharp
> [Authorize]  // returns 401 if no valid JWT
> public class AccountsController : BaseApiController
> {
>     [HttpDelete("{id}")]
>     public async Task<IActionResult> Delete(Guid id)
>     {
>         var account = await GetAccountAsync(id);
>         if (account is null) return NotFound(); // 404
>         if (account.UserId != CurrentUserId) return Forbid(); // 403
>         // proceed with delete
>     }
> }
> ```

---

**Q44.** A JWT token consists of:
- A) Header, Claims, Signature
- B) Header, Payload, Signature
- C) Username, Password, Expiry
- D) Issuer, Audience, Subject

✅ **Answer: B — Header, Payload, Signature**

> Three Base64Url-encoded parts separated by `.`:
> ```
> eyJhbGciOiJIUzI1NiJ9                  ← Header  {"alg":"HS256","typ":"JWT"}
> .eyJzdWIiOiJ1c2VyLTEyMyJ9             ← Payload (claims)
> .SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV  ← Signature (HMAC of header+payload)
> ```
> ```csharp
> // Payload claims (decoded):
> {
>   "sub":   "user-guid",         // Subject — user ID
>   "email": "user@example.com",
>   "role":  "Admin",
>   "iss":   "BudgetPH-API",      // Issuer
>   "aud":   "BudgetPH-Client",   // Audience
>   "exp":   1720000000           // Expiry Unix timestamp
> }
> // ⚠️ The payload is readable by anyone (Base64 decoded)
> // The SIGNATURE prevents tampering — only the server with the secret can verify it
> ```

---

**Q45.** Which SQL anomaly can occur at the `READ COMMITTED` isolation level?
- A) Dirty Read
- B) Non-Repeatable Read only
- C) Phantom Read and Non-Repeatable Read
- D) None — READ COMMITTED prevents all anomalies

✅ **Answer: C — Non-Repeatable Read AND Phantom Read**

> | Isolation Level | Dirty Read | Non-Repeatable Read | Phantom Read |
> |---|---|---|---|
> | READ UNCOMMITTED | ✅ possible | ✅ possible | ✅ possible |
> | **READ COMMITTED** | ❌ prevented | ✅ **possible** | ✅ **possible** |
> | REPEATABLE READ | ❌ prevented | ❌ prevented | ✅ possible |
> | SERIALIZABLE | ❌ prevented | ❌ prevented | ❌ prevented |
>
> - **Dirty Read**: reading another transaction's uncommitted (possibly rolled-back) data
> - **Non-Repeatable Read**: read same row twice → different values (another tx committed an update)
> - **Phantom Read**: read same range twice → different rows (another tx inserted/deleted)

---

**Q46.** Which Normal Form eliminates transitive dependencies?
- A) 1NF
- B) 2NF
- C) 3NF
- D) BCNF

✅ **Answer: C — Third Normal Form (3NF)**

> | NF | Eliminates |
> |---|---|
> | 1NF | Repeating groups, multiple values in one column |
> | 2NF | Partial dependencies (non-key depends on part of composite PK) |
> | **3NF** | Transitive dependencies (non-key column depends on another non-key column) |
>
> ```
> ❌ Violates 3NF:
> Orders(OrderId, CustomerId, CustomerCity, CustomerCountry)
> CustomerCity → CustomerCountry is a transitive dependency
> (CustomerCountry depends on CustomerCity, not on OrderId)
> 
> ✅ 3NF fix:
> Orders(OrderId, CustomerId)
> Customers(CustomerId, CustomerCity, CustomerCountry)
> ```

---

**Q47.** What is the difference between `TRUNCATE` and `DELETE`?
- A) `TRUNCATE` can have a `WHERE` clause; `DELETE` cannot
- B) `DELETE` fires triggers; `TRUNCATE` is minimally logged and does NOT fire triggers
- C) `TRUNCATE` cannot be rolled back; `DELETE` can
- D) They are identical

✅ **Answer: B**

> | Feature | DELETE | TRUNCATE |
> |---|---|---|
> | WHERE clause | ✅ Yes | ❌ No (removes all rows) |
> | Fires triggers | ✅ Yes | ❌ No |
> | Transaction log | Full (logs each row) | Minimal |
> | IDENTITY reset | ❌ No | ✅ Yes (resets counter to seed) |
> | Rollback | ✅ Yes | ✅ Yes (if in explicit transaction) |
> | Speed | Slower | Much faster |
>
> ```sql
> DELETE FROM Transactions WHERE UserId = @userId;  -- targeted, fires triggers, slower
> TRUNCATE TABLE TempImportStaging;                  -- wipes all, fast, no triggers
> ```

---

**Q48.** What does `OWASP A01: Broken Access Control` mean?
- A) Using weak passwords
- B) Users can access resources or perform actions they shouldn't be allowed to
- C) SQL injection vulnerabilities
- D) Outdated libraries

✅ **Answer: B**

> Broken Access Control is the #1 web vulnerability. A user can view or manipulate data belonging to other users by changing IDs in the request.
> ```csharp
> // ❌ VULNERABLE — IDOR (Insecure Direct Object Reference):
> [HttpGet("{id}")]
> public async Task<IActionResult> GetAccount(Guid id)
> {
>     var account = await db.Accounts.FindAsync(id);
>     return Ok(account); // no ownership check! User can access any account
> }
> // Attack: GET /api/accounts/other-users-guid → gets their data!
> 
> // ✅ FIXED — always filter by current user:
> [HttpGet("{id}")]
> public async Task<IActionResult> GetAccount(Guid id)
> {
>     var account = await db.Accounts
>         .Where(a => a.Id == id && a.UserId == CurrentUserId) // ownership enforced
>         .FirstOrDefaultAsync();
>     
>     if (account is null) return NotFound(); // don't reveal it exists
>     return Ok(account);
> }
> ```

---

**Q49.** Which HTTP method is idempotent but NOT safe?
- A) GET
- B) POST
- C) PUT
- D) HEAD

✅ **Answer: C — PUT**

> | Method | Safe (no side effects) | Idempotent (same result N times) |
> |---|---|---|
> | GET | ✅ | ✅ |
> | HEAD | ✅ | ✅ |
> | POST | ❌ | ❌ (creates new resource each call) |
> | **PUT** | **❌** | **✅** (replaces resource — calling 10x = same as 1x) |
> | DELETE | ❌ | ✅ (deleting already-deleted = same state) |
> | PATCH | ❌ | ❌ (depends on implementation) |
>
> PUT is idempotent: `PUT /accounts/123 {"balance":5000}` called 10 times = same result as once. But it's NOT safe because it modifies server state.

---

**Q50.** What is a covering index?
- A) An index that covers all tables in a schema
- B) A non-clustered index that includes all columns needed by a query, avoiding a key lookup
- C) An index that compresses all stored data
- D) A clustered index that spans multiple tables

✅ **Answer: B**

> A covering index contains all columns a query needs, so SQL Server satisfies the query entirely from the index without reading the actual data row ("key lookup").
> ```sql
> -- Query: find transactions by UserId, return Amount and Date
> SELECT Amount, TransactionDate FROM Transactions WHERE UserId = @userId
> 
> -- ❌ Non-covering — needs a KEY LOOKUP for Amount and TransactionDate:
> CREATE INDEX IX_Transactions_UserId ON Transactions (UserId)
> 
> -- ✅ Covering — all needed columns in the index:
> CREATE INDEX IX_Transactions_UserId_Covering
>     ON Transactions (UserId)
>     INCLUDE (Amount, TransactionDate)
> -- SQL Server can answer the entire query from the index alone — no table access
> ```

---

## Quick Score Check

| Score | Level |
|---|---|
| 45–50 | Senior — ready for the assessment |
| 38–44 | Strong Mid/Senior |
| 28–37 | Mid-level |
| 18–27 | Junior/Mid |
| < 18 | More review needed |
