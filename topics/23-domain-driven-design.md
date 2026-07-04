# Topic 23 — Domain-Driven Design (DDD)

> DDD is the architectural approach most senior .NET developers are expected to understand. It aligns code structure with business concepts.

---

## Strategic DDD

### Bounded Contexts
A Bounded Context is a boundary within which a domain model has a specific, consistent meaning.

```
BudgetPH — Bounded Contexts:

┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│   ACCOUNTS      │  │  TRANSACTIONS   │  │    BUDGETS      │
│                 │  │                 │  │                 │
│  Account        │  │  Transaction    │  │  Budget         │
│  Balance        │  │  Amount         │  │  BudgetItem     │
│  AccountType    │  │  Category       │  │  Allocation     │
└────────┬────────┘  └────────┬────────┘  └────────┬────────┘
         │ AccountId           │ AccountId          │ UserId
         └─────────────────────┴────────────────────┘
                       Shared Kernel: UserId, Money
```

Each context has its own model. "Account" means something different in Accounts context vs Transactions context.

### Context Map
How bounded contexts interact:
- **Shared Kernel** — shared code (UserId, Money value object)
- **Customer/Supplier** — upstream context defines API, downstream consumes
- **Anticorruption Layer (ACL)** — translates between foreign models
- **Published Language** — integration events as a shared contract

---

## Tactical DDD Building Blocks

### Entities

```csharp
// Entity — has identity (Id), mutable over time:
public class Account : BaseEntity // BaseEntity has Id, CreatedAt, etc.
{
    // Private setters — controlled mutations via methods:
    public string Name { get; private set; }
    public decimal Balance { get; private set; }
    public AccountType Type { get; private set; }
    
    // Factory method — validates invariants on creation:
    public static Result<Account> Create(string name, AccountType type, decimal openingBalance)
    {
        if (string.IsNullOrWhiteSpace(name))
            return Result.Failure<Account>("Account name is required");
        if (openingBalance < 0)
            return Result.Failure<Account>("Opening balance cannot be negative");
        
        var account = new Account
        {
            Name = name,
            Type = type,
            Balance = openingBalance
        };
        account.RaiseDomainEvent(new AccountCreatedEvent(account.Id, name, type));
        return Result.Success(account);
    }
    
    // Domain methods — enforce business rules:
    public Result Deposit(Money amount)
    {
        if (amount.Value <= 0)
            return Result.Failure("Deposit amount must be positive");
        
        Balance += amount.Value;
        RaiseDomainEvent(new FundsDepositedEvent(Id, amount));
        return Result.Success();
    }
    
    public Result Withdraw(Money amount)
    {
        if (amount.Value <= 0)
            return Result.Failure("Withdrawal amount must be positive");
        if (Balance < amount.Value)
            return Result.Failure("Insufficient funds");
        
        Balance -= amount.Value;
        RaiseDomainEvent(new FundsWithdrawnEvent(Id, amount));
        return Result.Success();
    }
}
```

### Value Objects

```csharp
// Value Object — no identity, equality by value, immutable:
public record Money
{
    public decimal Value { get; }
    public string Currency { get; }
    
    public Money(decimal value, string currency)
    {
        if (value < 0) throw new DomainException("Money value cannot be negative");
        if (string.IsNullOrWhiteSpace(currency)) throw new DomainException("Currency required");
        Value = Math.Round(value, 2);
        Currency = currency.ToUpperInvariant();
    }
    
    public static Money Zero(string currency) => new(0, currency);
    public static Money PHP(decimal amount) => new(amount, "PHP");
    
    public Money Add(Money other)
    {
        if (Currency != other.Currency)
            throw new DomainException($"Cannot add {Currency} and {other.Currency}");
        return new Money(Value + other.Value, Currency);
    }
    
    public Money Subtract(Money other)
    {
        if (Currency != other.Currency)
            throw new DomainException("Currency mismatch");
        return new Money(Value - other.Value, Currency);
    }
    
    public override string ToString() => $"{Currency} {Value:N2}";
}

// More value objects:
public record EmailAddress
{
    public string Value { get; }
    
    public EmailAddress(string value)
    {
        if (!IsValid(value)) throw new DomainException($"Invalid email: {value}");
        Value = value.ToLowerInvariant();
    }
    
    private static bool IsValid(string email) =>
        !string.IsNullOrWhiteSpace(email) && email.Contains('@') && email.Contains('.');
    
    public static implicit operator string(EmailAddress e) => e.Value;
    public static explicit operator EmailAddress(string s) => new(s);
}

public record PhilippinePhoneNumber
{
    public string Value { get; }
    
    public PhilippinePhoneNumber(string value)
    {
        var cleaned = value.Replace("-", "").Replace(" ", "").Replace("+63", "0");
        if (!cleaned.StartsWith("09") || cleaned.Length != 11)
            throw new DomainException("Invalid PH phone number");
        Value = cleaned;
    }
}
```

