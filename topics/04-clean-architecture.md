# 04 — Clean Architecture & SOLID Principles

> 🔗 **BudgetPH reference**: [Application/DependencyInjection.cs](https://github.com/randolfsegubre/BudgetPH/blob/master/src/Core/FinanceManager.Application/DependencyInjection.cs) | [Domain/Entities/](https://github.com/randolfsegubre/BudgetPH/blob/master/src/Core/FinanceManager.Domain/Entities/)

---

## Table of Contents
1. [SOLID Principles](#1-solid-principles)
2. [Clean Architecture Layers](#2-clean-architecture-layers)
3. [CQRS with MediatR](#3-cqrs-with-mediatr)
4. [Repository Pattern](#4-repository-pattern)
5. [Unit of Work Pattern](#5-unit-of-work-pattern)
6. [Domain Events](#6-domain-events)
7. [Value Objects](#7-value-objects)
8. [Common Interview Questions](#8-common-interview-questions)

---

## 1. SOLID Principles

### S — Single Responsibility Principle (SRP)
> A class should have **one reason to change** — one job.

```csharp
// ❌ VIOLATES SRP — does too many things
public class UserService
{
    public User GetUser(int id) { ... }         // business logic
    public void SendEmail(string to) { ... }    // email logic
    public void LogAction(string msg) { ... }   // logging logic
    public User ParseFromCsv(string csv) { ... } // parsing logic
}

// ✅ SRP — each class has one responsibility
public class UserService
{
    public User GetUser(int id) { ... }
}

public class EmailService
{
    public void SendEmail(string to, string subject, string body) { ... }
}

public class UserCsvParser
{
    public User ParseFromCsv(string csv) { ... }
}
```

### O — Open/Closed Principle (OCP)
> Classes should be **open for extension, closed for modification**.

```csharp
// ❌ VIOLATES OCP — must modify class to add new shapes
public class AreaCalculator
{
    public double Calculate(object shape)
    {
        if (shape is Circle c) return Math.PI * c.Radius * c.Radius;
        if (shape is Rectangle r) return r.Width * r.Height;
        // ← must add 'if' for every new shape = modifying the class
        return 0;
    }
}

// ✅ OCP — extend without modifying
public interface IShape
{
    double Area();
}

public class Circle : IShape
{
    public double Radius { get; set; }
    public double Area() => Math.PI * Radius * Radius;
}

public class Rectangle : IShape
{
    public double Width { get; set; }
    public double Height { get; set; }
    public double Area() => Width * Height;
}

// Triangle just needs to implement IShape — no changes to AreaCalculator!
public class Triangle : IShape
{
    public double Base { get; set; }
    public double Height { get; set; }
    public double Area() => 0.5 * Base * Height;
}

public class AreaCalculator
{
    public double Calculate(IShape shape) => shape.Area(); // unchanged!
}
```

### L — Liskov Substitution Principle (LSP)
> Subtypes must be **substitutable** for their base types without altering correctness.

```csharp
// ❌ VIOLATES LSP — Square breaks Rectangle's contract
public class Rectangle
{
    public virtual int Width { get; set; }
    public virtual int Height { get; set; }
    public int Area() => Width * Height;
}

public class Square : Rectangle
{
    public override int Width
    {
        set { base.Width = value; base.Height = value; } // forces height to match!
    }
}

// Code expecting Rectangle breaks when given Square:
Rectangle r = new Square();
r.Width = 5;
r.Height = 3;
Console.WriteLine(r.Area()); // Expected 15, gets 9 ← LSP violation!

// ✅ LSP — use a proper abstraction
public interface IShape
{
    int Area();
}
public class Rectangle : IShape { ... }
public class Square : IShape { ... }
```

### I — Interface Segregation Principle (ISP)
> Clients should not be forced to depend on **methods they don't use**.

```csharp
// ❌ VIOLATES ISP — fat interface forces empty implementations
public interface IWorker
{
    void Work();
    void Eat();
    void Sleep();
}

public class Robot : IWorker
{
    public void Work() { ... }
    public void Eat() { throw new NotImplementedException(); } // robots don't eat!
    public void Sleep() { throw new NotImplementedException(); }
}

// ✅ ISP — segregate by client need
public interface IWorkable { void Work(); }
public interface IEatable  { void Eat(); }
public interface ISleepable { void Sleep(); }

public class Human : IWorkable, IEatable, ISleepable
{
    public void Work() { ... }
    public void Eat() { ... }
    public void Sleep() { ... }
}

public class Robot : IWorkable
{
    public void Work() { ... }  // only implements what it needs
}
```

### D — Dependency Inversion Principle (DIP)
> High-level modules should **not depend on low-level modules**. Both should depend on **abstractions**.

```csharp
// ❌ VIOLATES DIP — high-level depends on concrete low-level
public class OrderService
{
    private SqlOrderRepository _repo = new SqlOrderRepository(); // tight coupling!
    
    public void PlaceOrder(Order order)
    {
        _repo.Save(order);
    }
}

// ✅ DIP — depend on abstraction
public interface IOrderRepository
{
    void Save(Order order);
}

public class OrderService
{
    private readonly IOrderRepository _repo; // depends on interface
    
    public OrderService(IOrderRepository repo) // injected via DI
    {
        _repo = repo;
    }
    
    public void PlaceOrder(Order order)
    {
        _repo.Save(order); // works with any IOrderRepository implementation
    }
}

// Now you can swap: SqlOrderRepository → MongoOrderRepository → FakeOrderRepository
```

---

## 2. Clean Architecture Layers

```
┌─────────────────────────────────────────────────────┐
│                    PRESENTATION                      │  ← API, Web, CLI
│            (FinanceManager.API, .Web)                │
├─────────────────────────────────────────────────────┤
│                  INFRASTRUCTURE                      │  ← DB, Email, Files
│           (FinanceManager.Infrastructure)            │
├─────────────────────────────────────────────────────┤
│                   APPLICATION                        │  ← Use Cases, DTOs
│            (FinanceManager.Application)              │
├─────────────────────────────────────────────────────┤
│                     DOMAIN                          │  ← Entities, Rules
│              (FinanceManager.Domain)                 │
└─────────────────────────────────────────────────────┘
       Dependency direction: inward only ↓
```

### Dependency Rule
**Inner layers must NOT depend on outer layers.**

| Layer | Contains | Depends On |
|---|---|---|
| **Domain** | Entities, Value Objects, Domain Events, Interfaces | Nothing |
| **Application** | Use Cases, Commands/Queries, DTOs, Validators | Domain only |
| **Infrastructure** | EF Core, Email, File Storage, External APIs | Application, Domain |
| **Presentation** | Controllers, Middleware, DTOs | Application only |

### 🔗 BudgetPH Layer Structure

```
Domain:          Account, Transaction, Budget, SavingsGoal entities + enums
Application:     TransactionsController logic, DTOs, MappingProfile, Validators
Infrastructure:  ApplicationDbContext (EF Core), AuthService (Identity)
Presentation:    Controllers, Program.cs, appsettings.json
```

### What Goes Where

```csharp
// ✅ DOMAIN — pure business rules, no framework dependencies
namespace FinanceManager.Domain.Entities;
public class Account : BaseEntity
{
    public string Name { get; private set; }
    public decimal Balance { get; private set; }
    
    // Domain method — enforces business rule
    public void Debit(decimal amount)
    {
        if (amount <= 0) throw new DomainException("Amount must be positive");
        if (Balance < amount) throw new DomainException("Insufficient funds");
        Balance -= amount;
    }
}

// ✅ APPLICATION — orchestrates domain, no infrastructure concerns
public class TransferFundsCommand
{
    public Guid FromAccountId { get; set; }
    public Guid ToAccountId { get; set; }
    public decimal Amount { get; set; }
}

public class TransferFundsHandler : IRequestHandler<TransferFundsCommand>
{
    private readonly IApplicationDbContext _context;
    
    public async Task Handle(TransferFundsCommand cmd, CancellationToken ct)
    {
        var from = await _context.Accounts.FindAsync(cmd.FromAccountId);
        var to = await _context.Accounts.FindAsync(cmd.ToAccountId);
        
        from.Debit(cmd.Amount);   // domain method
        to.Credit(cmd.Amount);    // domain method
        
        await _context.SaveChangesAsync(ct);
    }
}

// ✅ INFRASTRUCTURE — implements interfaces defined in Application/Domain
public class ApplicationDbContext : DbContext, IApplicationDbContext
{
    // EF Core — infrastructure concern, not in domain or application
}
```

---

## 3. CQRS with MediatR

**CQRS** = Command Query Responsibility Segregation — separate reading from writing.

- **Query** = reads data, no side effects, returns data
- **Command** = changes state, no return data (or just a result indicator)

```csharp
// Install: dotnet add package MediatR

// QUERY — get data
public class GetAccountByIdQuery : IRequest<AccountDto?>
{
    public Guid AccountId { get; set; }
    public Guid UserId { get; set; }
}

public class GetAccountByIdHandler : IRequestHandler<GetAccountByIdQuery, AccountDto?>
{
    private readonly ApplicationDbContext _context;
    private readonly IMapper _mapper;

    public GetAccountByIdHandler(ApplicationDbContext context, IMapper mapper)
    {
        _context = context; _mapper = mapper;
    }

    public async Task<AccountDto?> Handle(GetAccountByIdQuery query, CancellationToken ct)
    {
        var account = await _context.Accounts
            .AsNoTracking()
            .Include(a => a.CreditCardDetails)
            .FirstOrDefaultAsync(a => a.Id == query.AccountId && a.UserId == query.UserId, ct);
        
        return account is null ? null : _mapper.Map<AccountDto>(account);
    }
}

// COMMAND — change state
public class CreateAccountCommand : IRequest<Guid>
{
    public string Name { get; set; } = "";
    public AccountType AccountType { get; set; }
    public decimal InitialBalance { get; set; }
    public Guid UserId { get; set; }
}

public class CreateAccountHandler : IRequestHandler<CreateAccountCommand, Guid>
{
    private readonly ApplicationDbContext _context;

    public CreateAccountHandler(ApplicationDbContext context) => _context = context;

    public async Task<Guid> Handle(CreateAccountCommand cmd, CancellationToken ct)
    {
        var account = new Account
        {
            Id = Guid.NewGuid(),
            Name = cmd.Name,
            AccountType = cmd.AccountType,
            Balance = cmd.InitialBalance,
            UserId = cmd.UserId
        };

        _context.Accounts.Add(account);
        await _context.SaveChangesAsync(ct);
        return account.Id;
    }
}

// Controller — sends commands/queries via MediatR
[HttpGet("{id:guid}")]
public async Task<IActionResult> GetById(Guid id, CancellationToken ct)
{
    var query = new GetAccountByIdQuery { AccountId = id, UserId = GetCurrentUserId() };
    var result = await _mediator.Send(query, ct);
    return result is null ? NotFound() : Ok(result);
}

// Register MediatR
builder.Services.AddMediatR(cfg => 
    cfg.RegisterServicesFromAssembly(typeof(GetAccountByIdQuery).Assembly));
```

### MediatR Pipeline Behaviors (Cross-Cutting Concerns)

```csharp
// Validation behavior — runs FluentValidation before every command
public class ValidationBehavior<TRequest, TResponse> 
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : IRequest<TResponse>
{
    private readonly IEnumerable<IValidator<TRequest>> _validators;

    public ValidationBehavior(IEnumerable<IValidator<TRequest>> validators)
        => _validators = validators;

    public async Task<TResponse> Handle(
        TRequest request, 
        RequestHandlerDelegate<TResponse> next, 
        CancellationToken ct)
    {
        if (_validators.Any())
        {
            var context = new ValidationContext<TRequest>(request);
            var failures = _validators
                .Select(v => v.Validate(context))
                .SelectMany(r => r.Errors)
                .Where(f => f != null)
                .ToList();

            if (failures.Count != 0)
                throw new ValidationException(failures);
        }

        return await next(); // proceed to handler
    }
}

// Logging behavior
public class LoggingBehavior<TRequest, TResponse> 
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : IRequest<TResponse>
{
    private readonly ILogger<LoggingBehavior<TRequest, TResponse>> _logger;

    public async Task<TResponse> Handle(
        TRequest request, RequestHandlerDelegate<TResponse> next, CancellationToken ct)
    {
        var name = typeof(TRequest).Name;
        _logger.LogInformation("Handling {RequestName}", name);
        var response = await next();
        _logger.LogInformation("Handled {RequestName}", name);
        return response;
    }
}

// Register behaviors
services.AddTransient(typeof(IPipelineBehavior<,>), typeof(ValidationBehavior<,>));
services.AddTransient(typeof(IPipelineBehavior<,>), typeof(LoggingBehavior<,>));
```

---

## 4. Repository Pattern

The Repository pattern abstracts the data access layer, making business logic independent of database technology.

```csharp
// Generic repository interface (in Application/Domain layer)
public interface IRepository<T> where T : BaseEntity
{
    Task<T?> GetByIdAsync(Guid id, CancellationToken ct = default);
    Task<IReadOnlyList<T>> GetAllAsync(CancellationToken ct = default);
    Task AddAsync(T entity, CancellationToken ct = default);
    void Update(T entity);
    void Remove(T entity);
}

// Specific repository interface
public interface IAccountRepository : IRepository<Account>
{
    Task<IReadOnlyList<Account>> GetByUserIdAsync(Guid userId, CancellationToken ct = default);
    Task<Account?> GetWithDetailsAsync(Guid id, CancellationToken ct = default);
    Task<decimal> GetTotalBalanceAsync(Guid userId, CancellationToken ct = default);
}

// EF Core implementation (in Infrastructure layer)
public class AccountRepository : IAccountRepository
{
    private readonly ApplicationDbContext _context;

    public AccountRepository(ApplicationDbContext context) => _context = context;

    public async Task<Account?> GetByIdAsync(Guid id, CancellationToken ct)
        => await _context.Accounts.FindAsync(new object[] { id }, ct);

    public async Task<IReadOnlyList<Account>> GetByUserIdAsync(Guid userId, CancellationToken ct)
        => await _context.Accounts
            .AsNoTracking()
            .Where(a => a.UserId == userId && !a.IsDeleted)
            .OrderBy(a => a.Name)
            .ToListAsync(ct);

    public async Task<Account?> GetWithDetailsAsync(Guid id, CancellationToken ct)
        => await _context.Accounts
            .Include(a => a.CreditCardDetails)
            .Include(a => a.LoanDetails)
            .Include(a => a.InvestmentAccountDetails)
                .ThenInclude(i => i!.Holdings)
            .FirstOrDefaultAsync(a => a.Id == id, ct);

    public async Task AddAsync(Account entity, CancellationToken ct)
        => await _context.Accounts.AddAsync(entity, ct);

    public void Update(Account entity)
        => _context.Accounts.Update(entity);

    public void Remove(Account entity)
        => _context.Accounts.Remove(entity);

    public async Task<IReadOnlyList<Account>> GetAllAsync(CancellationToken ct)
        => await _context.Accounts.AsNoTracking().ToListAsync(ct);
}
```

---

## 5. Unit of Work Pattern

Coordinates multiple repositories to share the same database transaction:

```csharp
public interface IUnitOfWork : IAsyncDisposable
{
    IAccountRepository Accounts { get; }
    ITransactionRepository Transactions { get; }
    IBudgetRepository Budgets { get; }
    
    Task<int> SaveChangesAsync(CancellationToken ct = default);
    Task BeginTransactionAsync(CancellationToken ct = default);
    Task CommitTransactionAsync(CancellationToken ct = default);
    Task RollbackTransactionAsync(CancellationToken ct = default);
}

public class UnitOfWork : IUnitOfWork
{
    private readonly ApplicationDbContext _context;
    private IDbContextTransaction? _transaction;

    public IAccountRepository Accounts { get; }
    public ITransactionRepository Transactions { get; }
    public IBudgetRepository Budgets { get; }

    public UnitOfWork(
        ApplicationDbContext context,
        IAccountRepository accounts,
        ITransactionRepository transactions,
        IBudgetRepository budgets)
    {
        _context = context;
        Accounts = accounts;
        Transactions = transactions;
        Budgets = budgets;
    }

    public Task<int> SaveChangesAsync(CancellationToken ct)
        => _context.SaveChangesAsync(ct);

    public async Task BeginTransactionAsync(CancellationToken ct)
        => _transaction = await _context.Database.BeginTransactionAsync(ct);

    public async Task CommitTransactionAsync(CancellationToken ct)
    {
        await _context.SaveChangesAsync(ct);
        await _transaction!.CommitAsync(ct);
    }

    public async Task RollbackTransactionAsync(CancellationToken ct)
        => await _transaction?.RollbackAsync(ct)!;

    public async ValueTask DisposeAsync()
    {
        if (_transaction != null) await _transaction.DisposeAsync();
        await _context.DisposeAsync();
    }
}
```

---

## 6. Domain Events

Domain events notify other parts of the system when something significant happens in the domain:

```csharp
// Domain event interface (in Domain layer)
public interface IDomainEvent : INotification { } // INotification from MediatR

// Domain event
public class BudgetExceededEvent : IDomainEvent
{
    public Guid UserId { get; }
    public string BudgetName { get; }
    public decimal Limit { get; }
    public decimal Spent { get; }
    public BudgetExceededEvent(Guid userId, string name, decimal limit, decimal spent)
    {
        UserId = userId; BudgetName = name; Limit = limit; Spent = spent;
    }
}

// Base entity with domain events
public abstract class BaseEntity
{
    public Guid Id { get; protected set; } = Guid.NewGuid();
    
    private readonly List<IDomainEvent> _domainEvents = [];
    public IReadOnlyCollection<IDomainEvent> DomainEvents => _domainEvents.AsReadOnly();

    protected void AddDomainEvent(IDomainEvent domainEvent) => _domainEvents.Add(domainEvent);
    public void ClearDomainEvents() => _domainEvents.Clear();
}

// Domain entity raises event
public class BudgetItem : BaseEntity
{
    public decimal AllocatedAmount { get; private set; }
    public decimal SpentAmount { get; private set; }

    public void AddExpense(decimal amount)
    {
        SpentAmount += amount;
        if (SpentAmount > AllocatedAmount)
        {
            AddDomainEvent(new BudgetExceededEvent(
                BudgetOwnerId, CategoryName, AllocatedAmount, SpentAmount));
        }
    }
}

// Event handler — reacts to event (in Application layer)
public class BudgetExceededEventHandler : INotificationHandler<BudgetExceededEvent>
{
    private readonly INotificationService _notificationService;

    public async Task Handle(BudgetExceededEvent notification, CancellationToken ct)
    {
        await _notificationService.SendAsync(new UserNotification
        {
            UserId = notification.UserId,
            Title = "Budget Alert",
            Message = $"You've exceeded your {notification.BudgetName} budget!"
        });
    }
}

// Dispatch events after SaveChanges (in DbContext)
public override async Task<int> SaveChangesAsync(CancellationToken ct = default)
{
    var result = await base.SaveChangesAsync(ct);
    
    var entities = ChangeTracker.Entries<BaseEntity>()
        .Where(e => e.Entity.DomainEvents.Any())
        .Select(e => e.Entity);

    foreach (var entity in entities)
    {
        var events = entity.DomainEvents.ToList();
        entity.ClearDomainEvents();
        foreach (var domainEvent in events)
            await _mediator.Publish(domainEvent, ct);
    }
    
    return result;
}
```

---

## 7. Value Objects

Value objects have no identity — they are equal if their values are equal:

```csharp
// ✅ Using C# records (perfect for value objects)
public record Money(decimal Amount, string Currency)
{
    public Money Add(Money other)
    {
        if (Currency != other.Currency) 
            throw new InvalidOperationException("Cannot add different currencies");
        return this with { Amount = Amount + other.Amount };
    }
    
    public override string ToString() => $"{Currency} {Amount:N2}";
}

// Value object as class
public class EmailAddress
{
    public string Value { get; }
    
    public EmailAddress(string value)
    {
        if (string.IsNullOrWhiteSpace(value) || !value.Contains('@'))
            throw new ArgumentException("Invalid email address");
        Value = value.ToLowerInvariant();
    }
    
    // Equality based on value
    public override bool Equals(object? obj) 
        => obj is EmailAddress other && Value == other.Value;
    
    public override int GetHashCode() => Value.GetHashCode();
    
    public static implicit operator string(EmailAddress email) => email.Value;
    public static implicit operator EmailAddress(string value) => new(value);
}

// Usage
var email1 = new EmailAddress("User@EXAMPLE.COM");
var email2 = new EmailAddress("user@example.com");
Console.WriteLine(email1 == email2); // true — same value
```

---

## 8. Common Interview Questions

**Q1: What is the difference between Clean Architecture and Onion Architecture?**

> They are essentially the same concept with different names. Both place the domain/entities at the center with dependencies pointing inward. Clean Architecture (Robert Martin) defines 4 concentric circles: Entities → Use Cases → Interface Adapters → Frameworks. Onion Architecture (Jeffrey Palermo) has: Domain → Domain Services → Application Services → Infrastructure/UI. BudgetPH follows Clean Architecture.

**Q2: Explain CQRS and when you would use it.**

> CQRS separates read (Query) and write (Command) operations into separate models. Benefits: read models can be optimized independently (denormalized views, caching), write models can enforce strict business rules. Use CQRS for complex domains with different read/write patterns. For simple CRUD apps, it adds unnecessary complexity. In BudgetPH, the controller directly uses DbContext (simpler approach); CQRS via MediatR would be appropriate as it grows.

**Q3: What is the difference between a Repository and a DAO (Data Access Object)?**

> A **Repository** belongs to the domain layer — it speaks in domain terms and returns domain objects. A **DAO** is an infrastructure pattern that maps to tables directly. Repositories abstract away the persistence mechanism entirely; DAOs typically map 1:1 to database tables.

**Q4: Can you violate SOLID principles intentionally?**

> Yes — pragmatically. SOLID is a guideline, not a law. Over-applying SOLID can lead to over-engineered code. A simple script with a single `Main` method doesn't need ISP. The rule is: apply SOLID where complexity and change frequency justify it. Small applications benefit more from simplicity; large, long-lived systems benefit from SOLID adherence.

**Q5: How does MediatR's Pipeline Behavior relate to middleware?**

> Both are implementations of the **Chain of Responsibility** pattern. ASP.NET Core middleware runs for HTTP requests (transport layer). MediatR Pipeline Behaviors run for commands/queries (application layer). Behaviors are more focused: they only run for specific `IRequest<T>` types and can be conditionally applied using generic constraints.

**Q6: What is a Domain Service vs an Application Service?**

> A **Domain Service** contains business logic that doesn't naturally belong to a single entity (e.g., currency conversion that involves multiple exchange rates). A **Domain Service** knows nothing about persistence. An **Application Service** (Use Case) orchestrates domain objects and infrastructure — it calls repositories, domain services, and dispatches domain events.
