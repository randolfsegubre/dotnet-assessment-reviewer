# Topic 27 — Blazor

> Blazor lets you write interactive web UIs in C# without JavaScript. Senior .NET developers should understand both hosting models, when to use each, and Blazor-specific patterns.

---

## Blazor Hosting Models Comparison

| Feature | Blazor Server | Blazor WebAssembly (WASM) | Blazor United / Auto (.NET 8+) |
|---|---|---|---|
| **Where code runs** | Server | Client browser | Both (per-component) |
| **Initial load** | Fast | Slow (download runtime) | Fast (server-rendered first) |
| **Offline support** | ❌ No | ✅ Yes (PWA) | Partial |
| **Latency** | UI action → server roundtrip | Zero (local) | Depends on mode |
| **Server resources** | High (SignalR per user) | Low (client-only) | Mixed |
| **Access to .NET APIs** | Full server access | Browser-sandboxed | Depends |
| **SEO** | ✅ (server renders) | ❌ Poor | ✅ (pre-rendered) |
| **Best for** | Internal tools, intranets | Public SPAs, mobile-like | General purpose (.NET 8+) |

---

## Blazor Component Basics

```razor
@* Counter.razor *@
@page "/counter"

<h1>Counter</h1>
<p>Current count: @currentCount</p>

<button class="btn btn-primary" @onclick="IncrementCount">Click me</button>

@code {
    private int currentCount = 0;

    private void IncrementCount()
    {
        currentCount++;
        // StateHasChanged() called automatically after event handlers
    }
}
```

### Component Parameters

```razor
@* TransactionCard.razor *@

<div class="card @(IsHighlighted ? "border-yellow" : "")">
    <h3>@Transaction.Description</h3>
    <p>₱@Transaction.Amount.ToString("N2")</p>
    @if (OnDelete.HasDelegate)
    {
        <button @onclick="HandleDelete">Delete</button>
    }
</div>

@code {
    [Parameter] public TransactionDto Transaction { get; set; } = null!;
    [Parameter] public bool IsHighlighted { get; set; }
    [Parameter] public EventCallback<Guid> OnDelete { get; set; }

    private async Task HandleDelete()
    {
        await OnDelete.InvokeAsync(Transaction.Id);
    }
}

@* Parent usage: *@
<TransactionCard
    Transaction="@tx"
    IsHighlighted="@(tx.Amount > 10000)"
    OnDelete="HandleDelete" />
```

---

## Data Binding

```razor
@* Two-way binding: *@
<input @bind="searchText" @bind:event="oninput" placeholder="Search..." />
<p>Searching: @searchText</p>

@* For complex objects, use @bind-Value: *@
<InputText @bind-Value="newTransaction.Description" />

@code {
    private string searchText = "";
    private CreateTransactionDto newTransaction = new();
}
```

---

## Lifecycle Methods

```csharp
@code {
    // Called after each render (first + updates):
    protected override void OnParametersSet()
    {
        // Parameters have been set — use to react to param changes
    }

    // Called once after first render — start async work here:
    protected override async Task OnInitializedAsync()
    {
        transactions = await TransactionService.GetAllAsync();
    }

    // Called after DOM is rendered — use for JS interop:
    protected override async Task OnAfterRenderAsync(bool firstRender)
    {
        if (firstRender)
        {
            await JSRuntime.InvokeVoidAsync("initChart", canvasRef);
        }
    }

    // Dispose when component is removed:
    public void Dispose()
    {
        _subscription?.Dispose();
        _timer?.Dispose();
    }
}
```

---

## Dependency Injection

```razor
@inject ITransactionService TransactionService
@inject NavigationManager Navigation
@inject IJSRuntime JSRuntime
@inject AuthenticationStateProvider AuthStateProvider

@* Or in @code: *@
@code {
    [Inject] private ITransactionService TransactionService { get; set; } = null!;
}
```

---

## Forms & Validation

```razor
<EditForm Model="@model" OnValidSubmit="HandleSubmit">
    <DataAnnotationsValidator />
    <ValidationSummary />

    <div class="form-group">
        <label>Amount</label>
        <InputNumber @bind-Value="model.Amount" class="form-control" />
        <ValidationMessage For="@(() => model.Amount)" />
    </div>

    <div class="form-group">
        <label>Type</label>
        <InputSelect @bind-Value="model.Type" class="form-control">
            <option value="">Select type...</option>
            @foreach (var type in Enum.GetValues<TransactionType>())
            {
                <option value="@type">@type.ToString()</option>
            }
        </InputSelect>
    </div>

    <button type="submit" disabled="@isSubmitting">
        @(isSubmitting ? "Saving..." : "Save")
    </button>
</EditForm>

@code {
    private CreateTransactionDto model = new();
    private bool isSubmitting;

    private async Task HandleSubmit()
    {
        isSubmitting = true;
        try
        {
            await TransactionService.CreateAsync(model);
            Navigation.NavigateTo("/transactions");
        }
        finally
        {
            isSubmitting = false;
        }
    }
}
```

