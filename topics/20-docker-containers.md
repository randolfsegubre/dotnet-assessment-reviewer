# Topic 20 — Docker & Containers

> Docker is a senior developer requirement. Understanding containers, images, Dockerfiles, and compose is expected in any modern .NET role.

---

## Core Concepts

```
Image    — Blueprint (read-only). Like a class definition.
Container — Running instance of an image. Like an object instance.
Dockerfile — Instructions to build an image.
Registry  — Image storage (Docker Hub, GitHub Container Registry, Azure ACR).
Compose   — Multi-container orchestration via YAML.
```

---

## Dockerfile for .NET (Multi-stage)

Multi-stage builds: build in a full SDK image, run in a tiny runtime image.

```dockerfile
# Stage 1 — Build (full SDK, ~700MB)
FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build
WORKDIR /src

# Copy project files first (layer caching optimization)
COPY ["src/Presentation/AiTutor.API/AiTutor.API.csproj", "src/Presentation/AiTutor.API/"]
COPY ["src/Core/AiTutor.Application/AiTutor.Application.csproj", "src/Core/AiTutor.Application/"]
COPY ["src/Core/AiTutor.Domain/AiTutor.Domain.csproj", "src/Core/AiTutor.Domain/"]
COPY ["src/Infrastructure/AiTutor.Infrastructure/AiTutor.Infrastructure.csproj", "src/Infrastructure/AiTutor.Infrastructure/"]

# Restore (cached unless .csproj files change)
RUN dotnet restore "src/Presentation/AiTutor.API/AiTutor.API.csproj"

# Copy everything else and build
COPY . .
WORKDIR /src/src/Presentation/AiTutor.API
RUN dotnet build "AiTutor.API.csproj" -c Release -o /app/build

# Stage 2 — Publish (optimized)
FROM build AS publish
RUN dotnet publish "AiTutor.API.csproj" -c Release -o /app/publish \
    /p:UseAppHost=false

# Stage 3 — Runtime (tiny ASP.NET runtime image, ~200MB)
FROM mcr.microsoft.com/dotnet/aspnet:10.0 AS final
WORKDIR /app

# Security: don't run as root
RUN adduser --disabled-password --gecos "" appuser
USER appuser

EXPOSE 8080
ENV ASPNETCORE_URLS=http://+:8080
ENV ASPNETCORE_ENVIRONMENT=Production

COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "AiTutor.API.dll"]
```

---

## .dockerignore

```
# Speed up builds by excluding unnecessary files:
**/.git
**/.vs
**/bin
**/obj
**/*.user
**/node_modules
**/.env
**/appsettings.Development.json
**/Dockerfile
**/.dockerignore
```

---

## Docker Compose

```yaml
# docker-compose.yml — local development stack
version: '3.9'

services:
  api:
    build:
      context: .
      dockerfile: src/Presentation/AiTutor.API/Dockerfile
    container_name: aitutor-api
    ports:
      - "5000:8080"
    environment:
      - ASPNETCORE_ENVIRONMENT=Development
      - ConnectionStrings__DefaultConnection=Server=sqlserver;Database=AiTutorDb;User Id=sa;Password=${SQL_PASSWORD};TrustServerCertificate=True
      - JwtSettings__SecretKey=${JWT_SECRET}
    depends_on:
      sqlserver:
        condition: service_healthy
      ollama:
        condition: service_started
    volumes:
      - ./data/resources:/app/data/resources  # mount learning files
    restart: unless-stopped
    networks:
      - aitutor-network

  web:
    build:
      context: src/Presentation/AiTutor.Web
    container_name: aitutor-web
    ports:
      - "3000:80"
    depends_on:
      - api
    networks:
      - aitutor-network

  sqlserver:
    image: mcr.microsoft.com/mssql/server:2022-latest
    container_name: aitutor-sqlserver
    environment:
      - ACCEPT_EULA=Y
      - SA_PASSWORD=${SQL_PASSWORD}
    ports:
      - "1433:1433"
    volumes:
      - sqlserver-data:/var/opt/mssql
    healthcheck:
      test: ["CMD-SHELL", "/opt/mssql-tools18/bin/sqlcmd -S localhost -U sa -P ${SQL_PASSWORD} -Q 'SELECT 1' -No"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s
    networks:
      - aitutor-network

  ollama:
    image: ollama/ollama:latest
    container_name: aitutor-ollama
    ports:
      - "11434:11434"
    volumes:
      - ollama-data:/root/.ollama  # persist downloaded models
    restart: unless-stopped
    networks:
      - aitutor-network
    # For GPU support:
    # deploy:
    #   resources:
    #     reservations:
    #       devices:
    #         - driver: nvidia
    #           count: 1
    #           capabilities: [gpu]

volumes:
  sqlserver-data:
  ollama-data:

networks:
  aitutor-network:
    driver: bridge
```

