# Mock Exam 1 — Multiple Choice (50 Questions)

> ⏱️ **Time limit: 40 minutes** | Simulate exam conditions — no notes, no IDE  
> Answers and explanations at the bottom.

---

## C# Language (Questions 1–15)

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

**Q2.** Which of the following correctly declares a generic method with a constraint that T must be a reference type with a parameterless constructor?
- A) `public T Create<T>() where T : new()`
- B) `public T Create<T>() where T : class`
- C) `public T Create<T>() where T : class, new()`
- D) `public T Create<T>() where T : struct, new()`

**Q3.** What is the difference between `Task.WhenAll` and `Task.WhenAny`?
- A) `WhenAll` runs tasks sequentially; `WhenAny` runs in parallel
- B) `WhenAll` completes when all tasks complete; `WhenAny` completes when the first task completes
- C) `WhenAll` completes when the first task completes; `WhenAny` completes when all complete
- D) There is no difference

**Q4.** Which LINQ method forces immediate execution?
- A) `Where()`
- B) `Select()`
- C) `OrderBy()`
- D) `ToList()`

**Q5.** What is a `record` type in C#?
- A) A class that supports inheritance and reference equality
- B) A reference type with value-based equality and immutability support
- C) A value type stored on the stack
- D) An alias for `struct`

**Q6.** What does `ConfigureAwait(false)` do?
- A) Prevents the async method from being awaited
- B) Configures the method to not capture the current synchronization context
- C) Runs the method synchronously
- D) Disables cancellation support

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

**Q8.** What is the correct way to raise an event safely?
- A) `MyEvent.Invoke(this, EventArgs.Empty)`
- B) `MyEvent?.Invoke(this, EventArgs.Empty)`
- C) `if (MyEvent != null) MyEvent(this, EventArgs.Empty)`
- D) Both B and C are correct (though B is preferred)

**Q9.** What is `SelectMany` used for?
- A) Selecting multiple properties from an object
- B) Flattening a collection of collections into a single collection
- C) Selecting the top N items from a collection
- D) Performing multiple projections in one operation

**Q10.** Which statement about `async void` is true?
- A) It's equivalent to `async Task`
- B) Exceptions from `async void` cannot be caught by the caller and will crash the app
- C) It should be used for all async methods that don't return a value
- D) It is the preferred pattern for event handlers that call async code

**Q11.** What is covariance in generics?
```csharp
IEnumerable<string> strings = new List<string>();
IEnumerable<object> objects = strings; // Is this valid?
```
- A) No — this always fails
- B) Yes — because `IEnumerable<T>` is covariant with `out T`
- C) Yes — because `List<T>` is covariant
- D) No — covariance only works with arrays

**Q12.** What does the `??=` operator do?
- A) Compares two nullable values
- B) Assigns the right operand to the left only if the left is `null`
- C) Returns the first non-null value between two operands
- D) Converts null to the default value of the type

**Q13.** What is the difference between `==` and `Equals()` for a `record` type?
- A) `==` compares references; `Equals()` compares values
- B) Both compare values — `record` overrides both to be value-based
- C) `==` compares values; `Equals()` compares references
- D) There is no difference for any type

**Q14.** Which statement is true about `IDisposable`?
- A) It automatically garbage-collects the object when out of scope
- B) Implementing `IDisposable` guarantees the finalizer will be called
- C) `using` statement calls `Dispose()` at the end of the block
- D) `Dispose()` is called automatically by the GC

**Q15.** What is the purpose of the `init` keyword?
```csharp
public string Name { get; init; }
```
- A) The property can only be set in the class constructor
- B) The property can only be set during object initialization (constructor or initializer), and never after
- C) The property is read-only
- D) The property is initialized with its default value

---

## .NET Core & ASP.NET Core (Questions 16–25)

**Q16.** Which DI lifetime creates a new instance every time it is requested?
- A) Singleton
- B) Scoped
- C) Transient
- D) Per-Request

**Q17.** What is the "captive dependency" problem?
- A) A Singleton service depends on another Singleton
- B) A Transient service depends on a Scoped service
- C) A Singleton service captures a Scoped service, causing incorrect shared state
- D) A Scoped service depends on a Transient service

**Q18.** In which order must middleware be registered in `Program.cs`?
- A) `UseAuthentication` → `UseAuthorization` → `UseCors`
- B) `UseCors` → `UseRouting` → `UseAuthentication` → `UseAuthorization`
- C) `UseAuthorization` → `UseAuthentication` → `UseCors`
- D) Order doesn't matter

**Q19.** What does `[ApiController]` attribute add over `[Controller]`?
- A) Nothing — they are the same
- B) Automatic 400 responses from model validation, binding source inference, Problem Details support
- C) Automatic database access
- D) Swagger documentation

