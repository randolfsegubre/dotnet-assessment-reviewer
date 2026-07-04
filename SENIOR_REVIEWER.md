
# 📘 Senior Full Stack .NET Developer — Complete Assessment Reviewer
### Coderbyte Assessment Preparation Guide
> **Author:** randolfsegubre | **Date:** July 2026  
> *This reviewer is self-contained. No external research needed.*

---

# TABLE OF CONTENTS

1. [C# Language Fundamentals](#1-c-language-fundamentals)
2. [ASP.NET Core & Web API](#2-aspnet-core--web-api)
3. [Entity Framework Core & Data Access](#3-entity-framework-core--data-access)
4. [SQL & Database Design](#4-sql--database-design)
5. [Frontend Integration (React & Angular)](#5-frontend-integration-react--angular)
6. [System Design & Architecture](#6-system-design--architecture)
7. [SOLID Principles & Design Patterns](#7-solid-principles--design-patterns)
8. [Security (JWT, OAuth2, HTTPS)](#8-security-jwt-oauth2-https)
9. [Performance, Caching & Scalability](#9-performance-caching--scalability)
10. [DevOps, CI/CD & Testing](#10-devops-cicd--testing)
11. [Algorithms & Data Structures (Coding Challenges)](#11-algorithms--data-structures-coding-challenges)
12. [MOCK EXAM 1 — Multiple Choice (50 Questions)](#12-mock-exam-1--multiple-choice-50-questions)
13. [MOCK EXAM 2 — Coding Challenges (10 Problems)](#13-mock-exam-2--coding-challenges-10-problems)
14. [MOCK EXAM 3 — System Design Scenarios](#14-mock-exam-3--system-design-scenarios)
15. [Answer Keys & Detailed Explanations](#15-answer-keys--detailed-explanations)

---

# 1. C# LANGUAGE FUNDAMENTALS

## 1.1 Value Types vs Reference Types

| Feature | Value Type | Reference Type |
|---|---|---|
| Stored in | Stack | Heap |
| Examples | `int`, `bool`, `struct`, `enum` | `class`, `string`, `array`, `interface` |
| Default value | Zero/false/null-equivalent | `null` |
| Assignment | Copies the value | Copies the reference |

```csharp
// Value Type — independent copy
int a = 10;
int b = a;
b = 20;
Console.WriteLine(a); // 10 — unchanged

// Reference Type — shared reference
var list1 = new List<int> { 1, 2, 3 };
var list2 = list1;
list2.Add(4);
Console.WriteLine(list1.Count); // 4 — both point to same object
```

---

## 1.2 Class vs Struct vs Interface vs Abstract Class

| Feature | Class | Struct | Interface | Abstract Class |
|---|---|---|---|---|
| Can be instantiated | ✅ | ✅ | ❌ | ❌ |
| Supports inheritance | ✅ (single) | ❌ | ✅ (multiple) | ✅ (single) |
| Can have constructors | ✅ | ✅ | ❌ (default only in C# 11) | ✅ |
| Can have fields | ✅ | ✅ | ❌ | ✅ |
| Can have method bodies | ✅ | ✅ | ✅ (default in C# 8+) | ✅ |
| Stored on | Heap | Stack (usually) | N/A | Heap |

**When to use what:**
- **Class** → general-purpose objects with behavior and state
- **Struct** → small, immutable data containers (e.g., `Point`, `Color`)
- **Interface** → define a contract without implementation
- **Abstract Class** → share common base behavior while forcing child implementation

```csharp
public interface IAnimal {
    string Name { get; }
    void Speak();
}

public abstract class Animal : IAnimal {
    public string Name { get; set; }
    public abstract void Speak(); // must implement
    public void Sleep() => Console.WriteLine("Zzz"); // shared behavior
}

public class Dog : Animal {
    public override void Speak() => Console.WriteLine("Woof!");
}
```

---

## 1.3 Async/Await — Deep Dive

**Key Rules:**
- `async` marks a method as asynchronous
- `await` suspends execution without blocking the thread
- Always return `Task`, `Task<T>`, or `ValueTask<T>` from async methods
- Avoid `async void` except for event handlers

```csharp
// ✅ CORRECT — non-blocking async
public async Task<string> GetUserAsync(int id)
{
    var user = await _dbContext.Users.FindAsync(id);
    return user?.Name ?? "Unknown";
}

// ❌ WRONG — blocks thread, causes deadlock risk
public string GetUser(int id)
{
    return GetUserAsync(id).Result; // NEVER do this in ASP.NET Core
}
```

**Common Pitfall — Deadlock:**
```csharp
// In synchronous context, .Result or .Wait() can deadlock
// Always use await all the way up the call stack
```

**ConfigureAwait(false) — use in library code:**
```csharp
var result = await someService.GetDataAsync().ConfigureAwait(false);
// Prevents capturing the synchronization context — better performance in libraries
```

---

## 1.4 Delegates, Events, and Lambda Expressions

```csharp
// Delegate — a type-safe function pointer
public delegate int MathOperation(int a, int b);

MathOperation add = (a, b) => a + b;
Console.WriteLine(add(3, 4)); // 7

// Func<> and Action<> — built-in delegate types
Func<int, int, int> multiply = (a, b) => a * b;
Action<string> print = msg => Console.WriteLine(msg);
Predicate<int> isEven = n => n % 2 == 0;

// Events
public class Button {
    public event EventHandler Clicked;
    public void Click() => Clicked?.Invoke(this, EventArgs.Empty);
}
var btn = new Button();
btn.Clicked += (sender, e) => Console.WriteLine("Button was clicked!");
btn.Click();
```

---

## 1.5 LINQ — Full Reference

```csharp
var numbers = new List<int> { 1, 2, 3, 4, 5, 6, 7, 8, 9, 10 };

// Filtering
var evens = numbers.Where(n => n % 2 == 0).ToList(); // [2,4,6,8,10]

// Projection
var squares = numbers.Select(n => n * n).ToList(); // [1,4,9,16...]

// Aggregation
int sum = numbers.Sum();
double avg = numbers.Average();
int max = numbers.Max();

// Ordering
var desc = numbers.OrderByDescending(n => n).ToList();

// Grouping
var grouped = numbers.GroupBy(n => n % 2 == 0 ? "even" : "odd");

// Chaining
var result = numbers
    .Where(n => n > 3)
    .Select(n => new { Value = n, Square = n * n })
    .OrderBy(x => x.Value)
    .ToList();

// Join
var customers = new[] { new { Id = 1, Name = "Alice" }, new { Id = 2, Name = "Bob" } };
var orders = new[] { new { CustomerId = 1, Amount = 100 }, new { CustomerId = 1, Amount = 200 } };

var joined = customers.Join(
    orders,
    c => c.Id,
    o => o.CustomerId,
    (c, o) => new { c.Name, o.Amount }
);
```

---

## 1.6 Generics

```csharp
// Generic method
public T Max<T>(T a, T b) where T : IComparable<T>
    => a.CompareTo(b) >= 0 ? a : b;

// Generic class
public class Repository<T> where T : class
{
    private readonly List<T> _items = new();
    public void Add(T item) => _items.Add(item);
    public IEnumerable<T> GetAll() => _items.AsReadOnly();
}

// Generic constraints
where T : class         // reference type
where T : struct        // value type
where T : new()         // has parameterless constructor
where T : IDisposable   // implements interface
where T : BaseClass     // inherits from a class
```

---

## 1.7 Nullable Types and Null Safety (C# 8+)

```csharp
// Nullable value types
int? age = null;
int definiteAge = age ?? 18; // null coalescing

// Nullable reference types (C# 8+)
string? maybeNull = null;
string definite = maybeNull ?? "default";

// Null-conditional operators
var length = maybeNull?.Length;         // null if maybeNull is null
var upper = maybeNull?.ToUpper() ?? ""; // safe chaining

// Pattern matching with null
if (maybeNull is not null)
    Console.WriteLine(maybeNull.Length);

// Null-forgiving operator (use sparingly)
string forcedNonNull = maybeNull!; // tells compiler "trust me, not null"
```

---

# 2. ASP.NET CORE & WEB API

## 2.1 Middleware Pipeline

The **request pipeline** in ASP.NET Core is a series of middleware components that process HTTP requests and responses. Order matters!

```
Request → [Middleware 1] → [Middleware 2] → [Controller] → [Middleware 2] → [Middleware 1] → Response
```

```csharp
// Program.cs — configuring the pipeline
var app = builder.Build();

app.UseExceptionHandler("/error");    // 1. Exception handling (outermost)
app.UseHsts();                         // 2. HTTPS Strict Transport
app.UseHttpsRedirection();             // 3. Redirect HTTP → HTTPS
app.UseStaticFiles();                  // 4. Serve static files
app.UseRouting();                      // 5. Route matching
app.UseAuthentication();               // 6. Who are you?
app.UseAuthorization();                // 7. What can you do?
app.UseRateLimiter();                  // 8. Rate limiting
app.MapControllers();                  // 9. Route to controllers
```

**Custom Middleware:**
```csharp
public class RequestLoggingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<RequestLoggingMiddleware> _logger;

    public RequestLoggingMiddleware(RequestDelegate next, ILogger<RequestLoggingMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        _logger.LogInformation($"Request: {context.Request.Method} {context.Request.Path}");
        await _next(context); // call next middleware
        _logger.LogInformation($"Response: {context.Response.StatusCode}");
    }
}

// Register it:
app.UseMiddleware<RequestLoggingMiddleware>();
```

---

## 2.2 Dependency Injection (DI) — Service Lifetimes

| Lifetime | Created | Destroyed | Use For |
|---|---|---|---|
| **Singleton** | Once, on first request | App shutdown | Stateless services, caches, logging |
| **Scoped** | Once per HTTP request | End of request | DbContext, repositories, unit-of-work |
| **Transient** | Every time it's requested | After use | Lightweight, stateless services |

```csharp
// Registration
builder.Services.AddSingleton<IEmailSender, SmtpEmailSender>();
builder.Services.AddScoped<IUserRepository, UserRepository>();
builder.Services.AddTransient<IPasswordHasher, BcryptPasswordHasher>();

// Constructor injection
public class UserService
{
    private readonly IUserRepository _repo;
    private readonly IEmailSender _emailSender;

    public UserService(IUserRepository repo, IEmailSender emailSender)
    {
        _repo = repo;
        _emailSender = emailSender;
    }
}
```

**⚠️ Captive Dependency Anti-Pattern:**  
Never inject a **Scoped** or **Transient** service into a **Singleton** — the short-lived service gets trapped inside the long-lived one, causing bugs.

---

## 2.3 Controller Design & Action Results

```csharp
[ApiController]
[Route("api/[controller]")]
[Authorize]
public class UsersController : ControllerBase
{
    private readonly IUserService _service;

    public UsersController(IUserService service) => _service = service;

    // GET api/users
    [HttpGet]
    [AllowAnonymous]
    public async Task<ActionResult<IEnumerable<UserDto>>> GetAll()
    {
        var users = await _service.GetAllAsync();
        return Ok(users); // 200 + JSON body
    }

    // GET api/users/5
    [HttpGet("{id:int}")]
    public async Task<ActionResult<UserDto>> GetById(int id)
    {
        var user = await _service.GetByIdAsync(id);
        return user is null ? NotFound() : Ok(user); // 404 or 200
    }

    // POST api/users
    [HttpPost]
    public async Task<ActionResult<UserDto>> Create([FromBody] CreateUserRequest request)
    {
        if (!ModelState.IsValid) return BadRequest(ModelState); // 400
        var created = await _service.CreateAsync(request);
        return CreatedAtAction(nameof(GetById), new { id = created.Id }, created); // 201
    }

    // PUT api/users/5
    [HttpPut("{id:int}")]
    public async Task<IActionResult> Update(int id, [FromBody] UpdateUserRequest request)
    {
        var success = await _service.UpdateAsync(id, request);
        return success ? NoContent() : NotFound(); // 204 or 404
    }

    // DELETE api/users/5
    [HttpDelete("{id:int}")]
    public async Task<IActionResult> Delete(int id)
    {
        var success = await _service.DeleteAsync(id);
        return success ? NoContent() : NotFound();
    }
}
```

---

## 2.4 Model Validation & Data Annotations

```csharp
public class CreateUserRequest
{
    [Required(ErrorMessage = "Name is required")]
    [StringLength(100, MinimumLength = 2)]
    public string Name { get; set; }

    [Required]
    [EmailAddress]
    public string Email { get; set; }

    [Required]
    [MinLength(8)]
    [RegularExpression(@"^(?=.*[A-Z])(?=.*\d).+$",
        ErrorMessage = "Password must have at least one uppercase and one number")]
    public string Password { get; set; }

    [Range(18, 120)]
    public int Age { get; set; }
}
```

---

## 2.5 Global Exception Handling

```csharp
// Using IExceptionHandler (ASP.NET Core 8+)
public class GlobalExceptionHandler : IExceptionHandler
{
    private readonly ILogger<GlobalExceptionHandler> _logger;

    public GlobalExceptionHandler(ILogger<GlobalExceptionHandler> logger)
        => _logger = logger;

    public async ValueTask<bool> TryHandleAsync(
        HttpContext context, Exception exception, CancellationToken ct)
    {
        _logger.LogError(exception, "Unhandled exception");

        var statusCode = exception switch
        {
            KeyNotFoundException => StatusCodes.Status404NotFound,
            UnauthorizedAccessException => StatusCodes.Status401Unauthorized,
            ArgumentException => StatusCodes.Status400BadRequest,
            _ => StatusCodes.Status500InternalServerError
        };

        context.Response.StatusCode = statusCode;
        await context.Response.WriteAsJsonAsync(new {
            error = exception.Message,
            statusCode
        }, ct);

        return true;
    }
}

// Register:
builder.Services.AddExceptionHandler<GlobalExceptionHandler>();
app.UseExceptionHandler();
```

---

# 3. ENTITY FRAMEWORK CORE & DATA ACCESS

## 3.1 Code-First Setup

```csharp
// Domain model
public class Product
{
    public int Id { get; set; }
    public string Name { get; set; }
    public decimal Price { get; set; }
    public int CategoryId { get; set; }
    public Category Category { get; set; }
}

public class Category
{
    public int Id { get; set; }
    public string Name { get; set; }
    public ICollection<Product> Products { get; set; }
}

// DbContext
public class AppDbContext : DbContext
{
    public AppDbContext(DbContextOptions<AppDbContext> options) : base(options) { }

    public DbSet<Product> Products { get; set; }
    public DbSet<Category> Categories { get; set; }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<Product>(entity =>
        {
            entity.HasKey(p => p.Id);
            entity.Property(p => p.Name).IsRequired().HasMaxLength(200);
            entity.Property(p => p.Price).HasPrecision(18, 2);
            entity.HasOne(p => p.Category)
                  .WithMany(c => c.Products)
                  .HasForeignKey(p => p.CategoryId);
        });
    }
}

// Registration
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("Default")));

// Migrations
// dotnet ef migrations add InitialCreate
// dotnet ef database update
```

---

## 3.2 IQueryable vs IEnumerable

| | `IQueryable<T>` | `IEnumerable<T>` |
|---|---|---|
| Execution | Deferred — runs SQL on server | In-memory — loads all data first |
| Location | Database | Application memory |
| Filtering | WHERE clause in SQL | LINQ in C# after load |
| Best for | Large datasets, complex queries | In-memory collections, post-load filtering |

```csharp
// IQueryable — SQL: SELECT * FROM Products WHERE Price > 100
var expensive = _context.Products
    .Where(p => p.Price > 100)   // translated to SQL
    .OrderBy(p => p.Name)
    .ToListAsync();               // executes here

// IEnumerable — loads ALL products, then filters in memory (bad for large tables!)
IEnumerable<Product> all = _context.Products.ToList(); // executes immediately
var expensive2 = all.Where(p => p.Price > 100);        // in-memory
```

---

## 3.3 Repository Pattern + Unit of Work

```csharp
// Generic repository interface
public interface IRepository<T> where T : class
{
    Task<T?> GetByIdAsync(int id);
    Task<IEnumerable<T>> GetAllAsync();
    Task AddAsync(T entity);
    void Update(T entity);
    void Delete(T entity);
}

// Generic implementation
public class Repository<T> : IRepository<T> where T : class
{
    protected readonly AppDbContext _context;
    protected readonly DbSet<T> _dbSet;

    public Repository(AppDbContext context)
    {
        _context = context;
        _dbSet = context.Set<T>();
    }

    public async Task<T?> GetByIdAsync(int id) => await _dbSet.FindAsync(id);
    public async Task<IEnumerable<T>> GetAllAsync() => await _dbSet.ToListAsync();
    public async Task AddAsync(T entity) => await _dbSet.AddAsync(entity);
    public void Update(T entity) => _dbSet.Update(entity);
    public void Delete(T entity) => _dbSet.Remove(entity);
}

// Unit of Work
public interface IUnitOfWork : IDisposable
{
    IRepository<Product> Products { get; }
    IRepository<Category> Categories { get; }
    Task<int> SaveChangesAsync();
}

public class UnitOfWork : IUnitOfWork
{
    private readonly AppDbContext _context;
    public IRepository<Product> Products { get; }
    public IRepository<Category> Categories { get; }

    public UnitOfWork(AppDbContext context)
    {
        _context = context;
        Products = new Repository<Product>(context);
        Categories = new Repository<Category>(context);
    }

    public Task<int> SaveChangesAsync() => _context.SaveChangesAsync();
    public void Dispose() => _context.Dispose();
}
```

---

## 3.4 Common EF Core Pitfalls

**N+1 Query Problem:**
```csharp
// ❌ BAD — N+1: 1 query for products + N queries for each category
var products = await _context.Products.ToListAsync();
foreach (var p in products)
    Console.WriteLine(p.Category.Name); // lazy load per item!

// ✅ GOOD — eager loading with Include
var products = await _context.Products
    .Include(p => p.Category)
    .ToListAsync(); // single JOIN query
```

**Tracking vs No-Tracking:**
```csharp
// Use AsNoTracking() for read-only queries — faster, less memory
var products = await _context.Products
    .AsNoTracking()
    .Where(p => p.Price > 50)
    .ToListAsync();
```

---

# 4. SQL & DATABASE DESIGN

## 4.1 Core SQL Clauses

```sql
-- Full query structure (order matters!)
SELECT columns
FROM table
JOIN other_table ON condition
WHERE row_filter
GROUP BY columns
HAVING group_filter
ORDER BY columns
OFFSET n ROWS FETCH NEXT m ROWS ONLY;  -- pagination (SQL Server)
```

## 4.2 Window Functions (Senior-Level)

```sql
-- ROW_NUMBER, RANK, DENSE_RANK
SELECT
    Name,
    Salary,
    ROW_NUMBER() OVER (ORDER BY Salary DESC) AS RowNum,
    RANK() OVER (ORDER BY Salary DESC) AS Rank,
    DENSE_RANK() OVER (ORDER BY Salary DESC) AS DenseRank
FROM Employees;

-- PARTITION BY — within groups
SELECT
    DepartmentId,
    Name,
    Salary,
    AVG(Salary) OVER (PARTITION BY DepartmentId) AS DeptAvgSalary
FROM Employees;

-- Running total
SELECT
    OrderDate,
    Amount,
    SUM(Amount) OVER (ORDER BY OrderDate ROWS UNBOUNDED PRECEDING) AS RunningTotal
FROM Orders;
```

## 4.3 CTEs (Common Table Expressions)

```sql
-- Simple CTE
WITH TopCustomers AS (
    SELECT CustomerId, SUM(Amount) AS Total
    FROM Orders
    GROUP BY CustomerId
    HAVING SUM(Amount) > 10000
)
SELECT c.Name, tc.Total
FROM Customers c
JOIN TopCustomers tc ON c.Id = tc.CustomerId;

-- Recursive CTE — for hierarchical data (org charts, trees)
WITH OrgChart AS (
    -- Anchor: top-level employees
    SELECT Id, Name, ManagerId, 0 AS Level
    FROM Employees
    WHERE ManagerId IS NULL

    UNION ALL

    -- Recursive: subordinates
    SELECT e.Id, e.Name, e.ManagerId, oc.Level + 1
    FROM Employees e
    JOIN OrgChart oc ON e.ManagerId = oc.Id
)
SELECT * FROM OrgChart ORDER BY Level;
```

## 4.4 Indexing Strategy

```sql
-- Clustered index (physical order of rows — one per table)
CREATE CLUSTERED INDEX IX_Orders_OrderDate ON Orders(OrderDate);

-- Non-clustered index (separate structure)
CREATE NONCLUSTERED INDEX IX_Orders_CustomerId ON Orders(CustomerId)
INCLUDE (OrderDate, Amount); -- covering index

-- When to index:
-- ✅ Columns used in WHERE, JOIN, ORDER BY
-- ✅ Foreign key columns
-- ❌ Columns with very low cardinality (e.g., boolean flags)
-- ❌ Tables with heavy INSERT/UPDATE (index maintenance cost)

-- Explain query plan
SET STATISTICS IO ON;
SELECT * FROM Orders WHERE CustomerId = 5;
```

## 4.5 Transactions & ACID

```sql
BEGIN TRANSACTION;
    UPDATE Accounts SET Balance = Balance - 500 WHERE Id = 1;
    UPDATE Accounts SET Balance = Balance + 500 WHERE Id = 2;

    IF @@ERROR <> 0
        ROLLBACK TRANSACTION;
    ELSE
        COMMIT TRANSACTION;
```

| Property | Meaning |
|---|---|
| **Atomicity** | All or nothing — partial updates don't persist |
| **Consistency** | Database moves from one valid state to another |
| **Isolation** | Concurrent transactions don't interfere |
| **Durability** | Committed data survives crashes |

---

# 5. FRONTEND INTEGRATION (REACT & ANGULAR)

## 5.1 React — Core Concepts for Full Stack

```jsx
// Hooks — the foundation of modern React
import React, { useState, useEffect, useCallback, useMemo, useRef } from 'react';

function UserList() {
    const [users, setUsers] = useState([]);
    const [loading, setLoading] = useState(true);
    const [error, setError] = useState(null);

    useEffect(() => {
        const controller = new AbortController();

        const fetchUsers = async () => {
            try {
                const res = await fetch('/api/users', { signal: controller.signal });
                if (!res.ok) throw new Error(`HTTP ${res.status}`);
                const data = await res.json();
                setUsers(data);
            } catch (err) {
                if (err.name !== 'AbortError') setError(err.message);
            } finally {
                setLoading(false);
            }
        };

        fetchUsers();
        return () => controller.abort(); // cleanup on unmount

    }, []); // empty deps = run once on mount

    if (loading) return <div>Loading...</div>;
    if (error) return <div>Error: {error}</div>;

    return (
        <ul>
            {users.map(user => (
                <li key={user.id}>{user.name} — {user.email}</li>
            ))}
        </ul>
    );
}
```

**`useMemo` vs `useCallback`:**
```jsx
// useMemo — memoizes a computed VALUE
const sortedList = useMemo(
    () => items.sort((a, b) => a.name.localeCompare(b.name)),
    [items] // recompute only when items changes
);

// useCallback — memoizes a FUNCTION reference
const handleDelete = useCallback((id) => {
    setItems(prev => prev.filter(item => item.id !== id));
}, []); // stable reference, prevents child re-renders
```

---

## 5.2 Angular — Core Concepts

```typescript
// Component
@Component({
    selector: 'app-product-list',
    templateUrl: './product-list.component.html',
    styleUrls: ['./product-list.component.scss']
})
export class ProductListComponent implements OnInit, OnDestroy {
    products: Product[] = [];
    private destroy$ = new Subject<void>();

    constructor(private productService: ProductService) {}

    ngOnInit(): void {
        this.productService.getAll()
            .pipe(takeUntil(this.destroy$)) // prevent memory leaks
            .subscribe({
                next: (data) => this.products = data,
                error: (err) => console.error(err)
            });
    }

    ngOnDestroy(): void {
        this.destroy$.next();
        this.destroy$.complete();
    }
}

// Service
@Injectable({ providedIn: 'root' })
export class ProductService {
    private apiUrl = '/api/products';

    constructor(private http: HttpClient) {}

    getAll(): Observable<Product[]> {
        return this.http.get<Product[]>(this.apiUrl).pipe(
            catchError(err => { console.error(err); return throwError(() => err); })
        );
    }

    create(product: Partial<Product>): Observable<Product> {
        return this.http.post<Product>(this.apiUrl, product);
    }
}
```

---

## 5.3 JWT Auth Flow (Frontend ↔ Backend)

```
1. User submits credentials (POST /api/auth/login)
2. Server validates → returns { accessToken, refreshToken }
3. Client stores accessToken (memory or sessionStorage — NOT localStorage for security)
4. Client sends token in every request: Authorization: Bearer <token>
5. Server validates token on each request via middleware
6. On 401 response → client uses refreshToken to get new accessToken
```

```csharp
// Interceptor in Angular
@Injectable()
export class AuthInterceptor implements HttpInterceptor {
    constructor(private authService: AuthService) {}

    intercept(req: HttpRequest<any>, next: HttpHandler): Observable<HttpEvent<any>> {
        const token = this.authService.getToken();
        const authReq = token
            ? req.clone({ headers: req.headers.set('Authorization', `Bearer ${token}`) })
            : req;
        return next.handle(authReq);
    }
}
```

---

# 6. SYSTEM DESIGN & ARCHITECTURE

## 6.1 Clean Architecture (Onion Architecture)

```
┌───────────────────────────────────┐
│         Presentation Layer        │ ← Controllers, Razor Pages, React/Angular
├───────────────────────────────────┤
│         Application Layer         │ ← Use Cases, Commands, Queries, DTOs
├───────────────────────────────────┤
│          Domain Layer             │ ← Entities, Value Objects, Domain Services
├───────────────────────────────────┤
│       Infrastructure Layer        │ ← EF Core, External APIs, Email, Storage
└───────────────────────────────────┘
```

**Dependency Rule:** Inner layers know NOTHING about outer layers. Dependencies point inward.

---

## 6.2 CQRS (Command Query Responsibility Segregation)

**Concept:** Separate read models (queries) from write models (commands).

```csharp
// Command — changes state
public record CreateProductCommand(string Name, decimal Price, int CategoryId);

// Command Handler
public class CreateProductCommandHandler : IRequestHandler<CreateProductCommand, int>
{
    private readonly AppDbContext _context;
    public CreateProductCommandHandler(AppDbContext context) => _context = context;

    public async Task<int> Handle(CreateProductCommand cmd, CancellationToken ct)
    {
        var product = new Product { Name = cmd.Name, Price = cmd.Price, CategoryId = cmd.CategoryId };
        _context.Products.Add(product);
        await _context.SaveChangesAsync(ct);
        return product.Id;
    }
}

// Query — reads state (using a separate read model/DTO)
public record GetProductByIdQuery(int Id);

public class GetProductByIdQueryHandler : IRequestHandler<GetProductByIdQuery, ProductDto?>
{
    private readonly AppDbContext _context;
    public GetProductByIdQueryHandler(AppDbContext context) => _context = context;

    public async Task<ProductDto?> Handle(GetProductByIdQuery query, CancellationToken ct)
        => await _context.Products
            .Where(p => p.Id == query.Id)
            .Select(p => new ProductDto { Id = p.Id, Name = p.Name, Price = p.Price })
            .FirstOrDefaultAsync(ct);
}
```

---

## 6.3 Microservices Design Patterns

| Pattern | Problem Solved | How |
|---|---|---|
| **API Gateway** | Single entry point, routing | Ocelot, Azure API Management, YARP |
| **Service Discovery** | Services find each other | Consul, Kubernetes DNS |
| **Circuit Breaker** | Prevent cascade failures | Polly library |
| **Saga** | Distributed transactions | Choreography (events) or Orchestration |
| **Outbox Pattern** | Reliable event publishing | Persist events to DB before publishing |
| **CQRS + Event Sourcing** | Audit trail, temporal queries | Store events, not current state |

```csharp
// Circuit Breaker with Polly
var policy = Policy
    .Handle<HttpRequestException>()
    .CircuitBreakerAsync(
        exceptionsAllowedBeforeBreaking: 3,
        durationOfBreak: TimeSpan.FromSeconds(30),
        onBreak: (ex, ts) => Console.WriteLine($"Circuit open for {ts}"),
        onReset: () => Console.WriteLine("Circuit closed")
    );

await policy.ExecuteAsync(() => httpClient.GetAsync("/api/resource"));
```

---

# 7. SOLID PRINCIPLES & DESIGN PATTERNS

## 7.1 SOLID Principles with C# Examples

### S — Single Responsibility Principle
```csharp
// ❌ WRONG — does too many things
public class UserManager {
    public void CreateUser(User u) { /* DB logic */ }
    public void SendWelcomeEmail(User u) { /* email logic */ }
    public void LogActivity(string msg) { /* logging logic */ }
}

// ✅ CORRECT — separate responsibilities
public class UserRepository { public void Save(User u) { } }
public class EmailService { public void SendWelcome(User u) { } }
public class ActivityLogger { public void Log(string msg) { } }
```

### O — Open/Closed Principle
```csharp
// Open for extension, closed for modification
public abstract class DiscountCalculator {
    public abstract decimal Calculate(decimal price);
}
public class SeniorDiscount : DiscountCalculator {
    public override decimal Calculate(decimal price) => price * 0.8m;
}
public class StudentDiscount : DiscountCalculator {
    public override decimal Calculate(decimal price) => price * 0.9m;
}
// Adding a new discount = add new class, don't modify existing
```

### L — Liskov Substitution Principle
```csharp
// Subtypes must be substitutable for their base types
public class Rectangle { public virtual int Width { get; set; } public virtual int Height { get; set; } }

// ❌ VIOLATES LSP — Square breaks Rectangle behavior
public class Square : Rectangle {
    public override int Width { set { base.Width = base.Height = value; } }
    public override int Height { set { base.Width = base.Height = value; } }
}
// Correct fix: Don't inherit — use a common interface IShape instead
```

### I — Interface Segregation Principle
```csharp
// ❌ Fat interface — forces implementors to implement unused methods
public interface IWorker { void Work(); void Eat(); void Sleep(); }

// ✅ Segregated
public interface IWorkable { void Work(); }
public interface IFeedable { void Eat(); }
public class RobotWorker : IWorkable { public void Work() { } } // robots don't eat
```

### D — Dependency Inversion Principle
```csharp
// ❌ High-level depends on low-level
public class OrderService { private SqlOrderRepository _repo = new(); }

// ✅ Both depend on abstraction
public class OrderService {
    private readonly IOrderRepository _repo;
    public OrderService(IOrderRepository repo) => _repo = repo;
}
```

---

## 7.2 Design Patterns Quick Reference

### Singleton (thread-safe with lazy initialization)
```csharp
public sealed class AppSettings
{
    private static readonly Lazy<AppSettings> _instance =
        new(() => new AppSettings());

    private AppSettings() { }
    public static AppSettings Instance => _instance.Value;
    public string ConnectionString { get; set; }
}
```

### Factory Method
```csharp
public interface INotification { void Send(string message); }
public class EmailNotification : INotification { public void Send(string m) => Console.WriteLine($"Email: {m}"); }
public class SmsNotification : INotification { public void Send(string m) => Console.WriteLine($"SMS: {m}"); }

public static class NotificationFactory
{
    public static INotification Create(string type) => type switch
    {
        "email" => new EmailNotification(),
        "sms" => new SmsNotification(),
        _ => throw new ArgumentException($"Unknown type: {type}")
    };
}
```

### Observer Pattern (built on C# events)
```csharp
public class OrderCreatedEvent { public int OrderId { get; init; } }

public interface IEventHandler<TEvent> { Task HandleAsync(TEvent e); }

public class SendOrderConfirmationHandler : IEventHandler<OrderCreatedEvent>
{
    public Task HandleAsync(OrderCreatedEvent e)
    {
        Console.WriteLine($"Sending confirmation for order {e.OrderId}");
        return Task.CompletedTask;
    }
}
```

---

# 8. SECURITY (JWT, OAuth2, HTTPS)

## 8.1 JWT Authentication Setup

```csharp
// appsettings.json
{
  "Jwt": {
    "Key": "YourSuperSecretKeyWith32+Characters",
    "Issuer": "https://yourapp.com",
    "Audience": "https://yourapp.com",
    "ExpiryMinutes": 60
  }
}

// Token generation
public string GenerateToken(User user)
{
    var key = new SymmetricSecurityKey(
        Encoding.UTF8.GetBytes(_config["Jwt:Key"]));

    var claims = new[]
    {
        new Claim(ClaimTypes.NameIdentifier, user.Id.ToString()),
        new Claim(ClaimTypes.Name, user.Name),
        new Claim(ClaimTypes.Email, user.Email),
        new Claim(ClaimTypes.Role, user.Role)
    };

    var token = new JwtSecurityToken(
        issuer: _config["Jwt:Issuer"],
        audience: _config["Jwt:Audience"],
        claims: claims,
        expires: DateTime.UtcNow.AddMinutes(60),
        signingCredentials: new SigningCredentials(key, SecurityAlgorithms.HmacSha256)
    );

    return new JwtSecurityTokenHandler().WriteToken(token);
}

// Registration in Program.cs
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options => {
        options.TokenValidationParameters = new TokenValidationParameters {
            ValidateIssuer = true,
            ValidateAudience = true,
            ValidateLifetime = true,
            ValidateIssuerSigningKey = true,
            ValidIssuer = builder.Configuration["Jwt:Issuer"],
            ValidAudience = builder.Configuration["Jwt:Audience"],
            IssuerSigningKey = new SymmetricSecurityKey(
                Encoding.UTF8.GetBytes(builder.Configuration["Jwt:Key"]))
        };
    });
```

## 8.2 CORS Configuration

```csharp
builder.Services.AddCors(options =>
{
    options.AddPolicy("AllowFrontend", policy =>
        policy.WithOrigins("https://myapp.com", "http://localhost:3000")
              .AllowAnyHeader()
              .AllowAnyMethod()
              .AllowCredentials());
});

app.UseCors("AllowFrontend");
```

## 8.3 Common Vulnerabilities & Mitigations

| Attack | Description | Mitigation |
|---|---|---|
| **SQL Injection** | Malicious SQL in input | Use parameterized queries / EF Core (never string concat) |
| **XSS** | Inject scripts into pages | Encode output, use CSP headers, sanitize input |
| **CSRF** | Forged requests from other sites | Anti-forgery tokens, SameSite cookies |
| **IDOR** | Access other users' resources by changing IDs | Always authorize resource ownership, not just authentication |
| **Broken JWT** | Weak key, `none` algorithm | Strong key, validate all parameters, short expiry |

---

# 9. PERFORMANCE, CACHING & SCALABILITY

## 9.1 Caching Strategies

```csharp
// In-Memory Cache
builder.Services.AddMemoryCache();

public class ProductService
{
    private readonly IMemoryCache _cache;
    private readonly AppDbContext _db;

    public async Task<IEnumerable<Product>> GetFeaturedAsync()
    {
        return await _cache.GetOrCreateAsync("featured-products", async entry =>
        {
            entry.AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(15);
            entry.SlidingExpiration = TimeSpan.FromMinutes(5);
            return await _db.Products.Where(p => p.IsFeatured).ToListAsync();
        });
    }
}

// Distributed Cache (Redis)
builder.Services.AddStackExchangeRedisCache(options =>
    options.Configuration = "localhost:6379");

public async Task<string?> GetCachedAsync(string key)
{
    var cached = await _distributedCache.GetStringAsync(key);
    if (cached is not null) return cached;

    var data = await FetchFromDatabase(key);
    await _distributedCache.SetStringAsync(key, data,
        new DistributedCacheEntryOptions {
            AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(30)
        });
    return data;
}
```

**Cache Invalidation Strategies:**
- **TTL (Time-to-Live)** — expire automatically after duration
- **Write-through** — update cache immediately when data changes
- **Cache-aside** — application manages cache explicitly
- **Event-driven invalidation** — invalidate via message bus events

---

## 9.2 Response Compression & Pagination

```csharp
// Pagination — always paginate large datasets!
public async Task<PagedResult<ProductDto>> GetPagedAsync(int page, int pageSize)
{
    var query = _context.Products.AsNoTracking();
    var total = await query.CountAsync();
    var items = await query
        .OrderBy(p => p.Name)
        .Skip((page - 1) * pageSize)
        .Take(pageSize)
        .Select(p => new ProductDto { Id = p.Id, Name = p.Name })
        .ToListAsync();

    return new PagedResult<ProductDto>(items, total, page, pageSize);
}

public record PagedResult<T>(IEnumerable<T> Items, int Total, int Page, int PageSize)
{
    public int TotalPages => (int)Math.Ceiling((double)Total / PageSize);
    public bool HasNextPage => Page < TotalPages;
    public bool HasPreviousPage => Page > 1;
}
```

---

# 10. DEVOPS, CI/CD & TESTING

## 10.1 Unit Testing with xUnit

```csharp
public class UserServiceTests
{
    private readonly Mock<IUserRepository> _mockRepo;
    private readonly UserService _service;

    public UserServiceTests()
    {
        _mockRepo = new Mock<IUserRepository>();
        _service = new UserService(_mockRepo.Object);
    }

    [Fact]
    public async Task GetByIdAsync_WhenUserExists_ReturnsUserDto()
    {
        // Arrange
        var user = new User { Id = 1, Name = "Alice", Email = "alice@test.com" };
        _mockRepo.Setup(r => r.GetByIdAsync(1)).ReturnsAsync(user);

        // Act
        var result = await _service.GetByIdAsync(1);

        // Assert
        Assert.NotNull(result);
        Assert.Equal("Alice", result.Name);
        _mockRepo.Verify(r => r.GetByIdAsync(1), Times.Once);
    }

    [Theory]
    [InlineData(-1)]
    [InlineData(0)]
    public async Task GetByIdAsync_WithInvalidId_ThrowsArgumentException(int id)
    {
        await Assert.ThrowsAsync<ArgumentException>(() => _service.GetByIdAsync(id));
    }
}
```

## 10.2 Integration Testing

```csharp
public class ProductsApiTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly HttpClient _client;

    public ProductsApiTests(WebApplicationFactory<Program> factory)
    {
        _client = factory.WithWebHostBuilder(builder => {
            builder.ConfigureServices(services => {
                // Replace real DB with in-memory
                services.RemoveAll<DbContextOptions<AppDbContext>>();
                services.AddDbContext<AppDbContext>(opt =>
                    opt.UseInMemoryDatabase("TestDb"));
            });
        }).CreateClient();
    }

    [Fact]
    public async Task GET_api_products_ReturnsOk()
    {
        var response = await _client.GetAsync("/api/products");
        response.EnsureSuccessStatusCode();
        var content = await response.Content.ReadFromJsonAsync<List<ProductDto>>();
        Assert.NotNull(content);
    }
}
```

---

# 11. ALGORITHMS & DATA STRUCTURES (CODING CHALLENGES)

## 11.1 Arrays & Strings

```csharp
// Two Sum — O(n) with hash map
public int[] TwoSum(int[] nums, int target)
{
    var map = new Dictionary<int, int>();
    for (int i = 0; i < nums.Length; i++)
    {
        int complement = target - nums[i];
        if (map.TryGetValue(complement, out int j))
            return new[] { j, i };
        map[nums[i]] = i;
    }
    return Array.Empty<int>();
}

// Valid Parentheses — O(n) with stack
public bool IsValid(string s)
{
    var stack = new Stack<char>();
    foreach (char c in s)
    {
        if (c == '(' || c == '[' || c == '{') stack.Push(c);
        else {
            if (stack.Count == 0) return false;
            char top = stack.Pop();
            if (c == ')' && top != '(') return false;
            if (c == ']' && top != '[') return false;
            if (c == '}' && top != '{') return false;
        }
    }
    return stack.Count == 0;
}

// Reverse a string in-place — O(n)
public string ReverseString(string s)
{
    var chars = s.ToCharArray();
    int l = 0, r = chars.Length - 1;
    while (l < r) { (chars[l], chars[r]) = (chars[r], chars[l]); l++; r--; }
    return new string(chars);
}
```

## 11.2 Linked Lists

```csharp
public class ListNode { public int Val; public ListNode Next; }

// Reverse a linked list — O(n)
public ListNode ReverseList(ListNode head)
{
    ListNode prev = null, curr = head;
    while (curr != null)
    {
        var next = curr.Next;
        curr.Next = prev;
        prev = curr;
        curr = next;
    }
    return prev;
}

// Detect cycle — Floyd's algorithm O(n)
public bool HasCycle(ListNode head)
{
    var slow = head; var fast = head;
    while (fast?.Next != null)
    {
        slow = slow.Next;
        fast = fast.Next.Next;
        if (slow == fast) return true;
    }
    return false;
}
```

## 11.3 Trees & Graphs

```csharp
public class TreeNode { public int Val; public TreeNode Left, Right; }

// Binary tree inorder traversal
public IList<int> InorderTraversal(TreeNode root)
{
    var result = new List<int>();
    void Traverse(TreeNode node) {
        if (node == null) return;
        Traverse(node.Left);
        result.Add(node.Val);
        Traverse(node.Right);
    }
    Traverse(root);
    return result;
}

// BFS on a graph
public int[] BFS(Dictionary<int, List<int>> graph, int start)
{
    var visited = new HashSet<int>();
    var queue = new Queue<int>();
    var order = new List<int>();
    queue.Enqueue(start); visited.Add(start);
    while (queue.Count > 0)
    {
        int node = queue.Dequeue();
        order.Add(node);
        foreach (var neighbor in graph[node])
            if (visited.Add(neighbor)) queue.Enqueue(neighbor);
    }
    return order.ToArray();
}
```

---

# 12. MOCK EXAM 1 — MULTIPLE CHOICE (50 QUESTIONS)

> **Instructions:** Choose the BEST answer. Answers are at the end.

---

**1.** Which lifetime should you use for `DbContext` in ASP.NET Core?
- A) Singleton  
- B) Transient  
- C) Scoped ✅  
- D) Static  

**2.** What is the output?
```csharp
int a = 5; int b = a; b = 10; Console.WriteLine(a);
```
- A) 10  
- B) 0  
- C) 5 ✅  
- D) Compilation error  

**3.** Which HTTP status code means "resource created successfully"?
- A) 200  
- B) 201 ✅  
- C) 204  
- D) 301  

**4.** `IQueryable` vs `IEnumerable`: which executes the query on the server?
- A) IEnumerable  
- B) IQueryable ✅  
- C) Both  
- D) Neither  

**5.** Which SOLID principle states a class should have only one reason to change?
- A) Open/Closed  
- B) Dependency Inversion  
- C) Single Responsibility ✅  
- D) Interface Segregation  

