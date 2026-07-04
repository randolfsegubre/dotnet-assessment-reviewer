# Topic 26 — CI/CD & DevOps

> Senior developers own the deployment pipeline. CI/CD ensures code is automatically tested and deployed reliably.

---

## CI/CD Concepts

```
CI (Continuous Integration) — automatically build + test on every commit
CD (Continuous Delivery)    — automatically deploy to staging; manual approval for prod
CD (Continuous Deployment)  — automatically deploy to prod if all tests pass

Pipeline stages:
1. Trigger (push/PR)
2. Build (.NET publish / npm build)
3. Test (unit + integration tests)
4. Code Quality (linting, security scan)
5. Build Docker Image + Push to registry
6. Deploy to staging
7. Smoke tests
8. Manual approval gate (optional)
9. Deploy to production
```

---

## GitHub Actions — .NET Build + Test

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  build-and-test:
    name: Build & Test
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup .NET 10
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '10.0.x'

      - name: Restore dependencies
        run: dotnet restore

      - name: Build
        run: dotnet build --no-restore --configuration Release

      - name: Run tests
        run: dotnet test --no-build --configuration Release --verbosity normal \
               --logger "trx;LogFileName=results.trx" \
               --collect:"XPlat Code Coverage"

      - name: Upload test results
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: test-results
          path: '**/results.trx'

      - name: Upload coverage
        uses: codecov/codecov-action@v4
        with:
          token: ${{ secrets.CODECOV_TOKEN }}
```

---

## GitHub Actions — Docker Build & Push

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}/api

jobs:
  build-push:
    name: Build & Push Docker Image
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Log in to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=sha,prefix=sha-
            type=ref,event=branch
            type=semver,pattern={{version}}

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          file: ./src/Presentation/FinanceManager.API/Dockerfile
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

---

## GitHub Actions — Deploy to Azure App Service

```yaml
  deploy-staging:
    name: Deploy to Staging
    runs-on: ubuntu-latest
    needs: build-push
    environment: staging

    steps:
      - name: Deploy to Azure Web App
        uses: azure/webapps-deploy@v3
        with:
          app-name: 'financemanager-staging'
          publish-profile: ${{ secrets.AZURE_WEBAPP_PUBLISH_PROFILE }}
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:sha-${{ github.sha }}

  deploy-production:
    name: Deploy to Production
    runs-on: ubuntu-latest
    needs: deploy-staging
    environment: production  # has required reviewers (manual approval gate)

    steps:
      - name: Deploy to Azure Web App
        uses: azure/webapps-deploy@v3
        with:
          app-name: 'financemanager-prod'
          publish-profile: ${{ secrets.AZURE_WEBAPP_PUBLISH_PROFILE_PROD }}
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:sha-${{ github.sha }}
```

---

## Multi-Stage Dockerfile for ASP.NET Core

```dockerfile
# Multi-stage build — final image only contains runtime, not SDK:

# Stage 1: Build
FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build
WORKDIR /src

# Copy project files first (better layer caching):
COPY ["src/Core/FinanceManager.Domain/FinanceManager.Domain.csproj", "Core/Domain/"]
COPY ["src/Core/FinanceManager.Application/FinanceManager.Application.csproj", "Core/Application/"]
COPY ["src/Infrastructure/FinanceManager.Infrastructure/FinanceManager.Infrastructure.csproj", "Infrastructure/"]
COPY ["src/Presentation/FinanceManager.API/FinanceManager.API.csproj", "Presentation/API/"]

# Restore — cached unless .csproj files change:
RUN dotnet restore "Presentation/API/FinanceManager.API.csproj"

# Copy all source:
COPY src/ .

# Publish:
RUN dotnet publish "Presentation/API/FinanceManager.API.csproj" \
    --no-restore -c Release -o /app/publish

# Stage 2: Runtime (much smaller image — no SDK)
FROM mcr.microsoft.com/dotnet/aspnet:10.0 AS runtime
WORKDIR /app

# Security: run as non-root:
RUN addgroup --system appgroup && adduser --system --ingroup appgroup appuser
USER appuser

COPY --from=build /app/publish .

EXPOSE 8080
ENV ASPNETCORE_URLS=http://+:8080
ENTRYPOINT ["dotnet", "FinanceManager.API.dll"]
```

---

## Docker Compose for Local Dev

```yaml
# docker-compose.yml:
version: '3.9'