**Q20.** What is the difference between `IOptions<T>`, `IOptionsSnapshot<T>`, and `IOptionsMonitor<T>`?
- A) They are interchangeable
- B) `IOptions<T>` is Singleton; `IOptionsSnapshot<T>` is Scoped; `IOptionsMonitor<T>` is Singleton with change notification
- C) All three are Scoped
- D) `IOptions<T>` is the newest; the others are deprecated

**Q21.** What does `app.Run()` do in the middleware pipeline?
- A) Starts the web application
- B) Adds a terminal middleware that does NOT call the next middleware
- C) Runs all middleware in parallel
- D) Adds middleware that always calls the next middleware

**Q22.** What is `IServiceScopeFactory` used for?
- A) Registering scoped services
- B) Creating new DI scopes from a Singleton service to safely resolve Scoped services
- C) Listing all registered services
- D) Configuring service lifetimes

**Q23.** Which is the correct way to read a nested config value `"JwtSettings:SecretKey"`?
- A) `config.GetSection("JwtSettings:SecretKey")`
- B) `config["JwtSettings:SecretKey"]`
- C) `config.GetValue("JwtSettings").GetValue("SecretKey")`
- D) `config.Get<JwtSettings>().SecretKey`

**Q24.** What HTTP status code does ASP.NET Core return when model validation fails (with `[ApiController]`)?
- A) 500
- B) 422
- C) 400
- D) 404

**Q25.** Which is the correct way to create a background service that loops?
- A) Implement `IHostedService.StartAsync` with an infinite loop
- B) Extend `BackgroundService` and override `ExecuteAsync` with a `while (!stoppingToken.IsCancellationRequested)` loop
- C) Use `Task.Run` in `Startup.Configure`
- D) Use `Thread.Sleep` in a Singleton service constructor

---

## Entity Framework Core (Questions 26–33)

**Q26.** What is the N+1 problem in EF Core?
- A) A query that returns N+1 columns
- B) Loading N parent entities then issuing 1 query per entity to load related data
- C) Having N+1 migrations applied
- D) An index with N+1 columns

**Q27.** What does `AsNoTracking()` do?
- A) Prevents the entity from being saved to the database
- B) Disables change tracking — faster for read-only queries
- C) Makes the query execute asynchronously
- D) Prevents lazy loading

**Q28.** What is the difference between `Find()` and `FirstOrDefault()`?
- A) `Find()` is async; `FirstOrDefault()` is sync
- B) `Find()` checks the local change tracker before querying the DB; `FirstOrDefault()` always queries the DB
- C) `Find()` throws if not found; `FirstOrDefault()` returns null
- D) There is no difference

**Q29.** Which EF Core feature loads related entities in the same query?
- A) Lazy Loading
- B) Explicit Loading
- C) Eager Loading (`Include()`)
- D) Deferred Loading

**Q30.** When should you use `AsSplitQuery()`?
- A) When you want to run queries on a separate thread
- B) When including multiple collection navigation properties to avoid Cartesian explosion
- C) When you need to split a transaction across multiple databases
- D) When ordering by multiple columns

**Q31.** What happens if you call `SaveChanges()` and one operation fails?
- A) Only the successful operations are saved
- B) EF Core automatically retries
- C) All operations are rolled back (they're in a single transaction)
- D) The exception is swallowed

**Q32.** What is a migration in EF Core?
- A) Moving data from one table to another
- B) A code representation of incremental database schema changes, with Up() and Down() methods
- C) A tool for exporting data to CSV
- D) A type of index

**Q33.** Which method executes a bulk DELETE without loading entities?
- A) `RemoveRange()` followed by `SaveChanges()`
- B) `ExecuteDeleteAsync()` (EF Core 7+)
- C) `DeleteMany()`
- D) `BulkDelete()`

---

## Design Patterns & Architecture (Questions 34–42)

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

**Q35.** What is the purpose of the Repository pattern?
- A) Store data in multiple databases
- B) Abstract data access behind an interface, decoupling business logic from persistence details
- C) Cache frequently-accessed data
- D) Implement the Command pattern

**Q36.** What is CQRS?
- A) A security pattern for validating commands
- B) A framework for building microservices
- C) Separating read (Query) operations from write (Command) operations into different models
- D) A type of database transaction

**Q37.** Which Design Pattern does ASP.NET Core's middleware pipeline use?
- A) Observer
- B) Decorator
- C) Chain of Responsibility
- D) Strategy

**Q38.** What does the Decorator pattern do?
- A) Creates a new instance of an object by cloning
- B) Adds new behavior to an object by wrapping it, without modifying the original class
- C) Provides a simplified interface to a complex subsystem
- D) Ensures only one instance of a class exists

**Q39.** In Clean Architecture, which layer should NOT depend on any other layer?
- A) Application
- B) Infrastructure
- C) Presentation
- D) Domain

**Q40.** What is a MediatR Pipeline Behavior?
- A) A way to route HTTP requests to controllers
- B) Cross-cutting concern middleware for IRequest handlers (validation, logging, caching)
- C) A message queue implementation
- D) A caching strategy