**6.** What keyword prevents thread-blocking in async methods?
- A) lock  
- B) await ✅  
- C) yield  
- D) async  

**7.** In JWT, where should you store the access token in a browser for best security?
- A) localStorage  
- B) Cookie with HttpOnly flag  
- C) In-memory (JS variable) ✅ (or HttpOnly cookie — both acceptable)  
- D) URL query string  

**8.** What EF Core method prevents change tracking for read-only queries?
- A) .ReadOnly()  
- B) .NoTracking()  
- C) .AsNoTracking() ✅  
- D) .Detach()  

**9.** Which pattern separates read and write operations into distinct models?
- A) Repository  
- B) Unit of Work  
- C) CQRS ✅  
- D) Saga  

**10.** What is the time complexity of looking up a value in a `Dictionary<K,V>`?
- A) O(n)  
- B) O(log n)  
- C) O(1) ✅  
- D) O(n²)  

**11.** Which middleware must come BEFORE `UseAuthorization` in the pipeline?
- A) UseRouting  
- B) UseAuthentication ✅  
- C) UseStaticFiles  
- D) UseHttpsRedirection  

**12.** What is a "captive dependency" in DI?
- A) A service injected via property  
- B) A longer-lived service holding a shorter-lived one ✅  
- C) A circular dependency  
- D) A singleton depending on another singleton  

