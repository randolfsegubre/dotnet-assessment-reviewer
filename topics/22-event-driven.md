# Topic 22 — Event-Driven Architecture

> Event-driven systems decouple producers from consumers. Core senior-level topic for distributed systems, microservices, and scalable .NET backends.

---

## Core Concepts

```
Event     — Something that happened: "TransactionCreated", "BudgetExceeded"
Command   — Request to do something: "CreateTransaction", "SendEmail"
Message   — Generic term (event or command transmitted over a bus)
Producer  — Publishes events (doesn't know who consumes)
Consumer  — Subscribes to events (doesn't know who produced)
Message Broker — Middleman: RabbitMQ, Azure Service Bus, Kafka
```

| Pattern | Direction | Who knows whom? |
|---|---|---|
| **Request/Response** | Synchronous | Caller knows callee |
| **Events (pub/sub)** | Async, one→many | Nobody knows anybody |
| **Commands** | Async, one→one | Producer knows queue name |

---

## Domain Events (In-Process)

```csharp
// Domain event (in Domain layer):
public record TransactionCreatedEvent(
    Guid TransactionId,
    Guid UserId,
    Guid AccountId,
    decimal Amount,
    TransactionType Type,
    DateTime OccurredAt) : IDomainEvent;

// Base entity raises events:
public abstract class BaseEntity
{
    private readonly List<IDomainEvent> _events = [];
    public IReadOnlyList<IDomainEvent> DomainEvents => _events.AsReadOnly();
    
    protected void RaiseDomainEvent(IDomainEvent domainEvent) => _events.Add(domainEvent);
    public void ClearDomainEvents() => _events.Clear();
}

// Transaction raises event when created:
public class Transaction : BaseEntity
{
    public static Transaction Create(Guid accountId, Guid userId, decimal amount, TransactionType type)
    {
        var tx = new Transaction { AccountId = accountId, UserId = userId, Amount = amount, Type = type };
        tx.RaiseDomainEvent(new TransactionCreatedEvent(tx.Id, userId, accountId, amount, type, DateTime.UtcNow));
        return tx;
    }
}

// Dispatch after SaveChanges (Infrastructure):
public class AppDbContext : DbContext
{
    private readonly IPublisher _publisher;
    
    public override async Task<int> SaveChangesAsync(CancellationToken ct = default)
    {
        var result = await base.SaveChangesAsync(ct);
        
        // Dispatch domain events after successful save:
        var events = ChangeTracker.Entries<BaseEntity>()
            .SelectMany(e => e.Entity.DomainEvents)
            .ToList();
        
        foreach (var domainEvent in events)
            await _publisher.Publish(domainEvent, ct);
        
        ChangeTracker.Entries<BaseEntity>()
            .ToList()
            .ForEach(e => e.Entity.ClearDomainEvents());
        
        return result;
    }
}

// Handler (in Application layer):
public class SendBudgetAlertOnTransactionCreated
    : INotificationHandler<TransactionCreatedEvent>
{
    public async Task Handle(TransactionCreatedEvent notification, CancellationToken ct)
    {
        // Check if this transaction exceeds the budget for its category
        await _budgetService.CheckAndAlertAsync(notification.UserId, notification.AccountId, ct);
    }
}
```

---

## Integration Events (Cross-Service)

```csharp
// Integration event — crosses service boundaries via message broker:
public record TransactionCreatedIntegrationEvent(
    Guid TransactionId,
    string UserId,
    decimal Amount,
    string Type,
    string Currency,
    DateTime CreatedAt);

// Outbox Pattern — prevent lost messages:
// 1. Save transaction + outbox message in same DB transaction
// 2. Background worker reads outbox → publishes to broker → marks as sent
public class TransactionService
{
    public async Task CreateAsync(CreateTransactionCommand cmd, CancellationToken ct)
    {
        await using var dbTx = await _db.Database.BeginTransactionAsync(ct);
        try
        {
            var tx = Transaction.Create(cmd.AccountId, cmd.UserId, cmd.Amount, cmd.Type);
            _db.Transactions.Add(tx);
            
            // Save outbox message in same transaction — atomic!
            _db.OutboxMessages.Add(new OutboxMessage
            {
                Id = Guid.NewGuid(),
                Type = nameof(TransactionCreatedIntegrationEvent),
                Payload = JsonSerializer.Serialize(new TransactionCreatedIntegrationEvent(
                    tx.Id, cmd.UserId.ToString(), tx.Amount, tx.Type.ToString(), "PHP", DateTime.UtcNow)),
                CreatedAt = DateTime.UtcNow
            });
            
            await _db.SaveChangesAsync(ct);
            await dbTx.CommitAsync(ct);
        }
        catch
        {
            await dbTx.RollbackAsync(ct);
            throw;
        }
    }
}

// Outbox processor (background service reads and publishes):
public class OutboxProcessor : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        while (!ct.IsCancellationRequested)
        {
            var pending = await _db.OutboxMessages
                .Where(m => m.ProcessedAt == null)
                .OrderBy(m => m.CreatedAt)
                .Take(50)
                .ToListAsync(ct);
            
            foreach (var message in pending)
            {
                await _messageBus.PublishAsync(message.Type, message.Payload, ct);
                message.ProcessedAt = DateTime.UtcNow;
            }
            
            await _db.SaveChangesAsync(ct);
            await Task.Delay(TimeSpan.FromSeconds(5), ct);
        }
    }
}
```

---

## MassTransit + RabbitMQ

