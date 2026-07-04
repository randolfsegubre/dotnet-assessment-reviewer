# 09 — Testing Strategies

> Covers unit testing, integration testing, mocking, and test-driven development.

---

## Table of Contents
1. [Testing Pyramid](#1-testing-pyramid)
2. [xUnit — Unit Testing](#2-xunit--unit-testing)
3. [Moq — Mocking](#3-moq--mocking)
4. [FluentAssertions](#4-fluentassertions)
5. [Integration Testing in ASP.NET Core](#5-integration-testing-in-aspnet-core)
6. [Testing EF Core](#6-testing-ef-core)
7. [Test-Driven Development (TDD)](#7-test-driven-development-tdd)
8. [Common Interview Questions](#8-common-interview-questions)

---

## 1. Testing Pyramid

```
        /\
       /E2E\          ← End-to-End (few, slow, expensive)
      /------\        Selenium, Playwright
     /  Integ \       ← Integration (moderate)
    /----------\      WebApplicationFactory, TestContainers
   /   Unit    \      ← Unit (many, fast, cheap)
  /____________\      xUnit, NUnit, MSTest + Moq
```

| Layer | Speed | Cost | Coverage | Tools |
|-------|-------|------|----------|-------|
| Unit | Fast (ms) | Cheap | Business logic | xUnit, Moq |
| Integration | Medium (seconds) | Moderate | Multiple layers | WebApplicationFactory, EF InMemory |
| E2E | Slow (minutes) | Expensive | User flows | Playwright, Selenium |

---

## 2. xUnit — Unit Testing

### AAA Pattern (Arrange, Act, Assert)

```csharp
using Xunit;
using FluentAssertions;

public class BudgetItemTests
{
    [Fact]  // single test case
    public void AddExpense_WhenAmountExceedsLimit_ShouldMarkAsOverBudget()
    {
        // ARRANGE — set up test data
        var budgetItem = new BudgetItem
        {
            AllocatedAmount = 5000m,
            SpentAmount = 0m
        };

        // ACT — execute the code under test
        budgetItem.AddExpense(6000m);

        // ASSERT — verify the expected outcome
        budgetItem.IsOverBudget.Should().BeTrue();
        budgetItem.SpentAmount.Should().Be(6000m);
    }

    [Theory]  // multiple data-driven test cases
    [InlineData(1000, 5000, false)]
    [InlineData(5000, 5000, false)]
    [InlineData(5001, 5000, true)]
    [InlineData(10000, 5000, true)]
    public void IsOverBudget_ShouldReturnCorrectResult(
        decimal spent, decimal allocated, bool expectedOverBudget)
    {
        // Arrange
        var item = new BudgetItem { AllocatedAmount = allocated, SpentAmount = spent };

        // Act & Assert
        item.IsOverBudget.Should().Be(expectedOverBudget);
    }

    [Fact]
    public void AddExpense_WhenAmountIsNegative_ShouldThrowDomainException()
    {
        // Arrange
        var item = new BudgetItem { AllocatedAmount = 5000m };

        // Act & Assert — testing that exceptions are thrown
        var act = () => item.AddExpense(-100m);
        act.Should().Throw<DomainException>()
           .WithMessage("*positive*");
    }
}
```

### Test Class Fixtures & Shared Setup

```csharp
// IClassFixture — shared, expensive setup across all tests in a class
public class DatabaseFixture : IDisposable
{
    public ApplicationDbContext Context { get; }

    public DatabaseFixture()
    {
        var options = new DbContextOptionsBuilder<ApplicationDbContext>()
            .UseInMemoryDatabase(databaseName: "TestDb_" + Guid.NewGuid())
            .Options;
        Context = new ApplicationDbContext(options);
        Context.Database.EnsureCreated();
        SeedTestData(Context);
    }

    private void SeedTestData(ApplicationDbContext ctx)
    {
        ctx.Categories.AddRange(
            new Category { Id = Guid.NewGuid(), Name = "Food", Type = CategoryType.Expense },
            new Category { Id = Guid.NewGuid(), Name = "Salary", Type = CategoryType.Income }
        );
        ctx.SaveChanges();
    }

    public void Dispose() => Context.Dispose();
}

public class CategoryTests : IClassFixture<DatabaseFixture>
{
    private readonly DatabaseFixture _fixture;
    public CategoryTests(DatabaseFixture fixture) => _fixture = fixture;

    [Fact]
    public void Categories_ShouldHaveAtLeastTwoCategories()
    {
        _fixture.Context.Categories.Count().Should().BeGreaterThan(1);
    }
}

// IAsyncLifetime — async setup and teardown
public class AsyncSetupTests : IAsyncLifetime
{
    public async Task InitializeAsync()
    {
        // async setup
        await SeedDatabaseAsync();
    }

    public async Task DisposeAsync()
    {
        // async cleanup
        await CleanupAsync();
    }
}
```

---

## 3. Moq — Mocking

### Test Doubles

| Type | Description | Example |
|---|---|---|
| **Dummy** | Not used, just fills parameters | `new CancellationToken()` |
| **Stub** | Returns pre-defined data | `_repo.Setup(r => r.GetAll()).Returns(fakeList)` |
| **Mock** | Verifies interactions were made | `_emailService.Verify(e => e.Send(...), Times.Once)` |
| **Fake** | Simplified working implementation | In-memory repository |
| **Spy** | Like a mock but wraps the real implementation | `Mock<T>` with `CallBase = true` |

### Basic Moq Usage

```csharp
using Moq;

public class TransactionServiceTests
{
    private readonly Mock<ITransactionRepository> _mockRepo;
    private readonly Mock<INotificationService> _mockNotifier;
    private readonly TransactionService _service;

    public TransactionServiceTests()
    {
        _mockRepo = new Mock<ITransactionRepository>();
        _mockNotifier = new Mock<INotificationService>();
        _service = new TransactionService(_mockRepo.Object, _mockNotifier.Object);
    }

    [Fact]
    public async Task CreateAsync_ShouldSaveTransactionAndReturnId()
    {
        // Arrange
        var command = new CreateTransactionCommand
        {
            AccountId = Guid.NewGuid(),
            Amount = 5000m,
            Description = "Salary",
            TransactionType = TransactionType.Income
        };

        _mockRepo
            .Setup(r => r.AddAsync(It.IsAny<Transaction>(), It.IsAny<CancellationToken>()))
            .Returns(Task.CompletedTask);

        _mockRepo
            .Setup(r => r.SaveChangesAsync(It.IsAny<CancellationToken>()))
            .ReturnsAsync(1);

        // Act
        var id = await _service.CreateAsync(command, CancellationToken.None);

        // Assert
        id.Should().NotBeEmpty();
        _mockRepo.Verify(r => r.AddAsync(
            It.Is<Transaction>(t => t.Amount == 5000m && t.Description == "Salary"),
            It.IsAny<CancellationToken>()), Times.Once);
    }

    [Fact]
    public async Task CreateAsync_WhenAccountNotFound_ShouldThrowNotFoundException()
    {
        // Arrange
        _mockRepo
            .Setup(r => r.GetAccountAsync(It.IsAny<Guid>(), It.IsAny<CancellationToken>()))
            .ReturnsAsync((Account?)null); // returns null

        // Act & Assert
        var act = () => _service.CreateAsync(new CreateTransactionCommand(), CancellationToken.None);
        await act.Should().ThrowAsync<NotFoundException>()
            .WithMessage("*Account*");
    }

    [Fact]
    public async Task CreateExpense_WhenBudgetExceeded_ShouldSendNotification()
    {
        // Arrange — set up scenario where budget will be exceeded
        _mockRepo.Setup(r => r.GetCurrentBudgetSpentAsync(It.IsAny<Guid>(), It.IsAny<Guid>()))
            .ReturnsAsync(4500m); // already at 4500 of 5000 limit

        var command = new CreateTransactionCommand
            { Amount = 600m, TransactionType = TransactionType.Expense };

        // Act
        await _service.CreateAsync(command, CancellationToken.None);

        // Assert notification was sent
        _mockNotifier.Verify(n => n.SendAsync(
            It.Is<UserNotification>(notif => notif.NotificationType == NotificationType.BudgetExceeded),
            It.IsAny<CancellationToken>()), Times.Once);
    }
}
```

### Moq Advanced Patterns

```csharp
// Setup sequential returns
var mock = new Mock<IRandomService>();
mock.SetupSequence(s => s.GetNext())
    .Returns(1)
    .Returns(2)
    .Returns(3)
    .Throws(new InvalidOperationException("No more values"));

// Callback — capture arguments
Guid capturedId = Guid.Empty;
mock.Setup(r => r.AddAsync(It.IsAny<Transaction>(), It.IsAny<CancellationToken>()))
    .Callback<Transaction, CancellationToken>((t, ct) => capturedId = t.Id)
    .Returns(Task.CompletedTask);

// Verify argument matchers
mock.Verify(r => r.AddAsync(
    It.Is<Transaction>(t => t.Amount > 0 && t.UserId != Guid.Empty),
    It.IsAny<CancellationToken>()), Times.Once);

// Times enum
mock.Verify(r => r.SaveChangesAsync(It.IsAny<CancellationToken>()), Times.Exactly(2));
mock.Verify(r => r.NotifyAsync(It.IsAny<string>()), Times.Never);
mock.Verify(r => r.LogAsync(It.IsAny<string>()), Times.AtLeast(1));
```

---

## 4. FluentAssertions

Makes assertions more readable and provides better failure messages:

```csharp
// Basic
result.Should().BeTrue();
result.Should().Be(42);
result.Should().NotBeNull();
result.Should().BeNull();

// Strings
name.Should().Be("John");
name.Should().StartWith("J").And.EndWith("n");
name.Should().Contain("oh");
name.Should().Match("J*n"); // wildcard
name.Should().HaveLength(4);

// Numbers
amount.Should().BeGreaterThan(0);
amount.Should().BeInRange(100, 10_000);
amount.Should().BeApproximately(9.99m, 0.01m);

// Collections
accounts.Should().HaveCount(3);
accounts.Should().NotBeEmpty();
accounts.Should().Contain(a => a.IsActive);
accounts.Should().AllSatisfy(a => a.Balance.Should().BeGreaterThanOrEqualTo(0));
accounts.Should().BeInAscendingOrder(a => a.Name);
accounts.Should().ContainSingle(a => a.AccountType == AccountType.CreditCard);

// Exceptions
var act = () => service.Delete(nonExistentId);
act.Should().Throw<NotFoundException>();

var asyncAct = async () => await service.DeleteAsync(id, ct);
await asyncAct.Should().ThrowAsync<NotFoundException>()
    .WithMessage("*not found*");

// Object properties
account.Should().BeEquivalentTo(expectedAccount, options =>
    options.Excluding(a => a.CreatedAt).Excluding(a => a.UpdatedAt));
```

---

## 5. Integration Testing in ASP.NET Core

```csharp
using Microsoft.AspNetCore.Mvc.Testing;
using System.Net.Http.Json;

// Custom WebApplicationFactory with test configuration
public class BudgetPhWebApplicationFactory : WebApplicationFactory<Program>
{
    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.ConfigureTestServices(services =>
        {
            // Replace real DB with in-memory
            var descriptor = services.SingleOrDefault(
                d => d.ServiceType == typeof(DbContextOptions<ApplicationDbContext>));
            if (descriptor != null) services.Remove(descriptor);

            services.AddDbContext<ApplicationDbContext>(options =>
                options.UseInMemoryDatabase("IntegrationTestDb"));

            // Replace real email service with fake
            services.AddScoped<IEmailService, FakeEmailService>();
        });
    }
}

public class AuthControllerIntegrationTests 
    : IClassFixture<BudgetPhWebApplicationFactory>
{
    private readonly HttpClient _client;

    public AuthControllerIntegrationTests(BudgetPhWebApplicationFactory factory)
    {
        _client = factory.CreateClient();
    }

    [Fact]
    public async Task Register_WithValidData_ShouldReturn200AndToken()
    {
        // Arrange
        var request = new
        {
            Email = "test@budgetph.com",
            Password = "SecurePass123!",
            FirstName = "Test",
            LastName = "User"
        };

        // Act
        var response = await _client.PostAsJsonAsync("/api/auth/register", request);

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.OK);
        var content = await response.Content.ReadFromJsonAsync<AuthResponse>();
        content!.AccessToken.Should().NotBeNullOrEmpty();
    }

    [Fact]
    public async Task GetAccounts_WithoutToken_ShouldReturn401()
    {
        var response = await _client.GetAsync("/api/accounts");
        response.StatusCode.Should().Be(HttpStatusCode.Unauthorized);
    }

    [Fact]
    public async Task GetAccounts_WithValidToken_ShouldReturn200()
    {
        // First login to get token
        var loginResponse = await _client.PostAsJsonAsync("/api/auth/login", 
            new { Email = "test@budgetph.com", Password = "SecurePass123!" });
        var auth = await loginResponse.Content.ReadFromJsonAsync<AuthResponse>();

        // Set authorization header
        _client.DefaultRequestHeaders.Authorization = 
            new AuthenticationHeaderValue("Bearer", auth!.AccessToken);

        // Act
        var response = await _client.GetAsync("/api/accounts");

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.OK);
    }
}
```

---

## 6. Testing EF Core

```csharp
// Option 1: In-Memory Database (simple, fast, no SQL semantics)
private ApplicationDbContext CreateInMemoryContext()
{
    var options = new DbContextOptionsBuilder<ApplicationDbContext>()
        .UseInMemoryDatabase(databaseName: Guid.NewGuid().ToString())
        .Options;
    
    var context = new ApplicationDbContext(options);
    context.Database.EnsureCreated();
    return context;
}

// Option 2: SQLite (better SQL compatibility, still fast)
private ApplicationDbContext CreateSqliteContext()
{
    var connection = new SqliteConnection("DataSource=:memory:");
    connection.Open();
    
    var options = new DbContextOptionsBuilder<ApplicationDbContext>()
        .UseSqlite(connection)
        .Options;
    
    var context = new ApplicationDbContext(options);
    context.Database.EnsureCreated();
    return context;
}

// Test using real context with SQLite
public class AccountRepositoryTests : IDisposable
{
    private readonly ApplicationDbContext _context;
    private readonly AccountRepository _repository;

    public AccountRepositoryTests()
    {
        _context = CreateSqliteContext();
        _repository = new AccountRepository(_context);
    }

    [Fact]
    public async Task GetByUserIdAsync_ShouldReturnOnlyUserAccounts()
    {
        // Arrange
        var userId1 = Guid.NewGuid();
        var userId2 = Guid.NewGuid();
        
        _context.Accounts.AddRange(
            new Account { Id = Guid.NewGuid(), UserId = userId1, Name = "BPI" },
            new Account { Id = Guid.NewGuid(), UserId = userId1, Name = "BDO" },
            new Account { Id = Guid.NewGuid(), UserId = userId2, Name = "Metrobank" }
        );
        await _context.SaveChangesAsync();

        // Act
        var accounts = await _repository.GetByUserIdAsync(userId1, CancellationToken.None);

        // Assert
        accounts.Should().HaveCount(2);
        accounts.Should().AllSatisfy(a => a.UserId.Should().Be(userId1));
        accounts.Should().NotContain(a => a.Name == "Metrobank");
    }

    public void Dispose() => _context.Dispose();
}
```

---

## 7. Test-Driven Development (TDD)

TDD cycle: **Red → Green → Refactor**

1. **Red**: Write a failing test
2. **Green**: Write minimal code to make it pass
3. **Refactor**: Clean up without breaking tests

```csharp
// Step 1: RED — write failing test
[Fact]
public void LoanAmortization_Calculate_ShouldReturnCorrectMonthlyPayment()
{
    // Arrange
    var calculator = new LoanAmortizationCalculator();
    
    // Act
    var payment = calculator.CalculateMonthlyPayment(
        principal: 500_000m, annualRate: 0.12m, months: 60);
    
    // Assert
    payment.Should().BeApproximately(11_122.22m, 1m); // fails — class doesn't exist yet
}

// Step 2: GREEN — minimal implementation
public class LoanAmortizationCalculator
{
    public decimal CalculateMonthlyPayment(decimal principal, decimal annualRate, int months)
    {
        var monthlyRate = annualRate / 12;
        var factor = (decimal)Math.Pow((double)(1 + monthlyRate), months);
        return principal * monthlyRate * factor / (factor - 1);
    }
}

// Step 3: REFACTOR — improve without breaking tests
public class LoanAmortizationCalculator
{
    public LoanPaymentResult Calculate(LoanParameters parameters)
    {
        Guard.Against.NegativeOrZero(parameters.Principal, nameof(parameters.Principal));
        Guard.Against.OutOfRange(parameters.AnnualRate, nameof(parameters.AnnualRate), 0.001m, 0.99m);
        
        var monthlyPayment = CalculateMonthlyPayment(
            parameters.Principal, parameters.AnnualRate, parameters.Months);
        
        return new LoanPaymentResult(monthlyPayment, BuildAmortizationSchedule(parameters));
    }
}
```

---

## 8. Common Interview Questions

**Q1: What is the difference between a Mock and a Stub?**

> A **Stub** provides pre-defined answers to calls made during the test — it doesn't care if/how often it was called. A **Mock** has expectations built in — it verifies that specific calls were made (and possibly with specific arguments). You typically use stubs for queries (getting data back) and mocks for commands (verifying actions were taken).

**Q2: What is the testing pyramid and why does it matter?**

> The pyramid suggests having many unit tests, fewer integration tests, and even fewer E2E tests. Unit tests run in milliseconds, are cheap to write, and give fast feedback. E2E tests are slow, fragile, and expensive. An "ice cream cone" anti-pattern (mostly E2E, few unit tests) leads to slow, brittle test suites.

**Q3: What is the difference between `[Fact]` and `[Theory]` in xUnit?**

> `[Fact]` — a single test with no parameters. `[Theory]` — a parameterized test with multiple data sets. `[Theory]` is paired with `[InlineData(...)`, `[MemberData(...)]`, or `[ClassData(...)]` to provide test cases.

**Q4: How do you test code that depends on `DateTime.Now`?**

> Inject an `IDateTimeProvider` interface instead of calling `DateTime.Now` directly. In production, use `SystemDateTimeProvider` (returns `DateTime.Now`). In tests, use `FakeDateTimeProvider` (returns a fixed date). This makes time-dependent tests deterministic.

**Q5: What is the difference between integration tests and unit tests?**

> Unit tests test a **single unit** (class/method) in isolation with all dependencies mocked. They run fast and pinpoint failures precisely. Integration tests test **multiple components together** (e.g., API + database + middleware) — they test that components work together correctly but run slower and are harder to diagnose.

**Q6: Why should you avoid `Thread.Sleep` in tests?**

> `Thread.Sleep` makes tests slow and non-deterministic (the sleep may be too short on a slow machine). Instead: use `Task.Delay` with proper async test patterns, mock time-based operations, or use `AsyncManualResetEvent`/`SemaphoreSlim` for synchronization in concurrent tests.