**13.** Which C# type cannot be inherited?
- A) abstract class  
- B) partial class  
- C) sealed class ✅  
- D) generic class  

**14.** What SQL clause filters AFTER aggregation?
- A) WHERE  
- B) HAVING ✅  
- C) FILTER  
- D) GROUP BY  

**15.** Which Polly policy prevents cascade failures in microservices?
- A) Retry  
- B) Timeout  
- C) Circuit Breaker ✅  
- D) Bulkhead  

**16.** In React, which hook runs after every render by default?
- A) useState  
- B) useEffect ✅ (with no dependency array)  
- C) useCallback  
- D) useRef  

**17.** Which HTTP method is idempotent but NOT safe?
- A) GET  
- B) POST  
- C) PUT ✅  
- D) PATCH  

**18.** In EF Core, what causes the N+1 query problem?
- A) Using AsNoTracking  
- B) Lazy loading navigation properties in a loop ✅  
- C) Using Include()  
- D) Calling SaveChangesAsync multiple times  

**19.** Which design pattern defines a family of algorithms and makes them interchangeable?
- A) Template Method  
- B) Strategy ✅  
- C) State  
- D) Observer  

**20.** What does `ConfigureAwait(false)` do?
- A) Cancels the operation  
- B) Prevents resuming on the original synchronization context ✅  
- C) Sets a timeout  
- D) Runs on a background thread  