services:
  api:
    build:
      context: .
      dockerfile: src/Presentation/FinanceManager.API/Dockerfile
    ports:
      - "5106:8080"
    environment:
      ASPNETCORE_ENVIRONMENT: Development
      ConnectionStrings__DefaultConnection: "Server=sqlserver;Database=BudgetPH_Dev;User Id=sa;Password=${SA_PASSWORD};TrustServerCertificate=true"
      JwtSettings__SecretKey: "${JWT_SECRET}"
    depends_on:
      sqlserver:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  sqlserver:
    image: mcr.microsoft.com/mssql/server:2022-latest
    environment:
      SA_PASSWORD: "${SA_PASSWORD}"
      ACCEPT_EULA: "Y"
      MSSQL_PID: "Developer"
    ports:
      - "1433:1433"
    volumes:
      - sqldata:/var/opt/mssql
    healthcheck:
      test: ["CMD", "/opt/mssql-tools/bin/sqlcmd", "-S", "localhost", "-U", "sa", "-P", "${SA_PASSWORD}", "-Q", "SELECT 1"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  sqldata:
```

---

## Environment Management

```csharp
// appsettings hierarchy (later overrides earlier):
// appsettings.json → appsettings.{Environment}.json → environment variables → command args

// Never commit secrets. Use:
// - Dev: dotnet user-secrets
// - CI/CD: GitHub Secrets → environment variables
// - Production: Azure Key Vault / AWS Secrets Manager

// Access config in code:
var jwtSecret = builder.Configuration["JwtSettings:SecretKey"];
// Maps to: JWTSETTINGS__SECRETKEY env var (double underscore = colon)

// Bind config section to strongly-typed class:
builder.Services.Configure<JwtSettings>(
    builder.Configuration.GetSection(nameof(JwtSettings)));
```

---

## Branching Strategy (GitFlow simplified)

```
main        — production-ready, protected (require PR + review)
develop     — integration branch, deploy to staging on push
feature/*   — new features (branch from develop, PR back to develop)
hotfix/*    — urgent prod fixes (branch from main, PR to main + develop)
release/*   — release preparation (version bump, changelog)
```

---

## GitHub Secrets for CI/CD

```yaml
# Secrets stored in GitHub repo Settings → Secrets and variables → Actions:
# AZURE_WEBAPP_PUBLISH_PROFILE    — from Azure portal (download publish profile)
# AZURE_WEBAPP_PUBLISH_PROFILE_PROD
# SA_PASSWORD                     — SQL Server SA password for compose
# JWT_SECRET                      — JWT signing key
# CODECOV_TOKEN                   — code coverage reporting

# Access in workflow:
env:
  JWT_SECRET: ${{ secrets.JWT_SECRET }}
```

---

## Interview Q&A

**Q1: What is the difference between CI and CD?**
> CI (Continuous Integration) automatically builds and tests every push/PR — ensures code integrates without breaking. CD has two meanings: Continuous Delivery (auto-deploy to staging, manual gate for prod) and Continuous Deployment (auto-deploy all the way to prod on every green build). Most teams do CI + Continuous Delivery, keeping a manual approval for production.

**Q2: Why use multi-stage Docker builds?**
> The SDK image is ~900MB; the runtime image is ~220MB. Multi-stage builds use the SDK only to compile, then copy just the published output to the runtime image. The result: smaller images (faster pulls, less storage), fewer attack vectors (no compiler tools in prod), and better layer caching (SDK layer cached until `.csproj` files change).

**Q3: How do you manage secrets in a CI/CD pipeline?**
> Never commit secrets to source control. In GitHub Actions: store secrets in repo/environment Secrets — they're encrypted at rest and masked in logs. Access via `${{ secrets.MY_SECRET }}`. For Azure: use Managed Identity or OIDC federation — no stored credentials at all. Rotate secrets regularly; use short-lived tokens where possible.

**Q4: What is a deployment gate / environment protection rule?**
> A gate blocks deployment until conditions are met. In GitHub Actions, environment protection rules can require: manual reviewer approval, successful smoke tests, wait timer. This prevents immediate rollout to production — gives teams time to validate staging before promoting the release.
