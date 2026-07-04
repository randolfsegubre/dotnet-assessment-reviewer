# Topic 24 — Web Performance

> Senior full-stack developers own performance. You need to measure, identify bottlenecks, and fix them on both frontend and backend.

---

## Frontend Performance Metrics (Core Web Vitals)

| Metric | Measures | Good | Needs Work |
|---|---|---|---|
| **LCP** (Largest Contentful Paint) | Load performance | ≤ 2.5s | > 4s |
| **FID/INP** (Interaction to Next Paint) | Interactivity | ≤ 200ms | > 500ms |
| **CLS** (Cumulative Layout Shift) | Visual stability | ≤ 0.1 | > 0.25 |
| **TTFB** (Time to First Byte) | Server response | ≤ 800ms | > 1800ms |
| **FCP** (First Contentful Paint) | First render | ≤ 1.8s | > 3s |

---

## React Performance Optimization

### Preventing Unnecessary Re-renders

```typescript
// ❌ Re-renders every time parent renders:
const TransactionList = ({ transactions, onDelete }) => (
    transactions.map(tx => <TransactionCard key={tx.id} tx={tx} onDelete={onDelete} />)
)

// ✅ React.memo — skip re-render if props didn't change:
const TransactionCard = React.memo(({ tx, onDelete }: Props) => {
    return <div onClick={() => onDelete(tx.id)}>{tx.description}</div>
})

// ✅ useCallback — stable function reference (don't recreate on every render):
const handleDelete = useCallback((id: string) => {
    setTransactions(prev => prev.filter(t => t.id !== id))
}, []) // no deps = same reference forever

// ✅ useMemo — expensive computation cached:
const totals = useMemo(() => ({
    income:   transactions.filter(t => t.type === 'income').reduce((s, t) => s + t.amount, 0),
    expenses: transactions.filter(t => t.type === 'expense').reduce((s, t) => s + t.amount, 0),
}), [transactions]) // only recomputes when transactions changes

// When NOT to use useMemo/useCallback:
// - Simple calculations (the memoization overhead > savings)
// - Objects that always change anyway
// Rule: profile first, then optimize
```

### Code Splitting & Lazy Loading

```typescript
// Route-level code splitting (Vite/React):
import { lazy, Suspense } from 'react'

const DashboardPage = lazy(() => import('./pages/DashboardPage'))
const TransactionsPage = lazy(() => import('./pages/TransactionsPage'))
const ReportsPage = lazy(() => import('./pages/ReportsPage'))

function App() {
    return (
        <Suspense fallback={<PageSkeleton />}>
            <Routes>
                <Route path="/dashboard" element={<DashboardPage />} />
                <Route path="/transactions" element={<TransactionsPage />} />
                <Route path="/reports" element={<ReportsPage />} /> {/* loaded on demand */}
            </Routes>
        </Suspense>
    )
}

// Component-level splitting — heavy chart library loaded only when visible:
const HeavyChart = lazy(() => import('./components/HeavyChart'))

// Prefetch — load before user navigates:
const prefetchReports = () => import('./pages/ReportsPage') // call on hover
```

### List Virtualization

```typescript
// ❌ 10,000 DOM nodes = slow render + scroll:
transactions.map(tx => <TransactionCard key={tx.id} transaction={tx} />)

// ✅ Virtualization — only renders visible rows:
import { FixedSizeList as List } from 'react-window'

const TransactionList = ({ transactions }) => (
    <List
        height={600}           // visible area height
        itemCount={transactions.length}
        itemSize={72}          // each row height
        width="100%"
    >
        {({ index, style }) => (
            <TransactionCard
                key={transactions[index].id}
                transaction={transactions[index]}
                style={style}  // MUST pass style for positioning
            />
        )}
    </List>
)
// Renders ~10 items regardless of list size — O(1) DOM nodes
```

### Image Optimization

```html
<!-- Lazy load images: -->
<img src="chart.png" loading="lazy" alt="Spending chart" />

<!-- Responsive images: -->
<img
    srcset="chart-400.webp 400w, chart-800.webp 800w, chart-1200.webp 1200w"
    sizes="(max-width: 768px) 100vw, (max-width: 1024px) 50vw, 33vw"
    src="chart-800.webp"
    alt="Spending chart"
/>

<!-- Modern formats: WebP (30% smaller than JPEG), AVIF (50% smaller): -->
<picture>
    <source srcset="chart.avif" type="image/avif" />
    <source srcset="chart.webp" type="image/webp" />
    <img src="chart.jpg" alt="Spending chart" />
</picture>
```

---

## Vite Bundle Optimization

```typescript
// vite.config.ts:
export default defineConfig({
    build: {
        rollupOptions: {
            output: {
                // Manual chunks — split large deps into separate files:
                manualChunks: {
                    'react-vendor': ['react', 'react-dom', 'react-router-dom'],
                    'chart-vendor': ['recharts'],
                    'query-vendor': ['@tanstack/react-query'],
                }
            }
        },
        // Chunk size warning threshold:
        chunkSizeWarningLimit: 500 // KB
    }
})
```

---

## ASP.NET Core Performance

### Response Compression

