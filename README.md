# 🎓 Senior Full Stack .NET Developer — Assessment Reviewer

> Complete, self-contained reviewer for Coderbyte Senior Full Stack .NET Developer assessments.  
> **No external research needed** — everything you need is here.  
> Code examples are linked to the real **BudgetPH** project: [github.com/randolfsegubre/BudgetPH](https://github.com/randolfsegubre/BudgetPH)

---

## 📁 Repository Structure

```
dotnet-assessment-reviewer/
├── SENIOR_REVIEWER.md              ← Comprehensive single-file reference (all topics)
├── topics/                         ← Deep-dive per-topic files
│   ├── 01-csharp-advanced.md
│   ├── 02-dotnet-aspnetcore.md
│   ├── 03-entity-framework.md
│   ├── 04-clean-architecture.md
│   ├── 05-design-patterns.md
│   ├── 06-database-sql.md
│   ├── 07-auth-security.md
│   ├── 08-react-frontend.md
│   ├── 09-testing.md
│   ├── 10-api-design.md
│   ├── 11-performance-caching.md
│   ├── 12-microservices.md
│   ├── 13-typescript.md
│   ├── 14-javascript-es6.md
│   ├── 15-html5-css3.md
│   ├── 16-json-xml.md
│   ├── 17-angular.md
│   ├── 18-vuejs.md
│   ├── 19-signalr-realtime.md
│   ├── 20-docker-containers.md
│   ├── 21-azure-dotnet.md
│   ├── 22-event-driven.md
│   ├── 23-domain-driven-design.md
│   ├── 24-web-performance.md
│   ├── 25-observability-logging.md
│   ├── 26-cicd-devops.md
│   ├── 27-blazor.md
│   └── 28-system-design.md
└── mock-exams/
    ├── MOCK-EXAM-01-multiple-choice.md   ← 50 MCQ with answers
    ├── MOCK-EXAM-02-short-answer.md      ← 25 short answer + code writing
    └── MOCK-EXAM-03-coding-challenges.md ← 10 coding problems with solutions
```

---

## 📋 Topic Index

| # | Topic | Key Concepts |
|---|-------|-------------|
| 01 | [C# Advanced Features](./topics/01-csharp-advanced.md) | Generics, LINQ, async/await, delegates, records, pattern matching |
| 02 | [.NET Core & ASP.NET Core](./topics/02-dotnet-aspnetcore.md) | DI lifetimes, middleware, configuration, filters, minimal APIs |
| 03 | [Entity Framework Core](./topics/03-entity-framework.md) | Code-first, migrations, relationships, LINQ, N+1, performance |
| 04 | [Clean Architecture & SOLID](./topics/04-clean-architecture.md) | SOLID, layers, CQRS, MediatR, repository, domain events |
| 05 | [Design Patterns](./topics/05-design-patterns.md) | Creational, structural, behavioral — GoF patterns with C# examples |
| 06 | [Database & SQL](./topics/06-database-sql.md) | Normalization, indexing, transactions, window functions, T-SQL |
| 07 | [Auth & Security](./topics/07-auth-security.md) | JWT, OAuth2, ASP.NET Identity, OWASP Top 10, CORS |
| 08 | [React & Frontend](./topics/08-react-frontend.md) | Hooks, TanStack Query, Zustand, performance, react-hook-form |
| 09 | [Testing Strategies](./topics/09-testing.md) | xUnit, Moq, integration tests, AAA pattern, code coverage |
| 10 | [REST API Design](./topics/10-api-design.md) | REST principles, HTTP verbs/status codes, versioning, pagination |
| 11 | [Performance & Caching](./topics/11-performance-caching.md) | IMemoryCache, Redis, async patterns, EF Core tuning |
| 12 | [Microservices & Architecture](./topics/12-microservices.md) | Microservices, event-driven, message queues, API gateway |
| 13 | [TypeScript](./topics/13-typescript.md) | Generics, utility types, React integration, tsconfig, discriminated unions |
| 14 | [JavaScript ES6+](./topics/14-javascript-es6.md) | Closures, this, prototypes, Promises, async/await, modules, event loop |
| 15 | [HTML5 & CSS3](./topics/15-html5-css3.md) | Semantic HTML, APIs, Flexbox, Grid, custom properties, animations, a11y |
| 16 | [JSON & XML](./topics/16-json-xml.md) | System.Text.Json, JSON Schema, XML/LINQ, SOAP, content negotiation |
| 17 | [Angular](./topics/17-angular.md) | Standalone components, Services/DI, RxJS, Router/guards, Signals, forms |
| 18 | [Vue.js 3](./topics/18-vuejs.md) | Composition API, ref/reactive, Pinia, Vue Router, composables |
| 19 | [SignalR & Real-Time](./topics/19-signalr-realtime.md) | Hubs, IHubContext, JS client, streaming, Redis scale-out |
| 20 | [Docker & Containers](./topics/20-docker-containers.md) | Multi-stage Dockerfile, Docker Compose, layer caching, health checks |
| 21 | [Azure for .NET](./topics/21-azure-dotnet.md) | App Service, Key Vault, Blob Storage, Service Bus, App Insights, Functions |
| 22 | [Event-Driven Architecture](./topics/22-event-driven.md) | Domain/integration events, MassTransit, outbox pattern, saga, event sourcing |
| 23 | [Domain-Driven Design](./topics/23-domain-driven-design.md) | Bounded contexts, entities, value objects, aggregates, domain services |
| 24 | [Web Performance](./topics/24-web-performance.md) | Core Web Vitals, React.memo, code splitting, virtualization, EF tuning |
| 25 | [Observability & Logging](./topics/25-observability-logging.md) | Serilog, OpenTelemetry, Application Insights, health checks, correlation IDs |
| 26 | [CI/CD & DevOps](./topics/26-cicd-devops.md) | GitHub Actions, Docker pipelines, deploy to Azure, secrets management |
| 27 | [Blazor](./topics/27-blazor.md) | Server vs WASM, components, forms, JS interop, auth, lifecycle |
| 28 | [System Design](./topics/28-system-design.md) | CAP theorem, DB selection, caching, pagination, URL shortener, notifications |

---

## 🗓️ Study Plan

### ⚡ 1-Week Intensive
| Day | Topics | Mock Exam |
|-----|--------|-----------|
| 1 | 01 C# + 02 .NET Core + 13 TypeScript | — |
| 2 | 03 EF Core + 04 Clean Architecture + 23 DDD | — |
| 3 | 05 Design Patterns + 06 Database + 28 System Design | MOCK-EXAM-01 |
| 4 | 07 Auth/Security + 08 React + 14-18 JS/HTML/Frameworks | — |
| 5 | 09 Testing + 10 API Design + 19-20 SignalR/Docker | MOCK-EXAM-02 |
| 6 | 11 Performance + 12 Microservices + 22 Event-Driven | MOCK-EXAM-03 |
| 7 | 21 Azure + 24-27 Perf/Observability/CI-CD/Blazor | All timed |

### 📅 2-Week Plan
- **Week 1**: Topics 01–14 + `SENIOR_REVIEWER.md` sections
- **Week 2**: Topics 15–28 + all three mock exams

---

## 🔥 Highest-Priority Topics (Coderbyte Hot List)

These appear in almost every Senior .NET assessment:

1. **DI Lifetimes** (Singleton/Scoped/Transient + captive dependency) → [02](./topics/02-dotnet-aspnetcore.md)
2. **async/await** (Task, ValueTask, ConfigureAwait, deadlocks) → [01](./topics/01-csharp-advanced.md)
3. **LINQ** (deferred execution, IQueryable vs IEnumerable) → [01](./topics/01-csharp-advanced.md)
4. **SOLID Principles** (with code examples) → [04](./topics/04-clean-architecture.md)
5. **Design Patterns** (Singleton, Repository, Factory, Strategy) → [05](./topics/05-design-patterns.md)
6. **JWT Authentication** (structure, validation, refresh tokens) → [07](./topics/07-auth-security.md)
7. **N+1 Problem** (EF Core eager loading) → [03](./topics/03-entity-framework.md)
8. **SQL Normalization** (1NF–3NF) + Indexing → [06](./topics/06-database-sql.md)
9. **REST HTTP Status Codes** (200/201/204/400/401/403/404/409/500) → [10](./topics/10-api-design.md)
10. **React Hooks** (useState, useEffect, useCallback, useMemo) → [08](./topics/08-react-frontend.md)

---

## 🧪 Mock Exams

| File | Format | Questions | Estimated Time |
|------|--------|-----------|----------------|
| [MOCK-EXAM-01](./mock-exams/MOCK-EXAM-01-multiple-choice.md) | Multiple Choice | 50 | 40 min |
| [MOCK-EXAM-02](./mock-exams/MOCK-EXAM-02-short-answer.md) | Short Answer + Code | 25 | 45 min |
| [MOCK-EXAM-03](./mock-exams/MOCK-EXAM-03-coding-challenges.md) | Coding Challenges | 10 | 90 min |

**Simulate exam conditions:** No notes, timed, no IDE auto-complete.

---

## 🔗 BudgetPH Code References

All code examples in the topic files link to the real BudgetPH codebase:

| File | What it demonstrates |
|------|---------------------|
| [Program.cs](https://github.com/randolfsegubre/BudgetPH/blob/master/src/Presentation/FinanceManager.API/Program.cs) | DI setup, middleware pipeline, JWT config |
| [BaseApiController.cs](https://github.com/randolfsegubre/BudgetPH/blob/master/src/Presentation/FinanceManager.API/Controllers/BaseApiController.cs) | Base controller, claims-based auth |
| [AuthController.cs](https://github.com/randolfsegubre/BudgetPH/blob/master/src/Presentation/FinanceManager.API/Controllers/AuthController.cs) | JWT login, register, refresh token |
| [ApplicationDbContext.cs](https://github.com/randolfsegubre/BudgetPH/blob/master/src/Infrastructure/FinanceManager.Infrastructure/Data/ApplicationDbContext.cs) | EF Core, relationships, seed data |
| [MappingProfile.cs](https://github.com/randolfsegubre/BudgetPH/blob/master/src/Core/FinanceManager.Application/Mappings/MappingProfile.cs) | AutoMapper, DTO projection |
| [Application/DependencyInjection.cs](https://github.com/randolfsegubre/BudgetPH/blob/master/src/Core/FinanceManager.Application/DependencyInjection.cs) | Clean Architecture DI wiring |
| [vite.config.js](https://github.com/randolfsegubre/BudgetPH/blob/master/src/Presentation/FinanceManager.Web/ClientApp/vite.config.js) | Vite proxy, React setup |

---

*Last updated: 2026-07-05 | Stack: .NET 10, C# 13, React 18, TypeScript, Angular, Vue.js, Blazor, Docker, Azure | 28 topics + 3 mock exams*