---

## JavaScript Interop

```razor
@inject IJSRuntime JSRuntime

@* Call JS from C#: *@
@code {
    private ElementReference chartCanvas;

    protected override async Task OnAfterRenderAsync(bool firstRender)
    {
        if (firstRender)
        {
            var data = new { labels = new[] {"Jan","Feb","Mar"}, values = new[] {100,200,150} };
            await JSRuntime.InvokeVoidAsync("renderChart", chartCanvas, data);
        }
    }
    
    // Call JS function that returns a value:
    private async Task<string> GetClipboardText()
    {
        return await JSRuntime.InvokeAsync<string>("navigator.clipboard.readText");
    }
}
```

```javascript
// wwwroot/js/interop.js:
window.renderChart = (canvas, data) => {
    new Chart(canvas, {
        type: 'line',
        data: { labels: data.labels, datasets: [{ data: data.values }] }
    });
};

// Called from JS → C# (Blazor WASM):
DotNet.invokeMethodAsync('AiTutor.Web', 'OnNotificationReceived', messageJson);
```

---

## Authentication in Blazor WASM

```csharp
// Custom auth state provider:
public class JwtAuthStateProvider : AuthenticationStateProvider
{
    private readonly ILocalStorageService _localStorage;
    
    public override async Task<AuthenticationState> GetAuthenticationStateAsync()
    {
        var token = await _localStorage.GetItemAsync<string>("jwt");
        
        if (string.IsNullOrWhiteSpace(token))
            return new AuthenticationState(new ClaimsPrincipal(new ClaimsIdentity()));
        
        // Parse JWT claims:
        var claims = ParseClaimsFromJwt(token);
        var identity = new ClaimsIdentity(claims, "jwt");
        return new AuthenticationState(new ClaimsPrincipal(identity));
    }
    
    public void NotifyUserLoggedIn(string token)
    {
        var claims = ParseClaimsFromJwt(token);
        var identity = new ClaimsIdentity(claims, "jwt");
        NotifyAuthenticationStateChanged(
            Task.FromResult(new AuthenticationState(new ClaimsPrincipal(identity))));
    }
    
    public void NotifyUserLoggedOut()
    {
        NotifyAuthenticationStateChanged(
            Task.FromResult(new AuthenticationState(new ClaimsPrincipal(new ClaimsIdentity()))));
    }
}
```

```razor
@* AuthorizeView — show different content per auth state: *@
<AuthorizeView>
    <Authorized>
        <p>Welcome, @context.User.Identity?.Name!</p>
        <button @onclick="Logout">Logout</button>
    </Authorized>
    <NotAuthorized>
        <a href="/login">Login</a>
    </NotAuthorized>
</AuthorizeView>

@* Require auth for entire page: *@
@attribute [Authorize]
@attribute [Authorize(Roles = "Admin")]
```

---

## Blazor vs React — When to Choose

| Choose Blazor WASM When | Choose React When |
|---|---|
| Team is C#-first | Team is JS/TS-first |
| Rich .NET ecosystem needed on client | Massive JS ecosystem needed (plugins, etc.) |
| Internal tools, no SEO needed | Public-facing, SEO critical |
| Want single language across stack | Separate frontend team |

---

## Interview Q&A

**Q1: What is the main performance difference between Blazor Server and Blazor WASM?**
> Blazor Server is fast to load (no large download) but every UI interaction causes a server round-trip via SignalR — adds latency and doesn't work offline. Blazor WASM has a large initial download (loads the .NET runtime into the browser) but then runs fully client-side with zero network latency for UI interactions and works offline.

**Q2: What is `StateHasChanged()` and when do you need to call it?**
> `StateHasChanged()` tells Blazor the component state has changed and it should re-render. Blazor automatically calls it after event handlers (`@onclick`, etc.). You need to call it manually when: updating state from a background thread, a timer callback, a `Task` continuation, or after receiving a notification from a service.

**Q3: What is the difference between `@bind` and `@bind-Value`?**
> `@bind` is shorthand — it binds to the element's default bindable property (e.g., `value` for `<input>`). `@bind-Value` is used with Blazor InputBase components (`InputText`, `InputNumber`, `InputSelect`) which expose a `Value` parameter with validation support and work inside `EditForm`.

**Q4: How do you pass data from a child component to its parent in Blazor?**
> Use `EventCallback<T>` — a parent passes a callback as a parameter; the child invokes it via `await OnSomethingHappened.InvokeAsync(data)`. For global state across unrelated components, use a service registered as Scoped with `StateChanged` event, or use Fluxor (Redux-like state management for Blazor).