```csharp
// Install: MassTransit.RabbitMQ

// Program.cs:
builder.Services.AddMassTransit(x =>
{
    // Register consumers:
    x.AddConsumer<TransactionCreatedConsumer>();
    x.AddConsumer<BudgetExceededConsumer>();
    
    x.UsingRabbitMq((context, cfg) =>
    {
        cfg.Host("rabbitmq://localhost", h =>
        {
            h.Username("guest");
            h.Password("guest");
        });
        
        cfg.ConfigureEndpoints(context); // auto-configure based on consumers
    });
});

// Publisher:
public class TransactionCreatedPublisher
{
    private readonly IBus _bus;
    
    public async Task PublishAsync(TransactionCreatedIntegrationEvent evt, CancellationToken ct)
    {
        await _bus.Publish(evt, ct);
    }
}

// Consumer:
public class TransactionCreatedConsumer : IConsumer<TransactionCreatedIntegrationEvent>
{
    private readonly INotificationService _notifications;
    
    public async Task Consume(ConsumeContext<TransactionCreatedIntegrationEvent> context)
    {
        var evt = context.Message;
        
        // Update analytics
        await _analytics.RecordTransactionAsync(evt);
        
        // Send push notification
        await _notifications.SendAsync(evt.UserId, $"Transaction ₱{evt.Amount:N0} recorded");
    }
}
```

---

## Saga Pattern — Distributed Transactions

```csharp
// Orchestration Saga via MassTransit StateMachine:
// Scenario: CreateTransaction → UpdateBalance → SendNotification → Complete
// If UpdateBalance fails → CompensateTransaction → Fail

public class TransactionSaga : MassTransitStateMachine<TransactionSagaState>
{
    public State Creating { get; private set; } = null!;
    public State UpdatingBalance { get; private set; } = null!;
    public State Completed { get; private set; } = null!;
    public State Failed { get; private set; } = null!;

    public Event<StartTransactionCommand> Start { get; private set; } = null!;
    public Event<BalanceUpdated> BalanceUpdated { get; private set; } = null!;
    public Event<BalanceUpdateFailed> BalanceUpdateFailed { get; private set; } = null!;

    public TransactionSaga()
    {
        InstanceState(x => x.CurrentState);

        Initially(
            When(Start)
                .Then(ctx => ctx.Saga.TransactionId = ctx.Message.TransactionId)
                .TransitionTo(Creating)
                .Publish(ctx => new CreateTransactionCommand(ctx.Message.TransactionId))
        );

        During(Creating,
            When(BalanceUpdated)
                .TransitionTo(Completed)
                .Publish(ctx => new SendNotificationCommand(ctx.Saga.TransactionId)),
            When(BalanceUpdateFailed)
                .TransitionTo(Failed)
                .Publish(ctx => new CompensateTransactionCommand(ctx.Saga.TransactionId))
        );
    }
}
```

---

## Event Sourcing

```csharp
// Store events, not state — state is rebuilt by replaying events:
public abstract record DomainEvent(Guid StreamId, int Version, DateTime OccurredAt);
public record AccountOpenedEvent(Guid StreamId, int Version, DateTime OccurredAt, string Name, string Currency)
    : DomainEvent(StreamId, Version, OccurredAt);
public record TransactionPostedEvent(Guid StreamId, int Version, DateTime OccurredAt, decimal Amount, string Type)
    : DomainEvent(StreamId, Version, OccurredAt);

// Aggregate rebuilt from events:
public class Account
{
    public Guid Id { get; private set; }
    public string Name { get; private set; } = "";
    public decimal Balance { get; private set; }
    private int _version = 0;

    public static Account Rehydrate(IEnumerable<DomainEvent> events)
    {
        var account = new Account();
        foreach (var evt in events) account.Apply(evt);
        return account;
    }

    private void Apply(DomainEvent evt)
    {
        switch (evt)
        {
            case AccountOpenedEvent e:
                Id = e.StreamId; Name = e.Name; break;
            case TransactionPostedEvent e:
                Balance += e.Type == "income" ? e.Amount : -e.Amount; break;
        }
        _version = evt.Version;
    }
}

// Event store writes events, projections build read models:
// EventStore → Event Stream → Projector → ReadModel (SQL/Redis)
```

---

## Interview Q&A

**Q1: What is the Outbox Pattern and why is it needed?**
> Without the Outbox, you face a dual-write problem: if you save to DB and publish to the message broker, one can fail — data becomes inconsistent. The Outbox stores the message in the same DB transaction as the business data. A background process then reads unpublished messages and publishes them, guaranteeing at-least-once delivery.

**Q2: What is the difference between a queue and a topic?**
> A queue is point-to-point — one message goes to one consumer. A topic is publish/subscribe — one message goes to multiple subscribers (via subscriptions). Azure Service Bus supports both. RabbitMQ uses exchanges + routing keys to achieve the same.

**Q3: What is the difference between Domain Events and Integration Events?**
> Domain events are in-process (within one service) raised by domain entities when something significant happens — dispatched synchronously via MediatR before or after `SaveChanges`. Integration events cross service boundaries via a message broker — async, eventual consistency, used for inter-service communication.

**Q4: What is Event Sourcing and when would you use it?**
> Event Sourcing stores every state change as an immutable event, not the current state. State is reconstructed by replaying events. Benefits: full audit trail, time-travel debugging, easy projections into different read models. Costs: complexity, eventual consistency, storage growth. Best for financial systems, audit logs, and systems where history matters.