**21.** Which collection preserves insertion order AND allows duplicate values?
- A) HashSet<T>  
- B) Dictionary<K,V>  
- C) List<T> ✅  
- D) SortedSet<T>  

**22.** REST API best practice: what should a DELETE response return when successful?
- A) 200 with deleted object  
- B) 202 Accepted  
- C) 204 No Content ✅  
- D) 200 with null  

**23.** Which Angular decorator marks a class as injectable?
- A) @Component  
- B) @NgModule  
- C) @Injectable ✅  
- D) @Pipe  

**24.** In Clean Architecture, which layer contains business rules?
- A) Infrastructure  
- B) Presentation  
- C) Domain ✅  
- D) Application  

**25.** What is the primary advantage of a `struct` over a `class` in C#?
- A) Supports inheritance  
- B) Allocated on stack — better performance for small data ✅  
- C) Can implement interfaces  
- D) Reference semantics  

**26.** Which SQL function assigns sequential numbers within groups?
- A) RANK()  
- B) COUNT()  
- C) ROW_NUMBER() ✅  
- D) NTILE()  

**27.** What does CORS stand for?
- A) Cross-Origin Resource Sharing ✅  
- B) Cross-Origin Request Security  
- C) Client-Origin Request Service  
- D) Core Object Resource System  