**Q41.** Which SOLID principle does dependency injection enforce?
- A) Single Responsibility
- B) Open/Closed
- C) Dependency Inversion
- D) Interface Segregation

**Q42.** What is the difference between the Factory Method and Abstract Factory patterns?
- A) Factory Method is for creating single objects; Abstract Factory creates families of related objects
- B) Abstract Factory is a simplified version of Factory Method
- C) They are the same pattern
- D) Factory Method uses composition; Abstract Factory uses inheritance

---

## Security, REST & Databases (Questions 43–50)

**Q43.** What is the correct HTTP status code when a user is authenticated but lacks permission?
- A) 401 Unauthorized
- B) 403 Forbidden
- C) 404 Not Found
- D) 400 Bad Request

**Q44.** A JWT token consists of:
- A) Header, Claims, Signature
- B) Header, Payload, Signature
- C) Username, Password, Expiry
- D) Issuer, Audience, Subject

**Q45.** Which SQL anomaly can occur at the `READ COMMITTED` isolation level?
- A) Dirty Read
- B) Non-Repeatable Read
- C) Phantom Read and Non-Repeatable Read
- D) None — READ COMMITTED prevents all anomalies

**Q46.** Which Normal Form eliminates transitive dependencies?
- A) 1NF
- B) 2NF
- C) 3NF
- D) BCNF

**Q47.** What is the difference between `TRUNCATE` and `DELETE`?
- A) `TRUNCATE` can have a `WHERE` clause; `DELETE` cannot
- B) `DELETE` fires triggers; `TRUNCATE` is minimally logged and does NOT fire triggers
- C) `TRUNCATE` can be rolled back; `DELETE` cannot
- D) They are identical

**Q48.** What does `OWASP A01: Broken Access Control` mean?
- A) Using weak passwords
- B) Users can access resources or perform actions they shouldn't be allowed to
- C) SQL injection vulnerabilities
- D) Outdated libraries

**Q49.** Which HTTP method is idempotent but NOT safe?
- A) GET
- B) POST
- C) PUT
- D) HEAD

**Q50.** What is a covering index?
- A) An index that covers all tables in a schema
- B) A non-clustered index that includes all columns needed by a query, avoiding a key lookup
- C) An index that compresses all stored data
- D) A clustered index that spans multiple tables

---

# ✅ Answer Key with Explanations

| Q | Answer | Q | Answer | Q | Answer | Q | Answer | Q | Answer |
|---|--------|---|--------|---|--------|---|--------|---|--------|
| 1 | B | 11 | B | 21 | B | 31 | C | 41 | C |
| 2 | C | 12 | B | 22 | B | 32 | B | 42 | A |
| 3 | B | 13 | B | 23 | B | 33 | B | 43 | B |
| 4 | D | 14 | C | 24 | C | 34 | B | 44 | B |
| 5 | B | 15 | B | 25 | B | 35 | B | 45 | C |
| 6 | B | 16 | C | 26 | B | 36 | C | 46 | C |
| 7 | B | 17 | C | 27 | B | 37 | C | 47 | B |
| 8 | D | 18 | B | 28 | B | 38 | B | 48 | B |
| 9 | B | 19 | B | 29 | C | 39 | D | 49 | C |
| 10 | B | 20 | B | 30 | B | 40 | B | 50 | B |

---

## Detailed Explanations

**Q1 (B): `6 5`** — `a++` is post-increment: returns current value (5) to `b`, then increments `a` to 6.

**Q7 (B): `3`** — LINQ `Where()` is deferred. The query is evaluated when `Count()` is called, after `4` was added. But wait — `Where(n => n > 1)` matches 2, 3, 4 = 3 items. ✅

**Q10 (B)** — `async void` swallows exceptions. They propagate to the `SynchronizationContext` (or crash the app). Event handlers are the only acceptable use.

**Q13 (B)** — `record` types override `==`, `!=`, `Equals()`, and `GetHashCode()` to use value-based equality automatically.

**Q17 (C)** — Captive dependency: a Singleton that holds a Scoped service will reuse the same Scoped instance for the app's lifetime, defeating the purpose of Scoped.

**Q18 (B)** — `UseCors` must be before auth. `UseAuthentication` must be before `UseAuthorization`. Wrong order = broken auth or CORS.

**Q30 (B)** — `AsSplitQuery()` prevents the Cartesian product when including multiple collections. Without it: 1000 transactions × 50 tags = 50,000 SQL rows.

**Q37 (C)** — ASP.NET Core middleware is a classic Chain of Responsibility: each `RequestDelegate` calls the next, or terminates the chain.

**Q45 (C)** — At READ COMMITTED: Dirty Reads are prevented, but Non-Repeatable Reads and Phantom Reads are still possible. `REPEATABLE READ` prevents Non-Repeatable Reads; `SERIALIZABLE` prevents all.

**Q49 (C): PUT** — PUT is idempotent (calling it 10 times = same result as calling once) but NOT safe (it modifies data). GET is both safe and idempotent. POST is neither.