```csharp
builder.Services.AddResponseCompression(options =>
{
    options.EnableForHttps = true;
    options.Providers.Add<BrotliCompressionProvider>();
    options.Providers.Add<GzipCompressionProvider>();
    options.MimeTypes = ResponseCompressionDefaults.MimeTypes
        .Concat(["application/json", "image/svg+xml"]);
});

builder.Services.Configure<BrotliCompressionProviderOptions>(opt =>
    opt.Level = CompressionLevel.Fastest);

app.UseResponseCompression(); // before UseStaticFiles
```

### EF Core Query Performance

```csharp
// 1. AsNoTracking for read-only:
var txs = await db.Transactions.AsNoTracking().Where(...).ToListAsync();

// 2. Projection — select only needed columns:
var summary = await db.Transactions
    .AsNoTracking()
    .Where(t => t.UserId == userId)
    .Select(t => new { t.Amount, t.Type, t.Date }) // 3 columns, not 20
    .ToListAsync();

// 3. Compiled queries — reuse query plan:
private static readonly Func<AppDbContext, Guid, IAsyncEnumerable<Transaction>>
    GetByUserQuery = EF.CompileAsyncQuery(
        (AppDbContext db, Guid userId) =>
            db.Transactions.AsNoTracking().Where(t => t.UserId == userId)
    );

// 4. Pagination — never load all:
var page = await db.Transactions
    .OrderByDescending(t => t.Date)
    .Skip((pageNumber - 1) * pageSize)
    .Take(pageSize)
    .ToListAsync();

// 5. Split queries for multiple Includes:
var account = await db.Accounts
    .Include(a => a.Transactions)
    .Include(a => a.Budgets)
    .AsSplitQuery()
    .FirstOrDefaultAsync(a => a.Id == id);
```

### Output Caching (.NET 7+)

```csharp
builder.Services.AddOutputCache(options =>
{
    options.AddBasePolicy(b => b.Expire(TimeSpan.FromSeconds(60)));
    options.AddPolicy("Transactions", b => b
        .Expire(TimeSpan.FromMinutes(5))
        .Tag("transactions")
        .VaryByHeader("Authorization")); // different cache per user
});

[HttpGet]
[OutputCache(PolicyName = "Transactions", Duration = 300)]
public async Task<IActionResult> GetAll() { /* ... */ }

// Invalidate cache when data changes:
public async Task CreateTransaction(...)
{
    // save...
    await _outputCache.EvictByTagAsync("transactions"); // clears all transaction caches
}
```

### Parallel Async Operations

```csharp
// ❌ Sequential — total time = t1 + t2 + t3:
var accounts     = await _accountService.GetSummaryAsync(userId);
var transactions = await _txService.GetRecentAsync(userId);
var budgets      = await _budgetService.GetCurrentAsync(userId);

// ✅ Parallel — total time = max(t1, t2, t3):
var (accounts, transactions, budgets) = await (
    _accountService.GetSummaryAsync(userId),
    _txService.GetRecentAsync(userId),
    _budgetService.GetCurrentAsync(userId)
).WhenAll();
// ValueTuple + Task.WhenAll extension:
static async Task<(T1, T2, T3)> WhenAll<T1, T2, T3>(
    this (Task<T1>, Task<T2>, Task<T3>) tasks)
{
    await Task.WhenAll(tasks.Item1, tasks.Item2, tasks.Item3);
    return (tasks.Item1.Result, tasks.Item2.Result, tasks.Item3.Result);
}
```

---

## Profiling Tools

| Tool | For | What It Shows |
|---|---|---|
| Chrome DevTools Performance | Frontend | Rendering, scripting, layout, paint |
| Chrome Lighthouse | Frontend + SEO | CWV scores, best practice violations |
| React DevTools Profiler | React | Component render times, wasted renders |
| dotnet-counters | .NET | CPU, memory, GC, thread pool |
| dotnet-trace | .NET | CPU flame graphs |
| SQL Server Profiler / EF Logs | Database | Query execution times, N+1 detection |
| Application Insights | Full stack | End-to-end traces |

---

## Interview Q&A

**Q1: What is the Critical Rendering Path?**
> The sequence the browser follows to convert HTML/CSS/JS into pixels: Parse HTML → Build DOM → Parse CSS → Build CSSOM → Combine DOM+CSSOM into Render Tree → Layout (compute positions) → Paint (fill pixels) → Composite. Blocking resources (render-blocking CSS/JS) pause this path. Optimization: defer JS, inline critical CSS, preload key resources.

**Q2: What is `React.memo` and when should you NOT use it?**
> `React.memo` is a HOC that skips re-rendering if props haven't changed (shallow equality check). Don't use it when: props change every render anyway (making memoization overhead wasteful), the component is cheap to render, or props include complex objects that change reference but not value. Profile first — premature optimization adds complexity without benefit.

**Q3: What is the difference between `useCallback` and `useMemo`?**
> `useMemo` memoizes the return value of a function. `useCallback` memoizes the function itself. `useCallback(fn, deps)` is equivalent to `useMemo(() => fn, deps)`. Use `useCallback` when passing callbacks to memoized child components (so they don't re-render due to new function reference). Use `useMemo` for expensive computations.

**Q4: How would you diagnose and fix an N+1 query problem?**
> Diagnose: enable EF Core query logging or use Application Insights — look for many similar SQL queries in one request. Fix: add `Include()` for eager loading, or use projection with `Select()` to load related data in one query. Use `AsSplitQuery()` when including multiple collections to avoid Cartesian explosion.
