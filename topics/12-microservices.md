# 12 — Microservices & Architecture

---

## Table of Contents
1. [Microservices vs Monolith](#1-microservices-vs-monolith)
2. [Microservices Communication](#2-microservices-communication)
3. [Message Queues & Event-Driven Architecture](#3-message-queues--event-driven-architecture)
4. [API Gateway Pattern](#4-api-gateway-pattern)
5. [Service Discovery](#5-service-discovery)
6. [Saga Pattern](#6-saga-pattern)
7. [Circuit Breaker Pattern](#7-circuit-breaker-pattern)
8. [Docker & Containerization](#8-docker--containerization)
9. [Common Interview Questions](#9-common-interview-questions)

---

## 1. Microservices vs Monolith

### Monolith (BudgetPH is a Monolith)

```
┌──────────────────────────────┐
│         BudgetPH API          │
│  ┌────────┐  ┌────────────┐  │
│  │Accounts│  │Transactions│  │
│  └────────┘  └────────────┘  │
│  ┌────────┐  ┌────────────┐  │
│  │Budgets │  │SavingsGoals│  │
│  └────────┘  └────────────┘  │
│  ┌──────────────────────────┐ │
│  │     ApplicationDbContext  │ │
│  └──────────────────────────┘ │
└──────────────────────────────┘
                │
         SQL Server
```

**Monolith pros:**
- Simple to develop, test, deploy
- Single process — no network calls between components
- Easy transactions (single DB)

**Monolith cons:**
- Scaling: must scale entire app, not just hot components
- Team scaling: merge conflicts, tight coupling
- Technology lock-in
- Long deployment cycles as app grows

### Microservices

```
         API Gateway
         /    |    \
        /     |     \
  Accounts  Transactions  Budgets   ← separate services
    │           │           │
  DB_Acc    DB_Trans    DB_Budget   ← each has its own DB
```

**Microservices pros:**
- Independent deployment and scaling
- Technology diversity per service
- Smaller codebases, focused teams

**Microservices cons:**
- Distributed system complexity
- Network latency between services
- Distributed transactions are hard
- Operational overhead (Kubernetes, service mesh, etc.)

**Rule of thumb:** Start with a monolith. Extract services when you have a team/scaling problem, not before.

---

## 2. Microservices Communication

### Synchronous (Request/Response)

```csharp
// HTTP REST via HttpClient
public class AccountServiceClient
{
    private readonly HttpClient _httpClient;

    public AccountServiceClient(HttpClient httpClient)
    {
        _httpClient = httpClient;
    }

    public async Task<AccountDto?> GetAccountAsync(Guid accountId, CancellationToken ct)
    {
        var response = await _httpClient.GetAsync($"/api/accounts/{accountId}", ct);
        
        if (response.StatusCode == HttpStatusCode.NotFound)
            return null;
        
        response.EnsureSuccessStatusCode();
        return await response.Content.ReadFromJsonAsync<AccountDto>(ct);
    }
}

// Register typed HttpClient with Polly resilience
builder.Services.AddHttpClient<AccountServiceClient>(client =>
{
    client.BaseAddress = new Uri(builder.Configuration["Services:AccountsService:BaseUrl"]!);
    client.Timeout = TimeSpan.FromSeconds(30);
})
.AddStandardResilienceHandler(); // Polly: retry + circuit breaker + timeout
```

### gRPC (High-performance synchronous)

```protobuf
// account.proto
syntax = "proto3";
service AccountService {
    rpc GetAccount (GetAccountRequest) returns (AccountResponse);
    rpc GetAccounts (GetAccountsRequest) returns (stream AccountResponse);
}

message GetAccountRequest {
    string account_id = 1;
}
message AccountResponse {
    string id = 1;
    string name = 2;
    double balance = 3;
}
```

---

## 3. Message Queues & Event-Driven Architecture

### Why Message Queues?

```
❌ Synchronous coupling:
Transaction Service → [HTTP] → Budget Service
                    (Budget service must be up!)

✅ Async via message queue:
Transaction Service → [Message Queue] → Budget Service
                    (Budget service can be down temporarily)
```

### MassTransit + RabbitMQ (.NET)

```csharp
// Install: dotnet add package MassTransit.RabbitMQ

// Message (contract)
public record TransactionCreatedEvent
{
    public Guid TransactionId { get; init; }
    public Guid UserId { get; init; }
    public Guid? CategoryId { get; init; }
    public decimal Amount { get; init; }
    public TransactionType TransactionType { get; init; }
    public DateTime TransactionDate { get; init; }
}

// Publisher — Transaction Service publishes event
public class TransactionCommandHandler
{
    private readonly IPublishEndpoint _publishEndpoint;
    
    public async Task Handle(CreateTransactionCommand cmd, CancellationToken ct)
    {
        // ... create transaction in DB
        var transaction = new Transaction { ... };
        await _context.SaveChangesAsync(ct);
        
        // Publish event to message broker
        await _publishEndpoint.Publish(new TransactionCreatedEvent
        {
            TransactionId = transaction.Id,
            UserId = cmd.UserId,
            CategoryId = transaction.CategoryId,
            Amount = transaction.Amount,
            TransactionType = transaction.TransactionType,
            TransactionDate = transaction.TransactionDate
        }, ct);
    }
}

// Consumer — Budget Service listens and updates budget
public class TransactionCreatedConsumer : IConsumer<TransactionCreatedEvent>
{
    private readonly BudgetDbContext _budgetDb;
    
    public async Task Consume(ConsumeContext<TransactionCreatedEvent> context)
    {
        var msg = context.Message;
        if (msg.TransactionType != TransactionType.Expense || !msg.CategoryId.HasValue)
            return;
        
        var budgetItem = await _budgetDb.BudgetItems
            .FirstOrDefaultAsync(bi => bi.CategoryId == msg.CategoryId &&
                                        bi.Budget.UserId == msg.UserId &&
                                        bi.Budget.Month == msg.TransactionDate.Month);
        
        if (budgetItem != null)
        {
            budgetItem.SpentAmount += msg.Amount;
            await _budgetDb.SaveChangesAsync();
        }
    }
}

// Registration
builder.Services.AddMassTransit(x =>
{
    x.AddConsumer<TransactionCreatedConsumer>();
    
    x.UsingRabbitMq((ctx, cfg) =>
    {
        cfg.Host("rabbitmq://localhost", h =>
        {
            h.Username("guest");
            h.Password("guest");
        });
        cfg.ConfigureEndpoints(ctx);
    });
});
```

### Event Sourcing vs Traditional CRUD

| | Traditional CRUD | Event Sourcing |
|---|---|---|
| Stores | Current state | All state changes as events |
| Audit trail | Extra effort | Built-in |
| Time travel | No | ✅ Yes |
| Complexity | Low | High |
| Use when | Most apps | Financial systems, audit-required |

```csharp
// Event sourcing for account balance
public class AccountProjection
{
    public Guid Id { get; set; }
    public decimal Balance { get; set; }

    // Apply events to rebuild state
    public void Apply(MoneyDeposited e) => Balance += e.Amount;
    public void Apply(MoneyWithdrawn e) => Balance -= e.Amount;
    public void Apply(AccountOpened e) => Balance = e.InitialBalance;
}

// Rebuild from event stream
var account = new AccountProjection();
foreach (var @event in eventStore.GetEvents(accountId))
{
    account.Apply((dynamic)@event); // dynamic dispatch
}
```

---

## 4. API Gateway Pattern

```
Clients (React, Mobile) → API Gateway → Service A
                                       → Service B
                                       → Service C
```

API Gateway responsibilities:
- **Routing** — route requests to appropriate service
- **Authentication** — centralized JWT validation
- **Rate limiting** — protect downstream services
- **Load balancing** — distribute across service instances
- **SSL termination** — handle HTTPS at the gateway
- **Request/response transformation** — reshape payloads

```csharp
// YARP (Yet Another Reverse Proxy) — .NET API Gateway
builder.Services.AddReverseProxy()
    .LoadFromConfig(builder.Configuration.GetSection("ReverseProxy"));

// appsettings.json
{
  "ReverseProxy": {
    "Routes": {
      "accounts-route": {
        "ClusterId": "accounts-cluster",
        "Match": { "Path": "/api/accounts/{**catch-all}" }
      },
      "transactions-route": {
        "ClusterId": "transactions-cluster",
        "Match": { "Path": "/api/transactions/{**catch-all}" }
      }
    },
    "Clusters": {
      "accounts-cluster": {
        "Destinations": {
          "destination1": { "Address": "http://accounts-service:8080/" }
        }
      }
    }
  }
}
```

---

## 5. Service Discovery

```
❌ Hardcoded service URLs (fails when service moves):
http://192.168.1.10:8080/api/accounts

✅ Service discovery:
Client → asks registry: "Where is AccountsService?"
Registry → returns: http://pod-abc123:8080
```

Options:
- **Consul** — service registry and health checking
- **Kubernetes DNS** — automatic service discovery via DNS
- **Azure Service Fabric** — Microsoft's native discovery

---

## 6. Saga Pattern

Manages **distributed transactions** across microservices. When you can't use a single ACID transaction.

### Choreography Saga (Event-based, no central coordinator)

```
Transactions Service → publishes TransactionCreated
    Budget Service (listens) → updates budget → publishes BudgetUpdated
        NetWorth Service (listens) → updates net worth snapshot
```

If Budget Service fails: publishes `BudgetUpdateFailed` → Transactions Service compensates by rolling back.

### Orchestration Saga (Central coordinator)

```csharp
// MassTransit Saga State Machine
public class TransferStateMachine : MassTransitStateMachine<TransferState>
{
    public State Deducting { get; private set; }
    public State Crediting { get; private set; }
    public State Completed { get; private set; }
    public State Failed { get; private set; }

    public TransferStateMachine()
    {
        InstanceState(x => x.CurrentState);

        Event(() => TransferInitiated, x => x.CorrelateById(m => m.Message.TransferId));
        Event(() => DeductionCompleted, x => x.CorrelateById(m => m.Message.TransferId));

        Initially(
            When(TransferInitiated)
                .TransitionTo(Deducting)
                .Send(context => new DeductFromAccount(context.Message.FromAccountId)));

        During(Deducting,
            When(DeductionCompleted)
                .TransitionTo(Crediting)
                .Send(context => new CreditToAccount(context.Message.ToAccountId)),
            When(DeductionFailed)
                .TransitionTo(Failed));
    }
}
```

---

## 7. Circuit Breaker Pattern

Prevents cascading failures when a dependent service is down.

```
States: Closed → Open → Half-Open

Closed (normal): requests pass through, failures counted
Open (tripped):  requests fail immediately (no waiting), timer starts
Half-Open:       allow limited requests through, if succeed → Closed, if fail → Open
```

```csharp
// Polly Circuit Breaker (.NET resilience library)
builder.Services.AddHttpClient<ExchangeRateClient>()
    .AddResilienceHandler("exchange-rate-pipeline", pipeline =>
    {
        // Retry 3 times with exponential backoff
        pipeline.AddRetry(new HttpRetryStrategyOptions
        {
            MaxRetryAttempts = 3,
            Delay = TimeSpan.FromSeconds(1),
            BackoffType = DelayBackoffType.Exponential,
        });

        // Circuit breaker: open after 50% failure rate
        pipeline.AddCircuitBreaker(new HttpCircuitBreakerStrategyOptions
        {
            SamplingDuration = TimeSpan.FromSeconds(30),
            FailureRatio = 0.5,
            MinimumThroughput = 5,
            BreakDuration = TimeSpan.FromSeconds(30),
            OnOpened = args =>
            {
                Console.WriteLine("Circuit OPENED — exchange rate API is down");
                return ValueTask.CompletedTask;
            }
        });

        // Timeout per attempt
        pipeline.AddTimeout(TimeSpan.FromSeconds(10));
    });

// Fallback when circuit is open
public async Task<ExchangeRate> GetRateAsync(string from, string to, CancellationToken ct)
{
    try
    {
        return await _exchangeRateClient.GetRateAsync(from, to, ct);
    }
    catch (BrokenCircuitException)
    {
        // Fallback — use cached or default rate
        return _cache.GetLastKnownRate(from, to) ?? ExchangeRate.Default(from, to);
    }
}
```

---

## 8. Docker & Containerization

```dockerfile
# Dockerfile for BudgetPH API
FROM mcr.microsoft.com/dotnet/aspnet:10.0 AS base
WORKDIR /app
EXPOSE 8080

FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build
WORKDIR /src

# Copy and restore (layer cache optimization)
COPY ["src/Presentation/FinanceManager.API/FinanceManager.API.csproj", "src/Presentation/FinanceManager.API/"]
COPY ["src/Core/FinanceManager.Application/FinanceManager.Application.csproj", "src/Core/FinanceManager.Application/"]
COPY ["src/Core/FinanceManager.Domain/FinanceManager.Domain.csproj", "src/Core/FinanceManager.Domain/"]
COPY ["src/Infrastructure/FinanceManager.Infrastructure/FinanceManager.Infrastructure.csproj", "src/Infrastructure/FinanceManager.Infrastructure/"]
RUN dotnet restore "src/Presentation/FinanceManager.API/FinanceManager.API.csproj"

# Copy all source and build
COPY . .
RUN dotnet publish "src/Presentation/FinanceManager.API/FinanceManager.API.csproj" \
    -c Release -o /app/publish --no-restore

FROM base AS final
WORKDIR /app
COPY --from=build /app/publish .

# Run as non-root for security
USER app
ENTRYPOINT ["dotnet", "FinanceManager.API.dll"]
```

```yaml
# docker-compose.yml
version: '3.8'
services:
  api:
    build: .
    ports:
      - "5106:8080"
    environment:
      - ASPNETCORE_ENVIRONMENT=Development
      - ConnectionStrings__DefaultConnection=Server=db;Database=BudgetPH;...
    depends_on:
      db:
        condition: service_healthy

  web:
    build:
      context: ./src/Presentation/FinanceManager.Web/ClientApp
    ports:
      - "3000:80"

  db:
    image: mcr.microsoft.com/mssql/server:2022-latest
    environment:
      - SA_PASSWORD=YourStrong!Passw0rd
      - ACCEPT_EULA=Y
    healthcheck:
      test: /opt/mssql-tools/bin/sqlcmd -S localhost -U sa -P "YourStrong!Passw0rd" -Q "SELECT 1"
      interval: 10s
      retries: 5
```

---

## 9. Common Interview Questions

**Q1: When would you use microservices vs a monolith?**

> Start with a monolith unless you have specific reasons for microservices (different scaling requirements per service, multiple independent teams, different technology needs per service). Microservices add significant operational complexity. A well-structured monolith (modular monolith, Clean Architecture) can serve most applications effectively.

**Q2: How do you handle distributed transactions in microservices?**

> You don't — avoid them when possible. Options: (1) **Saga pattern** — choreography (event-driven compensation) or orchestration (central coordinator). (2) **Two-Phase Commit (2PC)** — works but adds latency and lock contention. (3) **Event Sourcing + CQRS** — events are immutable. (4) **Accept eventual consistency** — different services will be in sync *eventually*, not immediately.

**Q3: What is the difference between Choreography and Orchestration Sagas?**

> **Choreography**: no central coordinator — services react to events and emit their own events. Decentralized, but hard to track the overall workflow. **Orchestration**: a central saga orchestrator sends commands to services and tracks state. Easier to understand and debug, but creates a central point of coupling.

**Q4: What is eventual consistency?**

> In a distributed system, all nodes will eventually have the same data, but at any point in time, different nodes may have different versions. Example: After a bank transfer, the source account shows -₱5000 immediately, but the destination account might show +₱5000 a few milliseconds later (after the message is processed). Contrast with ACID transactions in a single database where all changes are immediately consistent.

**Q5: What is the Circuit Breaker pattern and why is it important?**

> Without a circuit breaker, if Service B is down, Service A keeps making HTTP calls that fail (and potentially time out after 30s each). Under load, this exhausts thread pool and connection pools — cascading failure. A circuit breaker "trips" after enough failures and immediately returns errors without making network calls. This gives Service B time to recover and prevents Service A from being dragged down.

**Q6: What is the difference between horizontal and vertical scaling?**

> **Vertical scaling** (scale up): add more CPU/RAM to existing server. Has limits, single point of failure, requires downtime. **Horizontal scaling** (scale out): add more server instances behind a load balancer. Theoretically unlimited, resilient to single instance failure. Microservices are designed for horizontal scaling — scale only the services that need it.
