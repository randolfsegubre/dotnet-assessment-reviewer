# Topic 19 — SignalR & Real-Time Web

> SignalR is the .NET real-time communication library. Essential for senior .NET developers building live dashboards, chat, notifications, and collaborative tools.

---

## Real-Time Options Comparison

| Technology | Protocol | Server Push? | Best for |
|---|---|---|---|
| **SignalR** | WebSocket / SSE / Long Polling | ✅ Yes | .NET real-time apps |
| **WebSocket** | WebSocket (raw) | ✅ Yes | Custom, non-.NET |
| **SSE** (Server-Sent Events) | HTTP/1.1 | ✅ One-way | Notifications, live feeds |
| **Long Polling** | HTTP | ✅ Simulated | Fallback |
| **Short Polling** | HTTP | ❌ Client pulls | Simple status checks |

SignalR **automatically negotiates** the best transport: WebSocket first, then SSE, then Long Polling.

---

## Hub (Server-Side)

```csharp
// Hubs/NotificationHub.cs:
[Authorize]
public class NotificationHub : Hub
{
    private readonly ILogger<NotificationHub> _logger;
    private readonly IConnectionManager _connectionManager;

    public NotificationHub(ILogger<NotificationHub> logger, IConnectionManager cm)
    {
        _logger = logger;
        _connectionManager = cm;
    }

    // Called when client connects:
    public override async Task OnConnectedAsync()
    {
        var userId = Context.UserIdentifier!;
        await Groups.AddToGroupAsync(Context.ConnectionId, $"user:{userId}");
        _logger.LogInformation("User {UserId} connected: {ConnId}", userId, Context.ConnectionId);
        await base.OnConnectedAsync();
    }

    // Called when client disconnects:
    public override async Task OnDisconnectedAsync(Exception? exception)
    {
        var userId = Context.UserIdentifier!;
        _logger.LogInformation("User {UserId} disconnected", userId);
        await base.OnDisconnectedAsync(exception);
    }

    // Hub method — clients can CALL this:
    public async Task JoinAccountGroup(string accountId)
    {
        // Verify user owns this account before joining
        await Groups.AddToGroupAsync(Context.ConnectionId, $"account:{accountId}");
    }

    public async Task LeaveAccountGroup(string accountId)
    {
        await Groups.RemoveFromGroupAsync(Context.ConnectionId, $"account:{accountId}");
    }

    // Receive a message from client and broadcast:
    public async Task SendBudgetAlert(string message)
    {
        await Clients.Group($"user:{Context.UserIdentifier}").SendAsync("ReceiveBudgetAlert", message);
    }
}
```

---

## Sending from Outside a Hub (Services)

```csharp
// Inject IHubContext to push notifications from controllers/services:
public class TransactionService
{
    private readonly IHubContext<NotificationHub> _hub;
    private readonly AppDbContext _db;

    public TransactionService(IHubContext<NotificationHub> hub, AppDbContext db)
    {
        _hub = hub;
        _db = db;
    }

    public async Task CreateTransactionAsync(CreateTransactionCommand cmd, CancellationToken ct)
    {
        var tx = new Transaction { Amount = cmd.Amount, /* ... */ };
        _db.Transactions.Add(tx);
        await _db.SaveChangesAsync(ct);

        // Push real-time update to the user who created it:
        await _hub.Clients
            .User(cmd.UserId.ToString())        // specific user
            .SendAsync("TransactionCreated", new
            {
                tx.Id, tx.Amount, tx.Description, tx.Date
            }, ct);

        // Check budget and alert if exceeded:
        if (await IsBudgetExceededAsync(cmd.UserId, cmd.CategoryId, ct))
        {
            await _hub.Clients
                .User(cmd.UserId.ToString())
                .SendAsync("BudgetExceeded", new { CategoryId = cmd.CategoryId }, ct);
        }
    }
}

// Target specific groups:
await _hub.Clients.Group("account:acc-001").SendAsync("BalanceUpdated", newBalance);
await _hub.Clients.All.SendAsync("SystemAlert", "Maintenance in 5 minutes");
await _hub.Clients.AllExcept(excludedConnectionIds).SendAsync("Update", data);
```

---

## Registration (Program.cs)