**28.** In JWT, which part contains the user claims?
- A) Header  
- B) Payload ✅  
- C) Signature  
- D) Footer  

**29.** Which LINQ method immediately executes the query?
- A) .Where()  
- B) .Select()  
- C) .OrderBy()  
- D) .ToList() ✅  

**30.** What problem does the Repository pattern solve?
- A) Improves query performance  
- B) Abstracts data access logic from business logic ✅  
- C) Manages database migrations  
- D) Handles authentication  

**31.** `async void` should be avoided because:
- A) It's slower  
- B) Exceptions cannot be caught with try/catch by callers ✅  
- C) It doesn't compile  
- D) It blocks the thread  

**32.** What does the `[ApiController]` attribute add in ASP.NET Core?
- A) Enables Swagger  
- B) Automatic model validation and binding source inference ✅  
- C) JWT validation  
- D) Response caching  

**33.** Which algorithm efficiently finds two numbers summing to a target?
- A) Bubble sort then binary search  
- B) Nested loops O(n²)  
- C) Hash map O(n) ✅  
- D) Merge sort  

**34.** What is "eventual consistency" in distributed systems?
- A) All nodes always have identical data  
- B) Data will become consistent across nodes over time ✅  
- C) Changes are immediately committed everywhere  
- D) A locking mechanism  