### Aggregates

```csharp
// Aggregate Root — consistency boundary, external access only through root:
public class Budget : BaseEntity // Budget is the Aggregate Root
{
    public Guid UserId { get; private set; }
    public string Name { get; private set; }
    public int Month { get; private set; }
    public int Year { get; private set; }
    
    // Private collection — can only be modified through Budget methods:
    private readonly List<BudgetItem> _items = [];
    public IReadOnlyList<BudgetItem> Items => _items.AsReadOnly();
    
    public Money TotalAllocated => new Money(_items.Sum(i => i.AllocatedAmount), "PHP");
    
    public Result AddItem(Guid categoryId, decimal allocatedAmount)
    {
        if (_items.Any(i => i.CategoryId == categoryId))
            return Result.Failure("Category already has a budget item");
        if (allocatedAmount <= 0)
            return Result.Failure("Allocated amount must be positive");
        
        var item = new BudgetItem(Id, categoryId, allocatedAmount);
        _items.Add(item);
        RaiseDomainEvent(new BudgetItemAddedEvent(Id, categoryId, allocatedAmount));
        return Result.Success();
    }
    
    public Result RecordSpending(Guid categoryId, decimal amount)
    {
        var item = _items.FirstOrDefault(i => i.CategoryId == categoryId);
        if (item is null)
            return Result.Failure("No budget item for this category");
        
        item.RecordSpending(amount);
        
        if (item.IsOverBudget)
            RaiseDomainEvent(new BudgetExceededEvent(Id, UserId, categoryId, item.SpentAmount, item.AllocatedAmount));
        
        return Result.Success();
    }
}

// BudgetItem is a child entity — only accessed through Budget:
public class BudgetItem
{
    public Guid BudgetId { get; private set; }
    public Guid CategoryId { get; private set; }
    public decimal AllocatedAmount { get; private set; }
    public decimal SpentAmount { get; private set; }
    public bool IsOverBudget => SpentAmount > AllocatedAmount;
    public decimal Remaining => AllocatedAmount - SpentAmount;
    
    internal void RecordSpending(decimal amount) => SpentAmount += amount;
}
```

### Domain Services

```csharp
// Domain Service — business logic that doesn't naturally belong to one entity:
public class FundsTransferService
{
    // Transfer involves TWO accounts — doesn't belong to either Account entity:
    public Result Transfer(Account from, Account to, Money amount)
    {
        var withdrawResult = from.Withdraw(amount);
        if (!withdrawResult.IsSuccess)
            return withdrawResult;
        
        var depositResult = to.Deposit(amount);
        if (!depositResult.IsSuccess)
        {
            // Compensate — this is why sagas exist for distributed scenarios
            from.Deposit(amount);
            return depositResult;
        }
        
        return Result.Success();
    }
}

// Application Service — orchestrates domain objects, NOT domain logic:
public class TransferCommandHandler : IRequestHandler<TransferCommand, Result>
{
    public async Task<Result> Handle(TransferCommand cmd, CancellationToken ct)
    {
        var fromAccount = await _accountRepo.GetByIdAsync(cmd.FromAccountId, ct);
        var toAccount   = await _accountRepo.GetByIdAsync(cmd.ToAccountId, ct);
        
        if (fromAccount is null || toAccount is null)
            return Result.Failure("Account not found");
        
        // Delegate to domain service:
        var result = _transferService.Transfer(fromAccount, toAccount, new Money(cmd.Amount, "PHP"));
        
        if (result.IsSuccess)
            await _unitOfWork.SaveChangesAsync(ct);
        
        return result;
    }
}
```

