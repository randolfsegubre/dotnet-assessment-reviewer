# 05 — Design Patterns

> Classic Gang of Four (GoF) patterns with C# examples.  
> Categorized into Creational, Structural, and Behavioral.

---

## Table of Contents
1. [Creational Patterns](#1-creational-patterns)
2. [Structural Patterns](#2-structural-patterns)
3. [Behavioral Patterns](#3-behavioral-patterns)
4. [Common Interview Questions](#4-common-interview-questions)

---

## 1. Creational Patterns

### Singleton
Ensures a class has **only one instance** and provides a global access point.

```csharp
// Thread-safe Singleton using Lazy<T>
public class AppSettings
{
    private static readonly Lazy<AppSettings> _instance = 
        new(() => new AppSettings());

    public static AppSettings Instance => _instance.Value;
    
    public string ConnectionString { get; private set; } = "";

    private AppSettings() { } // private constructor
}

// Usage
var settings = AppSettings.Instance;

// ⚠️ DI containers already manage singleton lifetime — prefer:
builder.Services.AddSingleton<IAppSettings, AppSettings>();
// Over rolling your own Singleton
```

**When to use:** Configuration objects, logger factories, caches.  
**Warning:** Singleton + mutable state = thread safety issues. Make singletons immutable or use proper locking.

---

### Factory Method
Defines an interface for creating objects, but lets **subclasses decide** which class to instantiate.

```csharp
// Product
public interface IReportGenerator
{
    byte[] Generate(ReportData data);
}

// Concrete products
public class PdfReportGenerator : IReportGenerator
{
    public byte[] Generate(ReportData data) { /* QuestPDF */ return Array.Empty<byte>(); }
}

public class ExcelReportGenerator : IReportGenerator
{
    public byte[] Generate(ReportData data) { /* ClosedXML */ return Array.Empty<byte>(); }
}

// Factory method (like BudgetPH's ReportsController pattern)
public static IReportGenerator Create(string format) => format.ToLower() switch
{
    "pdf"   => new PdfReportGenerator(),
    "excel" => new ExcelReportGenerator(),
    _       => throw new ArgumentException($"Unknown format: {format}")
};

// Usage
var generator = ReportGeneratorFactory.Create("pdf");
var bytes = generator.Generate(data);
```

---

### Abstract Factory
Creates **families of related objects** without specifying concrete classes.

```csharp
// Abstract factory
public interface IPaymentGatewayFactory
{
    IPaymentProcessor CreateProcessor();
    IRefundHandler CreateRefundHandler();
    IWebhookHandler CreateWebhookHandler();
}

// Concrete factory for GCash
public class GCashFactory : IPaymentGatewayFactory
{
    public IPaymentProcessor CreateProcessor() => new GCashProcessor();
    public IRefundHandler CreateRefundHandler() => new GCashRefundHandler();
    public IWebhookHandler CreateWebhookHandler() => new GCashWebhookHandler();
}

// Concrete factory for PayMaya
public class PayMayaFactory : IPaymentGatewayFactory
{
    public IPaymentProcessor CreateProcessor() => new PayMayaProcessor();
    public IRefundHandler CreateRefundHandler() => new PayMayaRefundHandler();
    public IWebhookHandler CreateWebhookHandler() => new PayMayaWebhookHandler();
}

// Client — uses abstract factory, doesn't care about specific types
public class PaymentService
{
    private readonly IPaymentProcessor _processor;
    private readonly IRefundHandler _refundHandler;

    public PaymentService(IPaymentGatewayFactory factory)
    {
        _processor = factory.CreateProcessor();
        _refundHandler = factory.CreateRefundHandler();
    }
}
```

---

### Builder
Constructs **complex objects step by step**.

```csharp
public class QueryBuilder
{
    private string _table = "";
    private readonly List<string> _conditions = [];
    private readonly List<string> _orderBy = [];
    private int? _limit;

    public QueryBuilder From(string table) { _table = table; return this; }
    public QueryBuilder Where(string condition) { _conditions.Add(condition); return this; }
    public QueryBuilder OrderBy(string column) { _orderBy.Add(column); return this; }
    public QueryBuilder Take(int n) { _limit = n; return this; }

    public string Build()
    {
        var sql = $"SELECT * FROM {_table}";
        if (_conditions.Any()) sql += $" WHERE {string.Join(" AND ", _conditions)}";
        if (_orderBy.Any()) sql += $" ORDER BY {string.Join(", ", _orderBy)}";
        if (_limit.HasValue) sql += $" FETCH NEXT {_limit} ROWS ONLY";
        return sql;
    }
}

// Usage — fluent, readable
var query = new QueryBuilder()
    .From("Transactions")
    .Where("UserId = @userId")
    .Where("Amount > 1000")
    .OrderBy("TransactionDate DESC")
    .Take(50)
    .Build();
// SELECT * FROM Transactions WHERE UserId = @userId AND Amount > 1000 
// ORDER BY TransactionDate DESC FETCH NEXT 50 ROWS ONLY
```

---

### Prototype
Creates new objects by **cloning** existing ones.

```csharp
public class BudgetTemplate : ICloneable
{
    public string Name { get; set; } = "";
    public List<BudgetItemTemplate> Items { get; set; } = [];

    // Shallow clone
    public object Clone() => MemberwiseClone();

    // Deep clone (important for reference types)
    public BudgetTemplate DeepClone()
    {
        var clone = (BudgetTemplate)MemberwiseClone();
        clone.Items = Items.Select(i => i.DeepClone()).ToList(); // deep copy list
        return clone;
    }
}

// Create budget from template (like "copy this month's budget")
var template = existingBudget.DeepClone();
template.Name = $"Budget for {DateTime.Now:MMMM yyyy}";
```

---

## 2. Structural Patterns

### Adapter
Converts the interface of a class into **another interface** that clients expect.

```csharp
// Legacy system you can't modify
public class LegacyPayrollSystem
{
    public double GetMonthlySalary(string employeeCode) { ... }
}

// New interface your application expects
public interface IPayrollService
{
    decimal GetMonthlySalary(Guid employeeId);
}

// Adapter bridges the gap
public class PayrollAdapter : IPayrollService
{
    private readonly LegacyPayrollSystem _legacy;
    private readonly IEmployeeMapper _mapper;

    public PayrollAdapter(LegacyPayrollSystem legacy, IEmployeeMapper mapper)
    {
        _legacy = legacy; _mapper = mapper;
    }

    public decimal GetMonthlySalary(Guid employeeId)
    {
        var code = _mapper.GetCode(employeeId);       // map new ID to old code
        var amount = _legacy.GetMonthlySalary(code);  // call old system
        return (decimal)amount;                        // type conversion
    }
}
```

---

### Decorator
**Wraps** an object to add behavior without modifying the original class.

```csharp
// Base interface
public interface ITransactionService
{
    Task<TransactionDto> CreateAsync(CreateTransactionCommand cmd, CancellationToken ct);
}

// Core implementation
public class TransactionService : ITransactionService
{
    public async Task<TransactionDto> CreateAsync(CreateTransactionCommand cmd, CancellationToken ct)
    {
        // core logic
    }
}

// Decorator — adds caching
public class CachedTransactionService : ITransactionService
{
    private readonly ITransactionService _inner;
    private readonly IMemoryCache _cache;

    public CachedTransactionService(ITransactionService inner, IMemoryCache cache)
    {
        _inner = inner; _cache = cache;
    }

    public async Task<TransactionDto> CreateAsync(CreateTransactionCommand cmd, CancellationToken ct)
    {
        var result = await _inner.CreateAsync(cmd, ct); // delegate to inner
        _cache.Remove($"transactions:{cmd.UserId}");    // invalidate cache
        return result;
    }
}

// Decorator — adds logging
public class LoggedTransactionService : ITransactionService
{
    private readonly ITransactionService _inner;
    private readonly ILogger<LoggedTransactionService> _logger;

    public async Task<TransactionDto> CreateAsync(CreateTransactionCommand cmd, CancellationToken ct)
    {
        _logger.LogInformation("Creating transaction for user {UserId}", cmd.UserId);
        var result = await _inner.CreateAsync(cmd, ct);
        _logger.LogInformation("Transaction created: {Id}", result.Id);
        return result;
    }
}

// Register with DI (decorator chain)
builder.Services.AddScoped<TransactionService>();
builder.Services.AddScoped<ITransactionService>(sp =>
    new LoggedTransactionService(
        new CachedTransactionService(
            sp.GetRequiredService<TransactionService>(),
            sp.GetRequiredService<IMemoryCache>()),
        sp.GetRequiredService<ILogger<LoggedTransactionService>>()));
```

---

### Facade
Provides a **simplified interface** to a complex subsystem.

```csharp
// Complex subsystems
class AccountValidator { public bool Validate(Account a) { ... } }
class CurrencyConverter { public decimal Convert(decimal a, string from, string to) { ... } }
class TransactionRecorder { public void Record(Transaction t) { ... } }
class NotificationSender { public void Notify(string userId, string msg) { ... } }

// Facade — simple interface over complex subsystem
public class MoneyTransferFacade
{
    private readonly AccountValidator _validator;
    private readonly CurrencyConverter _converter;
    private readonly TransactionRecorder _recorder;
    private readonly NotificationSender _notifier;

    public async Task<bool> TransferAsync(Guid fromId, Guid toId, decimal amount)
    {
        var from = GetAccount(fromId);
        var to = GetAccount(toId);

        if (!_validator.Validate(from) || !_validator.Validate(to)) return false;

        var convertedAmount = _converter.Convert(amount, from.Currency, to.Currency);
        
        from.Balance -= amount;
        to.Balance += convertedAmount;
        
        _recorder.Record(new Transaction { ... });
        _notifier.Notify(from.UserId, $"Transfer of {amount} complete");
        
        return true;
    }
}
```

---

### Proxy
Provides a **surrogate or placeholder** for another object, controlling access to it.

```csharp
// Real implementation
public class ExpensiveReportService : IReportService
{
    public async Task<ReportData> GetReportAsync(Guid userId, DateRange range)
    {
        // expensive database queries, PDF generation, etc.
        await Task.Delay(3000); // simulated
        return new ReportData();
    }
}

// Caching proxy
public class CachingReportServiceProxy : IReportService
{
    private readonly IReportService _real;
    private readonly IMemoryCache _cache;

    public CachingReportServiceProxy(IReportService real, IMemoryCache cache)
    {
        _real = real; _cache = cache;
    }

    public async Task<ReportData> GetReportAsync(Guid userId, DateRange range)
    {
        var cacheKey = $"report:{userId}:{range.Start:yyyyMM}:{range.End:yyyyMM}";
        
        if (_cache.TryGetValue(cacheKey, out ReportData? cached))
            return cached!; // return cached result immediately
        
        var result = await _real.GetReportAsync(userId, range); // fetch real
        _cache.Set(cacheKey, result, TimeSpan.FromMinutes(30)); // cache it
        return result;
    }
}
```

---

## 3. Behavioral Patterns

### Strategy
Defines a **family of algorithms** and makes them interchangeable at runtime.

```csharp
// Strategy interface
public interface IInterestCalculationStrategy
{
    decimal Calculate(decimal principal, decimal rate, int months);
}

// Concrete strategies
public class SimpleInterestStrategy : IInterestCalculationStrategy
{
    public decimal Calculate(decimal principal, decimal rate, int months)
        => principal * rate * months / 12;
}

public class CompoundInterestStrategy : IInterestCalculationStrategy
{
    public decimal Calculate(decimal principal, decimal rate, int months)
        => principal * (decimal)Math.Pow((double)(1 + rate / 12), months) - principal;
}

// Context — uses a strategy
public class LoanCalculator
{
    private IInterestCalculationStrategy _strategy;

    public LoanCalculator(IInterestCalculationStrategy strategy)
        => _strategy = strategy;

    // Swap strategy at runtime
    public void SetStrategy(IInterestCalculationStrategy strategy)
        => _strategy = strategy;

    public decimal CalculateInterest(decimal principal, decimal rate, int months)
        => _strategy.Calculate(principal, rate, months);
}

// Usage
var calculator = new LoanCalculator(new SimpleInterestStrategy());
var simple = calculator.CalculateInterest(100000, 0.12m, 12);

calculator.SetStrategy(new CompoundInterestStrategy());
var compound = calculator.CalculateInterest(100000, 0.12m, 12);
```

---

### Observer
Defines a **one-to-many** dependency: when one object changes state, all dependents are notified.

```csharp
// Subject interface
public interface ISubject
{
    void Subscribe(IObserver observer);
    void Unsubscribe(IObserver observer);
    void Notify();
}

// Observer interface
public interface IObserver
{
    void Update(string eventType, object data);
}

// Concrete subject
public class TransactionEventPublisher : ISubject
{
    private readonly List<IObserver> _observers = [];

    public void Subscribe(IObserver observer) => _observers.Add(observer);
    public void Unsubscribe(IObserver observer) => _observers.Remove(observer);
    
    public void Notify() { } // not used directly

    public void PublishTransactionCreated(Transaction transaction)
    {
        foreach (var observer in _observers)
            observer.Update("TransactionCreated", transaction);
    }
}

// Concrete observer — updates budget when transaction created
public class BudgetUpdateObserver : IObserver
{
    public void Update(string eventType, object data)
    {
        if (eventType == "TransactionCreated" && data is Transaction t)
        {
            // Update budget spent amount
            Console.WriteLine($"Budget updated for category: {t.CategoryId}");
        }
    }
}

// In .NET, prefer events or MediatR notifications over manual Observer pattern
```

---

### Command Pattern
Encapsulates a request as an **object**, allowing undo/redo, queuing, and logging.

```csharp
// Command interface
public interface ICommand
{
    Task ExecuteAsync();
    Task UndoAsync();
}

// Concrete command
public class CreateTransactionCommand : ICommand
{
    private readonly ApplicationDbContext _db;
    private readonly Transaction _transaction;
    private Guid _createdId;

    public CreateTransactionCommand(ApplicationDbContext db, Transaction transaction)
    {
        _db = db; _transaction = transaction;
    }

    public async Task ExecuteAsync()
    {
        _db.Transactions.Add(_transaction);
        await _db.SaveChangesAsync();
        _createdId = _transaction.Id;
    }

    public async Task UndoAsync()
    {
        var t = await _db.Transactions.FindAsync(_createdId);
        if (t != null) { _db.Transactions.Remove(t); await _db.SaveChangesAsync(); }
    }
}

// Command invoker with undo stack
public class TransactionCommandHandler
{
    private readonly Stack<ICommand> _history = new();

    public async Task ExecuteAsync(ICommand command)
    {
        await command.ExecuteAsync();
        _history.Push(command);
    }

    public async Task UndoLastAsync()
    {
        if (_history.TryPop(out var command))
            await command.UndoAsync();
    }
}
```

---

### Template Method
Defines the **skeleton of an algorithm** in a base class, letting subclasses fill in specific steps.

```csharp
public abstract class ReportGenerator
{
    // Template method — defines the algorithm structure
    public async Task<byte[]> GenerateAsync(ReportRequest request)
    {
        var data = await FetchDataAsync(request);    // step 1
        var validated = Validate(data);               // step 2
        var formatted = Format(validated);            // step 3
        return await RenderAsync(formatted);          // step 4
    }

    protected abstract Task<ReportData> FetchDataAsync(ReportRequest request);
    protected abstract ReportData Validate(ReportData data);
    protected abstract FormattedReport Format(ReportData data);
    protected abstract Task<byte[]> RenderAsync(FormattedReport report);
}

public class PdfReportGenerator : ReportGenerator
{
    protected override async Task<ReportData> FetchDataAsync(ReportRequest req)
    {
        // fetch data from DB
        return new ReportData();
    }
    protected override ReportData Validate(ReportData data) { return data; }
    protected override FormattedReport Format(ReportData data) { return new FormattedReport(); }
    protected override async Task<byte[]> RenderAsync(FormattedReport report)
    {
        // use QuestPDF to render
        return Array.Empty<byte>();
    }
}
```

---

### Chain of Responsibility
Passes a request along a **chain of handlers** until one handles it.

```csharp
public abstract class ApprovalHandler
{
    protected ApprovalHandler? _next;
    
    public ApprovalHandler SetNext(ApprovalHandler next) { _next = next; return next; }
    
    public abstract bool Handle(ExpenseRequest request);
}

public class ManagerApproval : ApprovalHandler
{
    public override bool Handle(ExpenseRequest request)
    {
        if (request.Amount <= 5000) { Console.WriteLine("Manager approved."); return true; }
        return _next?.Handle(request) ?? false;
    }
}

public class DirectorApproval : ApprovalHandler
{
    public override bool Handle(ExpenseRequest request)
    {
        if (request.Amount <= 50000) { Console.WriteLine("Director approved."); return true; }
        return _next?.Handle(request) ?? false;
    }
}

public class CeoApproval : ApprovalHandler
{
    public override bool Handle(ExpenseRequest request)
    {
        Console.WriteLine("CEO approved.");
        return true; // CEO approves anything
    }
}

// Chain setup
var chain = new ManagerApproval();
chain.SetNext(new DirectorApproval()).SetNext(new CeoApproval());

chain.Handle(new ExpenseRequest { Amount = 4000 });  // Manager approved
chain.Handle(new ExpenseRequest { Amount = 30000 }); // Director approved
chain.Handle(new ExpenseRequest { Amount = 200000 }); // CEO approved
```

---

## 4. Common Interview Questions

**Q1: What is the difference between Factory Method and Abstract Factory?**

> **Factory Method** uses inheritance — a subclass overrides a method to create one type of object. **Abstract Factory** uses composition — it creates families of related objects. Factory Method answers "which type?"; Abstract Factory answers "which family?".

**Q2: When should you NOT use a Singleton?**

> Avoid Singleton when: (1) The object has mutable state shared across threads (thread safety issues), (2) Unit testing — Singletons are hard to mock, (3) You need different instances for different contexts. In ASP.NET Core, DI manages Singleton lifetime — prefer that over hand-rolled Singletons.

**Q3: What's the difference between Decorator and Proxy?**

> Both wrap another object. **Decorator** adds new behavior to an object. **Proxy** controls access (lazy loading, caching, authorization). The intent is different: Decorator enhances; Proxy controls.

**Q4: Explain the difference between Strategy and State patterns.**

> **Strategy** — the client actively chooses the algorithm. Different strategies do the same thing differently (sorting strategies: bubble, quick, merge). **State** — the object itself changes behavior based on internal state. The state transitions happen internally based on conditions.

**Q5: How is CQRS related to the Command Pattern?**

> CQRS (architectural pattern) applies the **Command Pattern** (GoF) at an architectural level. Commands change state; Queries read state. MediatR's `IRequest<T>` is the Command pattern. The key addition CQRS makes is that the read and write models can be entirely separate — different databases, different representations.

**Q6: What is the Observer pattern in .NET?**

> .NET has built-in implementations of Observer: **events** (EventHandler), **IObservable<T>/IObserver<T>** (Reactive Extensions), and **INotificationHandler<T>** (MediatR). The `INotificationHandler<T>` in MediatR is essentially the Observer pattern — one notification can have multiple handlers.