**35.** Which pattern is used to handle distributed transactions across microservices?
- A) Repository  
- B) Saga ✅  
- C) CQRS  
- D) Gateway  

**36.** In React, what prevents a child component from re-rendering when parent re-renders?
- A) useEffect  
- B) React.memo ✅  
- C) useContext  
- D) useRef  

**37.** Which HTTP status code represents "Unauthorized" (not authenticated)?
- A) 400  
- B) 403  
- C) 401 ✅  
- D) 404  

**38.** What is a covering index in SQL?
- A) An index that spans multiple tables  
- B) An index that includes all columns needed for a query ✅  
- C) A clustered index  
- D) An index on a foreign key  

**39.** Which .NET feature enables compiling code at runtime?
- A) Reflection  
- B) Roslyn / Compiler APIs ✅  
- C) Expression Trees  
- D) Delegates  

**40.** What does `yield return` do in C#?
- A) Throws an exception  
- B) Terminates iteration  
- C) Produces a value lazily in an iterator ✅  
- D) Suspends the current thread  

**41.** In Angular, which operator prevents memory leaks from subscriptions?
- A) mergeMap  
- B) switchMap  
- C) takeUntil ✅  
- D) combineLatest  

**42.** What HTTP verb should be used for partial updates?
- A) PUT  
- B) POST  
- C) PATCH ✅  
- D) UPDATE  

**43.** Which cache strategy reads from cache first, then database on miss?
- A) Write-through  
- B) Write-behind  
- C) Cache-aside ✅  
- D) Read-through  

**44.** What is the purpose of a `refresh token`?
- A) Encrypt the access token  
- B) Allow obtaining a new access token without re-login ✅  
- C) Validate user identity  
- D) Extend the current token's lifetime  

**45.** Binary search requires the array to be:
- A) Filled with unique values  
- B) Sorted ✅  
- C) Of even length  
- D) Of fixed type  

**46.** Which principle says "depend on abstractions, not concretions"?
- A) Open/Closed  
- B) Liskov Substitution  
- C) Single Responsibility  
- D) Dependency Inversion ✅  

