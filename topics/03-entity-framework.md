# 03 — Entity Framework Core

> 🔗 **BudgetPH reference**: [ApplicationDbContext.cs](https://github.com/randolfsegubre/BudgetPH/blob/master/src/Infrastructure/FinanceManager.Infrastructure/Data/ApplicationDbContext.cs) | [Migrations/](https://github.com/randolfsegubre/BudgetPH/blob/master/src/Infrastructure/FinanceManager.Infrastructure/Migrations/)

---

## Table of Contents
1. [Code-First vs Database-First](#1-code-first-vs-database-first)
2. [DbContext & DbSet](#2-dbcontext--dbset)
3. [Relationships & Navigation Properties](#3-relationships--navigation-properties)
4. [Migrations](#4-migrations)
5. [Querying — LINQ to SQL](#5-querying--linq-to-sql)
6. [Eager vs Lazy vs Explicit Loading](#6-eager-vs-lazy-vs-explicit-loading)
7. [Change Tracking](#7-change-tracking)
8. [Transactions](#8-transactions)
9. [Performance Tips](#9-performance-tips)
10. [Raw SQL](#10-raw-sql)
11. [Common Interview Questions](#11-common-interview-questions)

---

## 1. Code-First vs Database-First

| Approach | Description | When to Use |
|---|---|---|
| **Code-First** | Write C# classes → EF generates DB schema | New projects, you own the DB |
| **Database-First** | Existing DB → EF scaffolds C# classes | Legacy DB, DBA owns schema |
| **Model-First** | Design diagram → generate both (deprecated) | Rarely used today |

**BudgetPH uses Code-First** — all entities are C# classes, migrations create the database.

---

## 2. DbContext & DbSet

### 🔗 BudgetPH DbContext

From [ApplicationDbContext.cs](https://github.com/randolfsegubre/BudgetPH/blob/master/src/Infrastructure/FinanceManager.Infrastructure/Data/ApplicationDbContext.cs):

```csharp
public class ApplicationDbContext : IdentityDbContext<IdentityUser>
{
    public ApplicationDbContext(DbContextOptions<ApplicationDbContext> options)
        : base(options) { }

    // Each DbSet<T> maps to a database table
    public DbSet<ApplicationUser> AppUsers => Set<ApplicationUser>();
    public DbSet<Account> Accounts => Set<Account>();
    public DbSet<Transaction> Transactions => Set<Transaction>();
    public DbSet<Budget> Budgets => Set<Budget>();
    public DbSet<SavingsGoal> SavingsGoals => Set<SavingsGoal>();
    public DbSet<Category> Categories => Set<Category>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        base.OnModelCreating(modelBuilder);
        
        // Fluent API configuration
        modelBuilder.Entity<Account>(entity =>
        {
            entity.HasKey(a => a.Id);
            entity.Property(a => a.Balance).HasPrecision(18, 4);
            entity.HasOne(a => a.User)
                  .WithMany(u => u.Accounts)
                  .HasForeignKey(a => a.UserId)
                  .OnDelete(DeleteBehavior.Cascade);
        });
        
        // Seed data
        SeedCategories(modelBuilder);
    }
}
```

### Registration

```csharp
// Scoped by default — perfect for EF Core
builder.Services.AddDbContext<ApplicationDbContext>(options =>
    options.UseSqlServer(configuration.GetConnectionString("DefaultConnection")));
```

---

## 3. Relationships & Navigation Properties

### One-to-Many (most common)

```csharp
// One User has many Accounts
public class ApplicationUser
{
    public Guid Id { get; set; }
    public string Email { get; set; } = "";
    
    // Navigation property — collection
    public ICollection<Account> Accounts { get; set; } = new List<Account>();
}

public class Account
{
    public Guid Id { get; set; }
    public Guid UserId { get; set; }  // Foreign Key
    
    // Navigation property — reference
    public ApplicationUser User { get; set; } = null!;
}

// Fluent API (in OnModelCreating)
modelBuilder.Entity<Account>()
    .HasOne(a => a.User)
    .WithMany(u => u.Accounts)
    .HasForeignKey(a => a.UserId)
    .OnDelete(DeleteBehavior.Cascade);
```

### One-to-One

```csharp
// Account → CreditCardDetails (only one credit card per account)
public class Account
{
    public CreditCardDetails? CreditCardDetails { get; set; }
}

public class CreditCardDetails
{
    public Guid AccountId { get; set; }  // FK is also PK
    public Account Account { get; set; } = null!;
}

// Fluent API
modelBuilder.Entity<CreditCardDetails>()
    .HasOne(c => c.Account)
    .WithOne(a => a.CreditCardDetails)
    .HasForeignKey<CreditCardDetails>(c => c.AccountId);
```

### Many-to-Many

```csharp
// Transaction ↔ Tags (many-to-many with join entity)
public class Transaction
{
    public ICollection<TransactionTag> TransactionTags { get; set; } = [];
}

public class Tag
{
    public ICollection<TransactionTag> TransactionTags { get; set; } = [];
}

public class TransactionTag  // Join entity
{
    public Guid TransactionId { get; set; }
    public Transaction Transaction { get; set; } = null!;
    public Guid TagId { get; set; }
    public Tag Tag { get; set; } = null!;
}

// OR — EF Core 5+ direct many-to-many (no join entity needed)
public class Transaction
{
    public ICollection<Tag> Tags { get; set; } = [];
}

public class Tag
{
    public ICollection<Transaction> Transactions { get; set; } = [];
}
// EF Core creates the join table automatically
```

### Cascade Delete Behaviors

| Behavior | Effect |
|---|---|
| `Cascade` | Delete parent → deletes children |
| `SetNull` | Delete parent → sets FK to null in children |
| `Restrict` | Delete parent → throws if children exist |
| `NoAction` | No action (like Restrict but deferred) |
| `ClientSetNull` | EF sets FK to null (not DB-level) |

---

## 4. Migrations

### CLI Commands (from BudgetPH setup)

```powershell
# Add a migration
dotnet ef migrations add InitialCreate \
  --project src/Infrastructure/FinanceManager.Infrastructure \
  --startup-project src/Presentation/FinanceManager.API

# Apply migrations to database
dotnet ef database update \
  --project src/Infrastructure/FinanceManager.Infrastructure \
  --startup-project src/Presentation/FinanceManager.API

# Revert last migration
dotnet ef migrations remove

# Generate SQL script (for production)
dotnet ef migrations script \
  --from InitialCreate --to AddUserPreferences \
  --output migrations.sql
```

### Auto-migrate on startup (BudgetPH pattern)

```csharp
// 🔗 From Program.cs
using (var scope = app.Services.CreateScope())
{
    var db = scope.ServiceProvider.GetRequiredService<ApplicationDbContext>();
    await db.Database.MigrateAsync();     // applies pending migrations
    await DbSeeder.SeedAsync(db);          // seeds initial data
}
```

### Migration File Structure

```csharp
// Auto-generated migration
public partial class AddTransactionTags : Migration
{
    protected override void Up(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.CreateTable(
            name: "Tags",
            columns: table => new
            {
                Id = table.Column<Guid>(nullable: false),
                Name = table.Column<string>(maxLength: 50, nullable: false)
            });
    }

    protected override void Down(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.DropTable(name: "Tags");
    }
}
```

---

## 5. Querying — LINQ to SQL

### Basic CRUD

```csharp
// SELECT
var accounts = await context.Accounts
    .Where(a => a.UserId == userId && a.IsActive)
    .OrderBy(a => a.Name)
    .ToListAsync(ct);

// SELECT with projection (avoid over-fetching)
var accountNames = await context.Accounts
    .Where(a => a.UserId == userId)
    .Select(a => new { a.Id, a.Name, a.Balance })  // only fetch what you need
    .ToListAsync(ct);

// INSERT
var account = new Account { Name = "BPI Savings", UserId = userId, Balance = 5000 };
context.Accounts.Add(account);
await context.SaveChangesAsync(ct);

// UPDATE
var account = await context.Accounts.FindAsync(accountId);
account.Balance += 1000;
await context.SaveChangesAsync(ct);

// DELETE
var account = await context.Accounts.FindAsync(accountId);
context.Accounts.Remove(account);
await context.SaveChangesAsync(ct);

// DELETE without loading entity (EF Core 7+)
await context.Accounts
    .Where(a => a.Id == accountId)
    .ExecuteDeleteAsync(ct);

// UPDATE without loading entity (EF Core 7+)
await context.Accounts
    .Where(a => a.UserId == userId && !a.IsActive)
    .ExecuteUpdateAsync(setters => setters
        .SetProperty(a => a.UpdatedAt, DateTime.UtcNow), ct);
```

### Filtering, Sorting, Paging

```csharp
// Paginated transactions (pattern used in BudgetPH)
public async Task<PagedResult<TransactionDto>> GetTransactionsAsync(
    Guid userId, int page, int pageSize, string? search, CancellationToken ct)
{
    var query = context.Transactions
        .Where(t => t.UserId == userId)
        .AsQueryable();  // keep as IQueryable for deferred execution

    // Apply optional filters
    if (!string.IsNullOrWhiteSpace(search))
        query = query.Where(t => t.Description.Contains(search) || 
                                  t.Merchant!.Contains(search));

    var total = await query.CountAsync(ct);  // separate count query
    
    var items = await query
        .OrderByDescending(t => t.TransactionDate)
        .Skip((page - 1) * pageSize)
        .Take(pageSize)
        .Select(t => new TransactionDto
        {
            Id = t.Id,
            Description = t.Description,
            Amount = t.Amount,
            CategoryName = t.Category!.Name
        })
        .ToListAsync(ct);

    return new PagedResult<TransactionDto>(items, total, page, pageSize);
}
```

---

## 6. Eager vs Lazy vs Explicit Loading

### Eager Loading (`Include` / `ThenInclude`)

Loads related entities **in the same query** using SQL JOIN.

```csharp
// Include navigation property
var accounts = await context.Accounts
    .Include(a => a.Transactions)          // load transactions
    .Include(a => a.CreditCardDetails)     // load CC details
    .Where(a => a.UserId == userId)
    .ToListAsync(ct);

// ThenInclude — nested navigation
var budgets = await context.Budgets
    .Include(b => b.Items)
        .ThenInclude(i => i.Category)      // Items → Category
    .Where(b => b.UserId == userId)
    .ToListAsync(ct);

// Filtered includes (EF Core 5+)
var accounts = await context.Accounts
    .Include(a => a.Transactions.Where(t => t.TransactionDate >= DateTime.Today.AddMonths(-1)))
    .ToListAsync(ct);
```

### Lazy Loading

Loads related entities **on access** (additional SQL query per property access). Requires:
- Virtual navigation properties
- `UseLazyLoadingProxies()` or `ILazyLoader` injection

```csharp
// Setup
services.AddDbContext<AppDbContext>(options =>
    options.UseLazyLoadingProxies()
           .UseSqlServer(connStr));

// Entity must use virtual
public class Account
{
    public virtual ICollection<Transaction> Transactions { get; set; } = [];
    // ↑ accessed lazily — triggers a new DB query
}

// ⚠️ N+1 Problem with lazy loading:
foreach (var account in accounts)  // 1 query for accounts
{
    var count = account.Transactions.Count; // N queries (one per account) ← BAD
}

// ✅ Fix with eager loading:
var accounts = await context.Accounts
    .Include(a => a.Transactions)
    .ToListAsync(); // 1 or 2 queries total
```

### Explicit Loading

Load related data on demand, after the entity is already loaded:

```csharp
var account = await context.Accounts.FindAsync(id);

// Later, explicitly load related data
await context.Entry(account)
    .Collection(a => a.Transactions)
    .LoadAsync(ct);

await context.Entry(account)
    .Reference(a => a.CreditCardDetails)
    .LoadAsync(ct);
```

---

## 7. Change Tracking

EF Core's change tracker monitors entity states:

| State | Meaning |
|---|---|
| `Added` | New entity, will be INSERT on SaveChanges |
| `Modified` | Existing entity was changed, will be UPDATE |
| `Deleted` | Marked for removal, will be DELETE |
| `Unchanged` | Loaded from DB, no changes |
| `Detached` | Not tracked by context |

```csharp
var account = await context.Accounts.FindAsync(id); // State: Unchanged
account.Balance = 9999;                              // State: Modified (auto-detected)
await context.SaveChangesAsync();                    // executes UPDATE

// Manually set state
context.Entry(account).State = EntityState.Modified;

// Detach to stop tracking (useful for performance)
context.Entry(account).State = EntityState.Detached;

// AsNoTracking — read-only queries (faster, less memory)
var accounts = await context.Accounts
    .AsNoTracking()
    .Where(a => a.UserId == userId)
    .ToListAsync(ct);
// ↑ No change tracking overhead — use for read-only operations
```

---

## 8. Transactions

### Implicit Transactions
`SaveChanges()` automatically wraps all changes in a transaction:
```csharp
context.Accounts.Add(fromAccount);     // INSERT
context.Accounts.Add(toAccount);       // INSERT  
await context.SaveChangesAsync(ct);    // ← both in same transaction
```

### Explicit Transactions

```csharp
// For operations spanning multiple SaveChanges calls
using var transaction = await context.Database.BeginTransactionAsync(ct);
try
{
    // Debit from source account
    fromAccount.Balance -= amount;
    await context.SaveChangesAsync(ct);
    
    // Credit to destination account
    toAccount.Balance += amount;
    await context.SaveChangesAsync(ct);
    
    // Create transaction record
    context.Transactions.Add(new Transaction { Amount = amount, ... });
    await context.SaveChangesAsync(ct);
    
    await transaction.CommitAsync(ct);
}
catch
{
    await transaction.RollbackAsync(ct);
    throw;
}
```

### Isolation Levels

```csharp
using var transaction = await context.Database.BeginTransactionAsync(
    IsolationLevel.ReadCommitted, ct);
```

---

## 9. Performance Tips

### 1. AsNoTracking for Read-Only Queries

```csharp
// ✅ Use when you won't modify the entities
var reports = await context.Transactions
    .AsNoTracking()
    .Where(t => t.UserId == userId)
    .ToListAsync(ct);
```

### 2. Select Only What You Need

```csharp
// ❌ Over-fetch — loads entire entity
var accounts = await context.Accounts.ToListAsync();
var names = accounts.Select(a => a.Name).ToList();

// ✅ Project early — only Name is fetched from DB
var names = await context.Accounts.Select(a => a.Name).ToListAsync();
```

### 3. Split Queries (EF Core 5+)

```csharp
// ❌ Single JOIN query — can cause Cartesian explosion with multiple collections
var users = await context.Users
    .Include(u => u.Orders)   // many
    .Include(u => u.Tags)     // many — cartesian explosion!
    .ToListAsync();

// ✅ Split into multiple queries
var users = await context.Users
    .Include(u => u.Orders)
    .Include(u => u.Tags)
    .AsSplitQuery()  // executes separate SQL SELECT for each include
    .ToListAsync();
```

### 4. Compiled Queries

```csharp
// Pre-compiled query — avoids query compilation overhead on each call
private static readonly Func<ApplicationDbContext, Guid, Task<List<Account>>>
    GetAccountsQuery = EF.CompileAsyncQuery(
        (ApplicationDbContext ctx, Guid userId) =>
            ctx.Accounts.Where(a => a.UserId == userId));

// Usage
var accounts = await GetAccountsQuery(context, userId);
```

### 5. Bulk Operations (EF Core 7+)

```csharp
// ✅ Bulk delete without loading entities
await context.Transactions
    .Where(t => t.TransactionDate < DateTime.UtcNow.AddYears(-5))
    .ExecuteDeleteAsync(ct);

// ✅ Bulk update without loading entities
await context.Accounts
    .Where(a => !a.IsActive)
    .ExecuteUpdateAsync(s => s
        .SetProperty(a => a.UpdatedAt, DateTime.UtcNow)
        .SetProperty(a => a.Notes, "Archived"), ct);
```

---

## 10. Raw SQL

```csharp
// Query returning entities
var accounts = await context.Accounts
    .FromSqlRaw("SELECT * FROM Accounts WHERE UserId = {0}", userId)
    .ToListAsync(ct);

// ✅ Parameterized (prevents SQL injection)
var accounts = await context.Accounts
    .FromSqlInterpolated($"SELECT * FROM Accounts WHERE UserId = {userId}")
    .ToListAsync(ct);

// Non-query (INSERT, UPDATE, DELETE)
await context.Database.ExecuteSqlInterpolatedAsync(
    $"UPDATE Accounts SET Balance = Balance + {amount} WHERE Id = {accountId}", ct);

// Custom SQL result (no mapped entity)
var results = await context.Database
    .SqlQueryRaw<MonthlyTotalDto>("EXEC sp_GetMonthlyTotals @UserId", userId)
    .ToListAsync(ct);
```

---

## 11. Common Interview Questions

**Q1: What is the N+1 problem in EF Core and how do you fix it?**

> The N+1 problem occurs when you load N entities and then access a navigation property on each, causing N additional queries (1 query for the list + N queries for related data). Fix: use `Include()` for eager loading, which generates a JOIN and loads everything in 1-2 queries. With `AsSplitQuery()`, EF runs separate queries but still avoids the loop.

**Q2: What is the difference between `AsNoTracking` and `AsNoTrackingWithIdentityResolution`?**

> `AsNoTracking()` — fastest, no identity map. If the same entity appears multiple times in results (e.g., multiple transactions with the same category), EF creates multiple objects. `AsNoTrackingWithIdentityResolution()` — maintains an identity map (one object per ID) but doesn't track changes. Use when you need correct reference equality without modification tracking.

**Q3: Explain `SaveChanges` vs `SaveChangesAsync`.**

> `SaveChanges()` is synchronous — blocks the thread while waiting for the database. `SaveChangesAsync()` is async — releases the thread during the I/O operation, better for scalability in web apps. Always prefer `SaveChangesAsync()` in ASP.NET Core controllers.

**Q4: How do you handle optimistic concurrency in EF Core?**

> Add a `[Timestamp]` or `rowversion` column (SQL Server) or a `[ConcurrencyCheck]` attribute to sensitive properties. EF Core will include these in `WHERE` clauses when updating. If the row was modified by another user, EF throws `DbUpdateConcurrencyException`, which you can catch and handle (retry, merge, or notify the user).

**Q5: What is the difference between `Find()` and `SingleOrDefault()`?**

> `Find(id)` — checks the context's local change tracker first, then queries the DB if not found. Useful for by-PK lookups. `SingleOrDefault()` — always queries the database. Use `Find()` for PK lookups (more efficient); use `Where().FirstOrDefault()` for complex filters.

**Q6: When would you use `ExecuteUpdate`/`ExecuteDelete` (EF Core 7)?**

> When performing bulk operations without needing to load entities into memory. Example: archiving old transactions, updating all inactive users. They translate directly to SQL `UPDATE`/`DELETE` without materializing objects — much faster for large datasets. They bypass change tracking and do not trigger EF Core interceptors or domain events.

**Q7: What is a migration and how do you apply it?**

> A migration is a code representation of database schema changes over time. EF Core compares the current model against the last migration snapshot and generates `Up()` and `Down()` methods. Apply with `dotnet ef database update` or programmatically with `await context.Database.MigrateAsync()` on app startup.

**Q8: Explain Fluent API vs Data Annotations — which is preferred?**

> Data Annotations (`[Required]`, `[MaxLength]`) are convenient for simple rules but mix infrastructure concerns into domain entities. Fluent API (`OnModelCreating`) keeps domain classes clean and supports all configuration options. For Clean Architecture (like BudgetPH), prefer Fluent API so domain entities remain infrastructure-agnostic.