```csharp
// Add SignalR:
builder.Services.AddSignalR(options =>
{
    options.EnableDetailedErrors = builder.Environment.IsDevelopment();
    options.KeepAliveInterval = TimeSpan.FromSeconds(15);
    options.ClientTimeoutInterval = TimeSpan.FromSeconds(60);
    options.MaximumReceiveMessageSize = 32 * 1024; // 32KB
});

// Map hub endpoint:
app.MapHub<NotificationHub>("/hubs/notifications");

// For scale-out (multiple servers) — use Redis backplane:
builder.Services.AddSignalR().AddStackExchangeRedis("redis-connection-string");
```

---

## Client-Side (JavaScript / React)

```typescript
// npm install @microsoft/signalr

import * as signalR from '@microsoft/signalr'

class NotificationService {
    private connection: signalR.HubConnection

    constructor(token: string) {
        this.connection = new signalR.HubConnectionBuilder()
            .withUrl('/hubs/notifications', {
                accessTokenFactory: () => token
            })
            .withAutomaticReconnect([0, 2000, 5000, 10000, 30000]) // retry intervals
            .configureLogging(signalR.LogLevel.Warning)
            .build()

        // Listen for server events:
        this.connection.on('TransactionCreated', (tx) => {
            console.log('New transaction:', tx)
            queryClient.invalidateQueries({ queryKey: ['transactions'] }) // refresh TanStack Query
        })

        this.connection.on('BudgetExceeded', ({ categoryId }) => {
            toast.warning(`Budget exceeded for category ${categoryId}`)
        })

        this.connection.on('BalanceUpdated', (newBalance) => {
            setBalance(newBalance)
        })

        // Connection state events:
        this.connection.onreconnecting(() => showToast('Reconnecting...'))
        this.connection.onreconnected(() => showToast('Reconnected!'))
        this.connection.onclose(() => showToast('Disconnected'))
    }

    async start() {
        try {
            await this.connection.start()
            console.log('SignalR connected')
        } catch (err) {
            console.error('SignalR connection failed:', err)
        }
    }

    async stop() {
        await this.connection.stop()
    }

    // Call server hub methods:
    async joinAccountGroup(accountId: string) {
        await this.connection.invoke('JoinAccountGroup', accountId)
    }
}

// React hook:
export function useSignalR() {
    const { token } = useAuthStore()
    const service = useRef<NotificationService>()

    useEffect(() => {
        if (!token) return
        service.current = new NotificationService(token)
        service.current.start()
        return () => { service.current?.stop() }
    }, [token])
}
```

---

## Streaming (Server → Client)

```csharp
// Hub with streaming:
public class ReportsHub : Hub
{
    // Server-to-client streaming:
    public async IAsyncEnumerable<TransactionDto> StreamTransactions(
        Guid userId,
        [EnumeratorCancellation] CancellationToken ct)
    {
        var transactions = _db.Transactions
            .Where(t => t.UserId == userId)
            .AsAsyncEnumerable();

        await foreach (var tx in transactions.WithCancellation(ct))
        {
            yield return _mapper.Map<TransactionDto>(tx);
            await Task.Delay(10, ct); // throttle if needed
        }
    }
}
```

```typescript
// Client consuming stream:
const stream = connection.stream('StreamTransactions', userId)
stream.subscribe({
    next: (tx) => addTransaction(tx),
    complete: () => setLoading(false),
    error: (err) => console.error(err)
})
```

---

## Interview Q&A

**Q1: What transports does SignalR support and how does it choose?**
> SignalR negotiates in order: WebSocket (best, full-duplex) → Server-Sent Events (one-way server push) → Long Polling (fallback). WebSocket requires the browser and server to both support it and no proxies blocking it. SignalR handles the negotiation automatically.

**Q2: What is the difference between `Clients.User()` and `Clients.Group()`?**
> `Clients.User(userId)` sends to all connections of a specific user (multiple tabs/devices). It uses the `ClaimTypes.NameIdentifier` claim from JWT for identification — requires `AddAuthentication()` configured. `Clients.Group(groupName)` sends to all connections in a named group — you manage group membership manually with `Groups.AddToGroupAsync()`.

**Q3: How do you scale SignalR across multiple servers?**
> By default, SignalR uses in-memory routing — connections on Server A can't receive messages sent from Server B. For scale-out: add a **Redis backplane** (`AddStackExchangeRedis`) or **Azure SignalR Service** (`AddAzureSignalR`). These act as a message bus between server instances.

**Q4: How do you authenticate SignalR connections?**
> JWT tokens can't be sent as headers in WebSocket connections (browser limitation). SignalR provides `accessTokenFactory` on the client to send the token as a query parameter (`?access_token=...`). On the server, configure JWT to read from query string for SignalR paths using `options.Events.OnMessageReceived`.