**47.** What does `AddScoped` mean in ASP.NET Core DI?
- A) One instance for the application lifetime  
- B) New instance every time it's requested  
- C) One instance per HTTP request ✅  
- D) Instance shared across threads  

**48.** In EF Core, what does `Include()` do?
- A) Adds a new entity  
- B) Eager-loads a related navigation property ✅  
- C) Includes raw SQL  
- D) Marks entity as modified  

**49.** A recursive CTE in SQL is typically used for:
- A) Window functions  
- B) Hierarchical or tree-structured data ✅  
- C) Aggregations  
- D) Cross joins  

**50.** What's the best way to avoid SQL injection in C#?
- A) Escape special characters manually  
- B) Use stored procedures only  
- C) Use parameterized queries or ORM ✅  
- D) Validate input length only  

---

# 13. MOCK EXAM 2 — CODING CHALLENGES (10 PROBLEMS)

---

### Problem 1: FizzBuzz (Warmup)
Write a method that returns "Fizz" for multiples of 3, "Buzz" for 5, "FizzBuzz" for both, else the number.

**Solution:**
```csharp
public string FizzBuzz(int n)
{
    if (n % 15 == 0) return "FizzBuzz";
    if (n % 3 == 0) return "Fizz";
    if (n % 5 == 0) return "Buzz";
    return n.ToString();
}
```

---

### Problem 2: Find Maximum Subarray Sum (Kadane's Algorithm)
Given an integer array, find the contiguous subarray with the largest sum.

**Example:** `[-2, 1, -3, 4, -1, 2, 1, -5, 4]` → `6` (subarray `[4,-1,2,1]`)

**Solution:**
```csharp
public int MaxSubArray(int[] nums)
{
    int maxSum = nums[0];
    int currentSum = nums[0];

    for (int i = 1; i < nums.Length; i++)
    {
        currentSum = Math.Max(nums[i], currentSum + nums[i]);
        maxSum = Math.Max(maxSum, currentSum);
    }
    return maxSum;
}
// Time: O(n), Space: O(1)
```

---

### Problem 3: Check if a String is a Valid Palindrome
Ignore non-alphanumeric characters and case.

**Example:** `"A man, a plan, a canal: Panama"` → `true`

**Solution:**
```csharp
public bool IsPalindrome(string s)
{
    int l = 0, r = s.Length - 1;
    while (l < r)
    {
        while (l < r && !char.IsLetterOrDigit(s[l])) l++;
        while (l < r && !char.IsLetterOrDigit(s[r])) r--;
        if (char.ToLower(s[l]) != char.ToLower(s[r])) return false;
        l++; r--;
    }
    return true;
}
```

---

### Problem 4: Group Anagrams
Given an array of strings, group anagrams together.

**Example:** `["eat","tea","tan","ate","nat","bat"]` → `[["eat","tea","ate"],["tan","nat"],["bat"]]`

**Solution:**
```csharp
public IList<IList<string>> GroupAnagrams(string[] strs)
{
    var map = new Dictionary<string, List<string>>();
    foreach (var s in strs)
    {
        var key = string.Concat(s.OrderBy(c => c)); // sort chars as key
        if (!map.ContainsKey(key)) map[key] = new List<string>();
        map[key].Add(s);
    }
    return map.Values.Cast<IList<string>>().ToList();
}
// Time: O(n * k log k) where k = max string length
```

---

### Problem 5: LRU Cache
Implement a Least Recently Used cache with `Get(key)` and `Put(key, value)` in O(1).

**Solution:**
```csharp
public class LRUCache
{
    private readonly int _capacity;
    private readonly Dictionary<int, LinkedListNode<(int key, int value)>> _map;
    private readonly LinkedList<(int key, int value)> _list;

    public LRUCache(int capacity)
    {
        _capacity = capacity;
        _map = new Dictionary<int, LinkedListNode<(int, int)>>(capacity);
        _list = new LinkedList<(int, int)>();
    }

    public int Get(int key)
    {
        if (!_map.TryGetValue(key, out var node)) return -1;
        _list.Remove(node);
        _list.AddFirst(node);
        return node.Value.value;
    }

    public void Put(int key, int value)
    {
        if (_map.TryGetValue(key, out var existing))
        {
            _list.Remove(existing);
            _map.Remove(key);
        }
        else if (_map.Count >= _capacity)
        {
            var lru = _list.Last;
            _map.Remove(lru.Value.key);
            _list.RemoveLast();
        }
        var newNode = _list.AddFirst((key, value));
        _map[key] = newNode;
    }
}
```

---

### Problem 6: SQL — Find Employees Earning More Than Their Manager
Given: `Employee(Id, Name, Salary, ManagerId)` — ManagerId references Id.

**Solution:**
```sql
SELECT e.Name AS Employee
FROM Employee e
JOIN Employee m ON e.ManagerId = m.Id
WHERE e.Salary > m.Salary;
```

---

### Problem 7: Build a Thread-Safe Singleton in C#

**Solution:**
```csharp
public sealed class ConfigManager
{
    // Lazy<T> is thread-safe by default
    private static readonly Lazy<ConfigManager> _instance =
        new Lazy<ConfigManager>(() => new ConfigManager());

    private ConfigManager()
    {
        // Load config here
    }

    public static ConfigManager Instance => _instance.Value;

    public string GetSetting(string key) => /* load from config */ key;
}
```

---

### Problem 8: Implement a Generic Stack

**Solution:**
```csharp
public class GenericStack<T>
{
    private readonly List<T> _items = new();

    public void Push(T item) => _items.Add(item);

    public T Pop()
    {
        if (_items.Count == 0) throw new InvalidOperationException("Stack is empty");
        var item = _items[^1];
        _items.RemoveAt(_items.Count - 1);
        return item;
    }

    public T Peek()
    {
        if (_items.Count == 0) throw new InvalidOperationException("Stack is empty");
        return _items[^1];
    }

    public bool IsEmpty => _items.Count == 0;
    public int Count => _items.Count;
}
```

---

### Problem 9: React — Debounced Search Component

Build a search input that fetches results only after the user stops typing for 300ms.

**Solution:**
```jsx
import { useState, useEffect } from 'react';

function useDebounce(value, delay) {
    const [debouncedValue, setDebouncedValue] = useState(value);
    useEffect(() => {
        const timer = setTimeout(() => setDebouncedValue(value), delay);
        return () => clearTimeout(timer);
    }, [value, delay]);
    return debouncedValue;
}

function SearchComponent() {
    const [query, setQuery] = useState('');
    const [results, setResults] = useState([]);
    const debouncedQuery = useDebounce(query, 300);

    useEffect(() => {
        if (!debouncedQuery) { setResults([]); return; }
        fetch(`/api/search?q=${encodeURIComponent(debouncedQuery)}`)
            .then(res => res.json())
            .then(setResults);
    }, [debouncedQuery]);

    return (
        <div>
            <input
                value={query}
                onChange={e => setQuery(e.target.value)}
                placeholder="Search..."
            />
            <ul>{results.map(r => <li key={r.id}>{r.name}</li>)}</ul>
        </div>
    );
}
```

---

### Problem 10: Design a RESTful API for a Blog System

**Requirements:** Users can create posts, add comments, like posts.

**Solution — Endpoints:**
```
POST   /api/auth/register          → Register user
POST   /api/auth/login             → Returns JWT

GET    /api/posts                  → List posts (paginated)
POST   /api/posts                  → Create post [Authorize]
GET    /api/posts/{id}             → Get post + comments
PUT    /api/posts/{id}             → Update post [Authorize, owner only]
DELETE /api/posts/{id}             → Delete post [Authorize, owner only]

POST   /api/posts/{id}/comments    → Add comment [Authorize]
DELETE /api/comments/{id}          → Delete comment [Authorize]

POST   /api/posts/{id}/like        → Like a post [Authorize]
DELETE /api/posts/{id}/like        → Unlike a post [Authorize]
```

**Controller sample:**
```csharp
[HttpPost("{id}/like")]
[Authorize]
public async Task<IActionResult> LikePost(int id)
{
    var userId = int.Parse(User.FindFirstValue(ClaimTypes.NameIdentifier)!);
    var alreadyLiked = await _likeService.HasLikedAsync(userId, id);
    if (alreadyLiked) return Conflict("Already liked");

    await _likeService.LikeAsync(userId, id);
    return Ok();
}
```

---

# 14. MOCK EXAM 3 — SYSTEM DESIGN SCENARIOS

---

### Scenario 1: Design a URL Shortener