---

## Repository Pattern in DDD

```csharp
// Repository interface in Domain layer — no EF Core:
public interface IAccountRepository
{
    Task<Account?> GetByIdAsync(Guid id, CancellationToken ct);
    Task<List<Account>> GetByUserIdAsync(Guid userId, CancellationToken ct);
    Task AddAsync(Account account, CancellationToken ct);
    void Update(Account account);
    void Remove(Account account);
}

// Implementation in Infrastructure:
public class EfAccountRepository : IAccountRepository
{
    private readonly AppDbContext _db;
    
    public async Task<Account?> GetByIdAsync(Guid id, CancellationToken ct)
        => await _db.Accounts
            .Include(a => a.Transactions) // load what aggregate needs
            .FirstOrDefaultAsync(a => a.Id == id, ct);
    
    public async Task AddAsync(Account account, CancellationToken ct)
        => await _db.Accounts.AddAsync(account, ct);
    
    public void Update(Account account) => _db.Accounts.Update(account);
    public void Remove(Account account) => _db.Accounts.Remove(account);
}
```

---

## DDD vs CRUD vs Rich Domain Model

```csharp
// ❌ Anemic Domain Model (CRUD — business logic in services):
public class Transaction { public decimal Amount { get; set; } } // just data
public class TransactionService
{
    public void Approve(Transaction tx) { tx.Status = "Approved"; } // logic outside
}

// ✅ Rich Domain Model (DDD — business logic IN the entity):
public class Transaction : BaseEntity
{
    public decimal Amount { get; private set; }
    public TransactionStatus Status { get; private set; }
    
    public Result Approve(string approvedBy)
    {
        if (Status != TransactionStatus.Pending)
            return Result.Failure("Only pending transactions can be approved");
        
        Status = TransactionStatus.Approved;
        RaiseDomainEvent(new TransactionApprovedEvent(Id, approvedBy));
        return Result.Success();
    }
}
```

---

## Interview Q&A

**Q1: What is the difference between an Entity and a Value Object?**
> Entity has a unique identity (Id) that persists over time — two entities are different even if all properties match (`user1 ≠ user2` even if same name). Value Object has no identity — equality is based entirely on its properties (`Money(100, "PHP") == Money(100, "PHP")`). Value Objects are immutable; to change them, you create a new one.

**Q2: What is an Aggregate and why does it matter?**
> An Aggregate is a cluster of domain objects (root + children) treated as a single unit for data consistency. All mutations go through the Aggregate Root. External code never holds references to child entities — only to the root. This enforces consistency boundaries and simplifies concurrency.

**Q3: What is the difference between a Domain Service and an Application Service?**
> A Domain Service contains business logic that doesn't belong to a single entity (e.g., fund transfer between two accounts). It works with domain objects only. An Application Service orchestrates the use case: loads aggregates from repositories, calls domain services/methods, saves via unit of work. Application Services are thin — the real logic is in the domain.

**Q4: What is an Anticorruption Layer (ACL)?**
> An ACL translates between your domain model and a foreign/external model. When integrating with a legacy system or third-party API, the ACL prevents their messy model from polluting yours. It maps external DTOs to your domain entities and vice versa. Named because it "protects" your domain from corruption by external models.

**Q5: What is the difference between strategic and tactical DDD?**
> Strategic DDD deals with the big picture: identifying Bounded Contexts, defining Context Maps, and organizing teams. Tactical DDD provides building blocks within a bounded context: Entities, Value Objects, Aggregates, Domain Events, Repositories, Domain Services. Strategic first, tactical within each bounded context.
