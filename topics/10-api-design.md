# 10 — REST API Design

---

## Table of Contents
1. [REST Principles](#1-rest-principles)
2. [HTTP Methods](#2-http-methods)
3. [HTTP Status Codes](#3-http-status-codes)
4. [URL Design Best Practices](#4-url-design-best-practices)
5. [Request & Response Standards](#5-request--response-standards)
6. [Pagination & Filtering](#6-pagination--filtering)
7. [API Versioning](#7-api-versioning)
8. [Error Response Format](#8-error-response-format)
9. [HATEOAS](#9-hateoas)
10. [Common Interview Questions](#10-common-interview-questions)

---

## 1. REST Principles

REST (Representational State Transfer) has 6 architectural constraints:

| Constraint | Meaning | Example |
|---|---|---|
| **Client-Server** | Separation of concerns | React (client) ↔ .NET API (server) |
| **Stateless** | Each request contains all necessary info | JWT in every request |
| **Cacheable** | Responses indicate if they can be cached | `Cache-Control` headers |
| **Uniform Interface** | Consistent interface (resources, HTTP methods) | `/api/accounts/{id}` |
| **Layered System** | Client doesn't know if it's talking to server directly | API Gateway, load balancer |
| **Code on Demand** | (Optional) Server can send executable code | JavaScript from CDN |

---

## 2. HTTP Methods

| Method | Operation | Idempotent? | Safe? | Body? |
|--------|-----------|------------|-------|-------|
| `GET` | Read | ✅ Yes | ✅ Yes | ❌ No |
| `POST` | Create | ❌ No | ❌ No | ✅ Yes |
| `PUT` | Replace (full update) | ✅ Yes | ❌ No | ✅ Yes |
| `PATCH` | Partial update | ⚠️ Depends | ❌ No | ✅ Yes |
| `DELETE` | Delete | ✅ Yes | ❌ No | Optional |
| `HEAD` | Like GET but no body | ✅ Yes | ✅ Yes | ❌ No |
| `OPTIONS` | List supported methods | ✅ Yes | ✅ Yes | ❌ No |

**Idempotent** = calling N times produces same result as calling once.  
**Safe** = doesn't modify server state (read-only).

```csharp
// BudgetPH REST endpoints pattern
GET    /api/accounts          → GetAll accounts for current user
GET    /api/accounts/{id}     → GetById
POST   /api/accounts          → Create new account
PUT    /api/accounts/{id}     → Full replace (update all fields)
PATCH  /api/accounts/{id}     → Partial update (only changed fields)
DELETE /api/accounts/{id}     → Soft delete

// Nested resources (sub-resources)
GET    /api/savings-goals/{id}/contributions     → list contributions
POST   /api/savings-goals/{id}/contributions     → add contribution
```

---

## 3. HTTP Status Codes

### 2xx — Success

| Code | Name | When to Use |
|------|------|-------------|
| `200 OK` | OK | Successful GET, PUT, PATCH |
| `201 Created` | Created | Successful POST that created a resource |
| `204 No Content` | No Content | Successful DELETE or PUT (no body to return) |
| `206 Partial Content` | Partial | Range requests (paginated file downloads) |

### 3xx — Redirection

| Code | Name | When to Use |
|------|------|-------------|
| `301 Moved Permanently` | Moved | URL changed permanently |
| `304 Not Modified` | Not Modified | Cache is fresh (used with ETag) |

### 4xx — Client Errors

| Code | Name | When to Use |
|------|------|-------------|
| `400 Bad Request` | Bad Request | Invalid input, validation failed |
| `401 Unauthorized` | Unauthorized | Not authenticated (missing/invalid token) |
| `403 Forbidden` | Forbidden | Authenticated but not authorized |
| `404 Not Found` | Not Found | Resource doesn't exist |
| `405 Method Not Allowed` | Method Not Allowed | HTTP method not supported |
| `409 Conflict` | Conflict | Duplicate (email already registered) |
| `410 Gone` | Gone | Resource permanently deleted |
| `422 Unprocessable Entity` | Unprocessable | Semantically invalid (valid JSON, invalid business rules) |
| `429 Too Many Requests` | Too Many Requests | Rate limit exceeded |

### 5xx — Server Errors

| Code | Name | When to Use |
|------|------|-------------|
| `500 Internal Server Error` | Internal Error | Unexpected server error |
| `502 Bad Gateway` | Bad Gateway | Upstream service error |
| `503 Service Unavailable` | Service Unavailable | Server overloaded or in maintenance |
| `504 Gateway Timeout` | Gateway Timeout | Upstream service timed out |

```csharp
// 🔗 BudgetPH controller pattern
[HttpGet("{id:guid}")]
public async Task<IActionResult> GetById(Guid id, CancellationToken ct)
{
    var appUser = await GetAppUserAsync(ct);
    if (appUser is null) return Unauthorized();             // 401

    var account = await context.Accounts
        .FirstOrDefaultAsync(a => a.Id == id, ct);

    if (account is null) return NotFound();                  // 404
    if (account.UserId != appUser.Id) return Forbid();       // 403

    return Ok(_mapper.Map<AccountDto>(account));             // 200
}

[HttpPost]
public async Task<IActionResult> Create([FromBody] CreateAccountRequest request, CancellationToken ct)
{
    var account = new Account { ... };
    context.Accounts.Add(account);
    await context.SaveChangesAsync(ct);
    
    return CreatedAtAction(nameof(GetById), new { id = account.Id },  // 201 + Location header
        _mapper.Map<AccountDto>(account));
}

[HttpDelete("{id:guid}")]
public async Task<IActionResult> Delete(Guid id, CancellationToken ct)
{
    // ... delete logic
    return NoContent(); // 204 — success, no body
}
```

---

## 4. URL Design Best Practices

```
✅ GOOD URLs:
GET  /api/v1/accounts                    → list all accounts
GET  /api/v1/accounts/{id}               → get account by ID
POST /api/v1/accounts                    → create account
GET  /api/v1/accounts/{id}/transactions  → transactions for account
GET  /api/v1/transactions?page=1&pageSize=20&type=expense  → filtered

❌ BAD URLs (RPC-style, verbs in URL):
GET  /api/getAccounts
POST /api/createAccount
GET  /api/getAccountTransactions?accountId=123
POST /api/deleteAccount/123

❌ BAD URLs (inconsistent pluralization):
GET /api/account    (should be /accounts)
GET /api/user/1     (mixed: /api/users/{id})

✅ URL conventions:
- Use lowercase, hyphens (not underscores): /savings-goals, /budget-items
- Use nouns (resources), not verbs
- Use plural nouns: /users, /accounts, /transactions
- Nest by relationship (max 2 levels): /accounts/{id}/transactions
- Keep hierarchy shallow: /api/v1/{resource}/{id}/{sub-resource}
- Actions on resources: POST /api/accounts/{id}/deactivate
```

---

## 5. Request & Response Standards

### Response Envelope (Consistent Response Shape)

```json
// ✅ Consistent paginated list response
{
  "data": [...],
  "pagination": {
    "page": 1,
    "pageSize": 20,
    "total": 150,
    "totalPages": 8,
    "hasNextPage": true,
    "hasPreviousPage": false
  }
}

// ✅ Single item response — just the object
{
  "id": "guid",
  "name": "BPI Savings",
  "balance": 50000.00,
  "accountType": "savings"
}
```

### JSON Naming Conventions

```csharp
// ASP.NET Core — configure JSON serialization
builder.Services.AddControllers()
    .AddJsonOptions(options =>
    {
        options.JsonSerializerOptions.PropertyNamingPolicy = JsonNamingPolicy.CamelCase;
        options.JsonSerializerOptions.DefaultIgnoreCondition = 
            JsonIgnoreCondition.WhenWritingNull;
        options.JsonSerializerOptions.Converters.Add(new JsonStringEnumConverter());
    });
```

```json
// C# "Balance" → JSON "balance" (camelCase)
// C# "AccountType" → JSON "accountType"
// C# "TransactionDate" → JSON "transactionDate"
```

---

## 6. Pagination & Filtering

### Offset-Based Pagination (Used in BudgetPH)

```csharp
// Query parameters
GET /api/transactions?page=2&pageSize=20&sort=transactionDate&direction=desc&search=salary

[HttpGet]
public async Task<IActionResult> GetAll(
    [FromQuery] int page = 1,
    [FromQuery] int pageSize = 20,
    [FromQuery] string? search = null,
    [FromQuery] string sort = "transactionDate",
    [FromQuery] string direction = "desc",
    CancellationToken ct = default)
{
    var appUser = await GetAppUserAsync(ct);
    
    var query = context.Transactions
        .Where(t => t.UserId == appUser!.Id)
        .AsQueryable();

    // Dynamic search
    if (!string.IsNullOrWhiteSpace(search))
        query = query.Where(t => t.Description.Contains(search) || 
                                  t.Merchant!.Contains(search));

    // Dynamic sorting (safe — only allow known columns)
    query = sort.ToLower() switch
    {
        "amount" => direction == "asc" 
            ? query.OrderBy(t => t.Amount)
            : query.OrderByDescending(t => t.Amount),
        _ => direction == "asc" 
            ? query.OrderBy(t => t.TransactionDate)
            : query.OrderByDescending(t => t.TransactionDate)
    };

    var total = await query.CountAsync(ct);
    var items = await query
        .Skip((page - 1) * pageSize)
        .Take(pageSize)
        .Select(t => _mapper.Map<TransactionDto>(t))
        .ToListAsync(ct);

    return Ok(new
    {
        data = items,
        pagination = new { page, pageSize, total, totalPages = (int)Math.Ceiling((double)total / pageSize) }
    });
}
```

### Cursor-Based Pagination (Better for Large Datasets)

```csharp
// Cursor pagination — no skipping, uses last item's ID/timestamp
GET /api/transactions?cursor=last-transaction-id&pageSize=20

public async Task<IActionResult> GetAll(
    [FromQuery] string? cursor = null,
    [FromQuery] int pageSize = 20,
    CancellationToken ct = default)
{
    DateTime? cursorDate = cursor is not null 
        ? Convert.FromBase64String(cursor).ToDateTime() 
        : null;

    var query = context.Transactions.Where(t => t.UserId == userId);

    if (cursorDate.HasValue)
        query = query.Where(t => t.TransactionDate < cursorDate); // start after cursor

    var items = await query
        .OrderByDescending(t => t.TransactionDate)
        .Take(pageSize + 1) // fetch one extra to detect hasNextPage
        .ToListAsync(ct);

    var hasNextPage = items.Count > pageSize;
    if (hasNextPage) items.RemoveAt(items.Count - 1);

    var nextCursor = hasNextPage 
        ? Convert.ToBase64String(Encoding.UTF8.GetBytes(items.Last().TransactionDate.ToString("O")))
        : null;

    return Ok(new { data = items, nextCursor, hasNextPage });
}
```

---

## 7. API Versioning

```csharp
// Install: dotnet add package Asp.Versioning.Mvc

// Program.cs
builder.Services.AddApiVersioning(options =>
{
    options.DefaultApiVersion = new ApiVersion(1, 0);
    options.AssumeDefaultVersionWhenUnspecified = true;
    options.ReportApiVersions = true; // adds api-supported-versions response header
    options.ApiVersionReader = ApiVersionReader.Combine(
        new UrlSegmentApiVersionReader(),       // /api/v1/accounts
        new HeaderApiVersionReader("api-version"), // Header: api-version: 1.0
        new QueryStringApiVersionReader("ver")  // ?ver=1.0
    );
});

// Controller versioning
[ApiController]
[Route("api/v{version:apiVersion}/[controller]")]
[ApiVersion("1.0")]
[ApiVersion("2.0")]
public class AccountsController : ControllerBase
{
    [HttpGet]
    [MapToApiVersion("1.0")]
    public IActionResult GetAllV1() { ... }

    [HttpGet]
    [MapToApiVersion("2.0")]
    public IActionResult GetAllV2() { ... } // new response shape
}
```

### Versioning Strategies Compared

| Strategy | URL Example | Pros | Cons |
|---|---|---|---|
| **URL Segment** | `/api/v1/accounts` | Easy to see, easy to test | "Dirty" URLs, not truly REST |
| **Header** | `api-version: 1.0` | Clean URLs | Harder to test in browser |
| **Query String** | `?api-version=1.0` | Easy to test | Pollutes query string |
| **Media Type** | `Accept: application/vnd.v1+json` | Most RESTful | Complex, rarely used |

---

## 8. Error Response Format

### RFC 7807 — Problem Details (Standard)

```csharp
// ASP.NET Core has built-in Problem Details support
builder.Services.AddProblemDetails();

// Custom Problem Details
return Problem(
    title: "Account not found",
    detail: $"No account with ID {id} exists for the current user.",
    statusCode: 404,
    type: "https://budgetph.com/errors/account-not-found",
    instance: $"/api/accounts/{id}"
);

// Response JSON:
{
    "type": "https://budgetph.com/errors/account-not-found",
    "title": "Account not found",
    "status": 404,
    "detail": "No account with ID abc123 exists for the current user.",
    "instance": "/api/accounts/abc123",
    "traceId": "0HN2BGFD6QFKE:00000001"  // added by ASP.NET Core
}

// Validation error:
{
    "type": "https://tools.ietf.org/html/rfc7807",
    "title": "One or more validation errors occurred.",
    "status": 400,
    "errors": {
        "Amount": ["Amount must be positive"],
        "Description": ["Description is required"]
    }
}
```

---

## 9. HATEOAS

Hypermedia As The Engine Of Application State — responses include links to related actions.

```json
{
    "id": "account-guid",
    "name": "BPI Savings",
    "balance": 50000.00,
    "_links": {
        "self": { "href": "/api/accounts/account-guid", "method": "GET" },
        "transactions": { "href": "/api/accounts/account-guid/transactions", "method": "GET" },
        "update": { "href": "/api/accounts/account-guid", "method": "PUT" },
        "delete": { "href": "/api/accounts/account-guid", "method": "DELETE" },
        "transfer": { "href": "/api/transfers", "method": "POST" }
    }
}
```

HATEOAS is rarely fully implemented in practice (complex to maintain), but the concept is commonly tested.

---

## 10. Common Interview Questions

**Q1: What is the difference between PUT and PATCH?**

> `PUT` replaces the **entire resource** — you must send all fields. If you omit a field, it becomes null/default. `PATCH` applies a **partial update** — you only send the fields you want to change. Use `PATCH` for efficient updates of large resources where only a few fields change.

**Q2: Why is 401 called "Unauthorized" when it actually means unauthenticated?**

> It's a historical naming mistake. RFC 7235 clarifies: 401 means the request lacks valid authentication credentials (the user is not authenticated). 403 means the server understood the request but refuses to authorize it (the user is authenticated but lacks permission). Semantically: 401 = "you need to log in", 403 = "you're logged in but don't have access".

**Q3: What is the difference between REST and RPC?**

> **REST** focuses on resources (nouns) and uses HTTP semantics (methods, status codes) to indicate operations. `GET /accounts` fetches accounts; `DELETE /accounts/{id}` deletes. **RPC** (Remote Procedure Call) focuses on operations (verbs). `POST /getAccounts`, `POST /deleteAccount`. REST is resource-centric; RPC is operation-centric. REST is more aligned with HTTP's design.

**Q4: What are idempotent and safe HTTP methods?**

> **Safe** methods don't modify server state: GET, HEAD, OPTIONS. **Idempotent** methods produce the same result regardless of how many times called: GET, PUT, DELETE (and HEAD, OPTIONS). POST is neither safe nor idempotent — calling POST twice creates two resources. PATCH may or may not be idempotent (depends on implementation — absolute values = idempotent, relative changes like `add 5` = not).

**Q5: How would you design an API that needs to handle long-running operations?**

> Use the **202 Accepted** pattern: 1) Client posts a job request, server returns 202 with a `Location` header pointing to a status endpoint. 2) Client polls `GET /jobs/{id}` for status. 3) When complete, the status response contains a link to the result or the result itself. Alternatively, use webhooks (server calls client when done) or WebSockets for real-time updates.

**Q6: What is the `ETag` header and how does it work?**

> `ETag` is a cache validator. The server includes `ETag: "v1hash123"` in a response. On subsequent requests, the client sends `If-None-Match: "v1hash123"`. If the resource hasn't changed, the server returns `304 Not Modified` (no body — saves bandwidth). If changed, returns 200 with new content and a new ETag. Used with `If-Match` for optimistic concurrency control on updates.