```bash
# Common commands:
docker compose up -d          # start all services in background
docker compose down           # stop and remove containers
docker compose down -v        # also remove volumes (destroys DB!)
docker compose logs -f api    # follow API logs
docker compose ps             # list running services
docker compose exec api sh    # shell into running container
docker compose build --no-cache  # rebuild without cache
```

---

## Essential Docker Commands

```bash
# Images:
docker build -t aitutor-api:1.0 .          # build image
docker build -t aitutor-api:1.0 -f Dockerfile.prod .
docker images                               # list images
docker pull mcr.microsoft.com/dotnet/sdk:10.0
docker rmi aitutor-api:1.0                  # remove image
docker image prune                          # remove dangling images

# Containers:
docker run -d -p 5000:8080 --name api aitutor-api:1.0  # run detached
docker run --rm -it aitutor-api:1.0 sh     # interactive, remove when done
docker ps                                   # running containers
docker ps -a                                # all containers
docker stop api                             # stop gracefully
docker rm api                               # remove container
docker logs api -f                          # follow logs
docker exec -it api sh                      # shell into running container
docker inspect api                          # container details

# Volumes:
docker volume ls
docker volume create app-data
docker volume rm app-data

# Networks:
docker network ls
docker network create my-network
```

---

## Layer Caching Strategy

```dockerfile
# ❌ Bad — any file change invalidates COPY + restore:
COPY . .
RUN dotnet restore

# ✅ Good — restore is cached unless .csproj files change:
COPY *.sln .
COPY src/**/*.csproj ./src/
# restore only needed when project files change:
RUN dotnet restore
COPY . .   # now copy everything else
RUN dotnet build
```

---

## Environment Variables & Secrets

```dockerfile
# ❌ NEVER put secrets in Dockerfile:
ENV JWT_SECRET=mysecret123

# ✅ Pass at runtime:
docker run -e JWT_SECRET=$JWT_SECRET aitutor-api

# ✅ Use .env file (never commit to git!):
# .env file:
SQL_PASSWORD=StrongP@ss123!
JWT_SECRET=base64-encoded-256-bit-key
```

```yaml
# docker-compose uses .env automatically:
environment:
  - JwtSettings__SecretKey=${JWT_SECRET}  # reads from .env
```

---

## Health Checks in .NET

```csharp
// Program.cs:
builder.Services.AddHealthChecks()
    .AddDbContextCheck<AppDbContext>()
    .AddUrlGroup(new Uri("http://ollama:11434"), "ollama");

app.MapHealthChecks("/health", new HealthCheckOptions
{
    ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse
});
```

```dockerfile
# Reference in Dockerfile:
HEALTHCHECK --interval=30s --timeout=10s --start-period=30s --retries=3 \
    CMD curl -f http://localhost:8080/health || exit 1
```

---

## Interview Q&A

**Q1: What is the difference between a Docker image and a container?**
> An image is an immutable blueprint (template) built from a Dockerfile — it contains the OS, runtime, dependencies, and application code. A container is a running (or stopped) instance of an image — it adds a writable layer on top. Multiple containers can run from the same image simultaneously, each isolated.

**Q2: Why use multi-stage builds?**
> The final image only needs the ASP.NET runtime (~200MB), not the full SDK (~700MB). Multi-stage builds compile/publish in a larger image, then copy only the output to a small runtime image. Result: smaller attack surface, smaller image size, faster deployment.

**Q3: What does `COPY . .` do and why should it come last?**
> It copies all files from the build context to the container. It should come AFTER `COPY *.csproj` + `dotnet restore` because Docker caches each layer. If you copy everything first, ANY file change invalidates the restore cache. Copying project files first means `dotnet restore` is only re-run when `.csproj` files change.

**Q4: How do containers communicate in Docker Compose?**
> Containers in the same Compose network communicate via **service name** as hostname (DNS). So the API container connects to the database at `Server=sqlserver,1433` (not `localhost`) because `sqlserver` is the service name. Docker's internal DNS resolves it automatically.

**Q5: What is a Docker volume and why use it?**
> Volumes persist data outside the container's writable layer. When a container is removed, its writable layer is deleted — volumes survive. Use volumes for: databases, uploaded files, model data (Ollama), logs. Named volumes are managed by Docker; bind mounts map to a host directory.