**Requirements:** 1000 shortening requests/sec, 10,000 redirects/sec, 5-year retention.

**Solution:**
```
API:
  POST /shorten  { longUrl } → { shortCode, shortUrl }
  GET  /{shortCode}          → 301 Redirect to longUrl

Components:
  ┌────────────┐    ┌──────────────┐    ┌─────────────┐
  │ API Gateway│───▶│ URL Service  │───▶│  SQL/NoSQL  │
  └────────────┘    └──────────────┘    └─────────────┘
                           │
                    ┌──────▼──────┐
                    │  Redis Cache│  ← Cache top 20% of URLs
                    └─────────────┘

Short code generation:
  - Base62 encode (a-z, A-Z, 0-9) a counter or hash
  - 7 characters = 62^7 = 3.5 trillion unique URLs

Database Schema:
  UrlMappings(Id, ShortCode[indexed], LongUrl, UserId, CreatedAt, ExpiresAt, ClickCount)

Scaling:
  - Horizontal scaling of API servers (stateless)
  - Read replicas for redirect lookups
  - Redis for hot URLs
  - Async click counting (write to queue, batch process)
```

---

### Scenario 2: Design a Real-Time Chat System (.NET + SignalR)

**Solution:**
```csharp
// Hub
public class ChatHub : Hub
{
    public async Task SendMessage(string roomId, string message)
    {
        var user = Context.User?.Identity?.Name;
        await Clients.Group(roomId).SendAsync("ReceiveMessage", new {
            user, message, timestamp = DateTime.UtcNow
        });

        // Persist to DB
        await _messageService.SaveAsync(roomId, user, message);
    }

    public async Task JoinRoom(string roomId)
    {
        await Groups.AddToGroupAsync(Context.ConnectionId, roomId);
        await Clients.Group(roomId).SendAsync("UserJoined", Context.User?.Identity?.Name);
    }

    public override async Task OnDisconnectedAsync(Exception? ex)
    {
        // Handle cleanup
        await base.OnDisconnectedAsync(ex);
    }
}

// Architecture:
// Multiple servers → Use Redis backplane for SignalR
builder.Services.AddSignalR()
    .AddStackExchangeRedis("localhost:6379");
```

---

### Scenario 3: Design an E-Commerce Order Processing System

**Architecture:**
```
┌──────────┐   ┌─────────────────────────────────────────────────────┐
│ Frontend │──▶│ API Gateway                                          │
└──────────┘   └───┬─────────────────────────────────────────────────┘
                   │
         ┌─────────┼───────────────┐
         ▼         ▼               ▼
   ┌──────────┐ ┌──────────┐ ┌──────────────┐
   │  Order   │ │ Inventory│ │  Payment     │
   │ Service  │ │ Service  │ │  Service     │
   └─────┬────┘ └──────────┘ └──────────────┘
         │
   ┌─────▼────────────┐
   │  Message Broker  │  (RabbitMQ / Azure Service Bus)
   └─────┬────────────┘
         │
   ┌─────▼─────────┐
   │  Notification │
   │  Service      │
   └───────────────┘

Saga Flow (Choreography):
1. OrderService creates order (status: Pending) → publishes OrderCreated
2. InventoryService reserves stock → publishes StockReserved
3. PaymentService charges card → publishes PaymentProcessed
4. OrderService updates status → Confirmed → publishes OrderConfirmed
5. NotificationService sends email/SMS

On failure at step 3:
→ InventoryService receives PaymentFailed → releases stock
→ OrderService marks order as Failed
```

---

# 15. ANSWER KEYS & DETAILED EXPLANATIONS

## Mock Exam 1 — Answer Key

| # | Answer | Key Concept |
|---|---|---|
| 1 | C — Scoped | DbContext is NOT thread-safe; one per request |
| 2 | C — 5 | Value type copied; `a` and `b` are independent |
| 3 | B — 201 | Created response includes Location header |
| 4 | B — IQueryable | Translated to SQL; runs on DB server |
| 5 | C — SRP | "One reason to change" = one responsibility |
| 6 | B — await | Releases thread without blocking |
| 7 | C — In-memory (JS) | localStorage vulnerable to XSS; HttpOnly cookie also valid |
| 8 | C — AsNoTracking | Skips change tracker overhead |
| 9 | C — CQRS | Command = write; Query = read |
| 10 | C — O(1) | Hash table average-case lookup |
| 11 | B — UseAuthentication | Auth before authorization, always |
| 12 | B — Longer-lived holding shorter | Scoped inside Singleton = bug |
| 13 | C — sealed class | Cannot be subclassed |
| 14 | B — HAVING | WHERE filters rows; HAVING filters groups |
| 15 | C — Circuit Breaker | Opens after N failures; prevents cascade |
| 16 | B — useEffect | No deps = runs after every render |
| 17 | C — PUT | PUT is idempotent; same result multiple calls |
| 18 | B — Lazy loading in loop | Each access = separate DB query |
| 19 | B — Strategy | Defines interchangeable algorithm family |
| 20 | B — Prevents context capture | Library code optimization |
| 21 | C — List<T> | Ordered, allows duplicates |
| 22 | C — 204 No Content | Standard delete success response |
| 23 | C — @Injectable | Marks for DI container |
| 24 | C — Domain | Business logic; no framework dependencies |
| 25 | B — Stack allocation | Better perf for small, short-lived data |
| 26 | C — ROW_NUMBER() | Sequential, no gaps, no ties handling |
| 27 | A — Cross-Origin Resource Sharing | Browser security mechanism |
| 28 | B — Payload | Base64-encoded claims JSON |
| 29 | D — .ToList() | Materializes/executes the query |
| 30 | B — Abstracts data access | Decouples business logic from DB |
| 31 | B — Exceptions uncatchable | Callers can't await void |
| 32 | B — Auto model validation | 400 on invalid model automatically |
| 33 | C — Hash map O(n) | Single pass with complement lookup |
| 34 | B — Consistent over time | Distributed systems trade-off |
| 35 | B — Saga | Choreography or orchestration |
| 36 | B — React.memo | Memoizes component; skips re-render |
| 37 | C — 401 | 401 = unauthenticated; 403 = unauthorized |
| 38 | B — All columns included | No table lookup needed |
| 39 | B — Roslyn | .NET's compiler-as-a-service platform |
| 40 | C — Lazy value in iterator | Produces sequence without loading all |
| 41 | C — takeUntil | Completes on destroy subject |
| 42 | C — PATCH | Partial update; PUT replaces entirely |
| 43 | C — Cache-aside | App reads cache, falls back to DB |
| 44 | B — Get new access token | Long-lived; stored securely |
| 45 | B — Sorted | Required for binary search to work |
| 46 | D — DIP | Depend on abstractions/interfaces |
| 47 | C — One per request | Perfect for DbContext and services |
| 48 | B — Eager loads navigation | Prevents N+1 with single JOIN |
| 49 | B — Hierarchical data | Org charts, menus, categories |
| 50 | C — Parameterized queries/ORM | EF Core handles this automatically |

---

## Quick-Reference Cheat Sheet

### HTTP Status Codes
| Code | Meaning |
|---|---|
| 200 | OK |
| 201 | Created |
| 204 | No Content |
| 400 | Bad Request |
| 401 | Unauthorized (not authenticated) |
| 403 | Forbidden (authenticated but not allowed) |
| 404 | Not Found |
| 409 | Conflict |
| 422 | Unprocessable Entity |
| 429 | Too Many Requests |
| 500 | Internal Server Error |

### Big-O Complexity Quick Reference
| Operation | Array | Linked List | Dictionary | Sorted Array |
|---|---|---|---|---|
| Access | O(1) | O(n) | O(1) | O(1) |
| Search | O(n) | O(n) | O(1) | O(log n) |
| Insert | O(n) | O(1) | O(1) | O(n) |
| Delete | O(n) | O(1) | O(1) | O(n) |

### DI Lifetime Summary
| Lifetime | Scope | Use Case |
|---|---|---|
| Singleton | App lifetime | Config, caches, logging, stateless |
| Scoped | Per HTTP request | DbContext, repositories |
| Transient | Per injection | Lightweight, stateless utilities |

### EF Core Quick Commands
```bash
dotnet ef migrations add <Name>
dotnet ef database update
dotnet ef migrations remove
dotnet ef dbcontext scaffold "ConnectionString" Microsoft.EntityFrameworkCore.SqlServer
```

---

*End of Reviewer — Good luck on your assessment, Randolf! 🎯*