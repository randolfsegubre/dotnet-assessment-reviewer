# Topic 21 — Azure for .NET Developers

> Most .NET enterprise workloads run on Azure. Senior .NET developers are expected to understand core Azure services, deployment, and cloud-native patterns.

---

## Core Azure Services for .NET

| Service | Purpose | .NET SDK |
|---|---|---|
| **App Service** | Host ASP.NET Core apps (PaaS) | Deploy via VS / CLI / GitHub Actions |
| **Azure SQL** | Managed SQL Server | Same EF Core connection string |
| **Azure Blob Storage** | Files, images, documents | `Azure.Storage.Blobs` |
| **Azure Service Bus** | Message queues & topics | `Azure.Messaging.ServiceBus` |
| **Azure Key Vault** | Secrets management | `Azure.Extensions.AspNetCore.Configuration.Secrets` |
| **Application Insights** | Monitoring + telemetry | `Microsoft.ApplicationInsights.AspNetCore` |
| **Azure Functions** | Serverless compute | `Microsoft.Azure.Functions.Worker` |
| **Azure Container Apps** | Containerized apps | Docker deploy |
| **Azure SignalR Service** | Managed SignalR scale-out | `Microsoft.Azure.SignalR` |
| **Azure Cognitive Services / OpenAI** | AI/LLM | `Azure.AI.OpenAI` |

---

## Azure App Service — ASP.NET Core Deployment

```bash
# Azure CLI:
az login
az group create --name rg-aitutor --location eastasia

# Create App Service Plan (compute):
az appservice plan create \
    --name plan-aitutor \
    --resource-group rg-aitutor \
    --sku B1 \           # B1=Basic, S1=Standard, P1V3=Premium
    --is-linux

# Create Web App:
az webapp create \
    --name aitutor-api \
    --resource-group rg-aitutor \
    --plan plan-aitutor \
    --runtime "DOTNET|10.0"

# Deploy from local folder:
dotnet publish -c Release -o ./publish
az webapp deploy \
    --resource-group rg-aitutor \
    --name aitutor-api \
    --src-path ./publish \
    --type zip
```

---

## Azure Key Vault — Secrets Management

```csharp
// ❌ Never store secrets in appsettings.json or environment variables in plain text
// ✅ Use Azure Key Vault for production secrets

// Install: Azure.Extensions.AspNetCore.Configuration.Secrets

// Program.cs:
if (builder.Environment.IsProduction())
{
    var keyVaultUri = builder.Configuration["KeyVaultUri"]!;
    var credential = new DefaultAzureCredential(); // uses Managed Identity in Azure
    builder.Configuration.AddAzureKeyVault(new Uri(keyVaultUri), credential);
}

// Key Vault secret names map to config keys:
// Secret name: "JwtSettings--SecretKey" (-- = : in config)
// Accessed as: config["JwtSettings:SecretKey"]

// DefaultAzureCredential chain (tries these in order):
// 1. Environment variables
// 2. Workload Identity
// 3. Managed Identity          ← what runs in Azure (no credentials needed!)
// 4. Visual Studio credential  ← local dev
// 5. Azure CLI credential      ← local dev fallback
```

---

## Azure Blob Storage — File Storage

```csharp
// Install: Azure.Storage.Blobs

// Program.cs:
builder.Services.AddSingleton(x =>
    new BlobServiceClient(builder.Configuration["AzureStorage:ConnectionString"]));

// File upload service:
public class BlobStorageService : IFileStorageService
{
    private readonly BlobServiceClient _blobService;
    private const string ContainerName = "learning-resources";

    public async Task<string> UploadAsync(Stream stream, string fileName, string contentType, CancellationToken ct)
    {
        var container = _blobService.GetBlobContainerClient(ContainerName);
        await container.CreateIfNotExistsAsync(PublicAccessType.None, cancellationToken: ct);
        
        var blobName = $"{Guid.NewGuid()}/{fileName}";
        var blobClient = container.GetBlobClient(blobName);
        
        await blobClient.UploadAsync(stream, new BlobHttpHeaders { ContentType = contentType }, ct);
        
        return blobClient.Uri.ToString();
    }

    public async Task<Stream> DownloadAsync(string blobName, CancellationToken ct)
    {
        var container = _blobService.GetBlobContainerClient(ContainerName);
        var blobClient = container.GetBlobClient(blobName);
        var response = await blobClient.DownloadStreamingAsync(cancellationToken: ct);
        return response.Value.Content;
    }

    public async Task DeleteAsync(string blobName, CancellationToken ct)
    {
        var container = _blobService.GetBlobContainerClient(ContainerName);
        await container.GetBlobClient(blobName).DeleteIfExistsAsync(cancellationToken: ct);
    }
}
```

---

## Azure Service Bus — Message Queuing

```csharp
// Publish a message:
public class ServiceBusPublisher
{
    private readonly ServiceBusClient _client;
    
    public async Task PublishAsync<T>(string queueOrTopicName, T message, CancellationToken ct)
    {
        var sender = _client.CreateSender(queueOrTopicName);
        var json = JsonSerializer.Serialize(message);
        var sbMessage = new ServiceBusMessage(json)
        {
            ContentType = "application/json",
            CorrelationId = Activity.Current?.TraceId.ToString()
        };
        await sender.SendMessageAsync(sbMessage, ct);
    }
}

// Consume messages with a hosted service:
public class TransactionCreatedConsumer : BackgroundService
{
    private readonly ServiceBusClient _client;
    private ServiceBusProcessor? _processor;

    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        _processor = _client.CreateProcessor("transaction-created", new ServiceBusProcessorOptions
        {
            MaxConcurrentCalls = 5,
            AutoCompleteMessages = false
        });
        
        _processor.ProcessMessageAsync += async args =>
        {
            var message = JsonSerializer.Deserialize<TransactionCreatedEvent>(
                args.Message.Body.ToString());
            
            await HandleAsync(message!, args.CancellationToken);
            await args.CompleteMessageAsync(args.Message); // mark as processed
        };
        
        _processor.ProcessErrorAsync += args =>
        {
            _logger.LogError(args.Exception, "Service Bus error on {Entity}", args.EntityPath);
            return Task.CompletedTask;
        };
        
        await _processor.StartProcessingAsync(ct);
    }
}
```

---

## Application Insights — Monitoring

```csharp
// Install: Microsoft.ApplicationInsights.AspNetCore

// Program.cs:
builder.Services.AddApplicationInsightsTelemetry(
    options => options.ConnectionString = builder.Configuration["ApplicationInsights:ConnectionString"]
);

// Automatic tracking: HTTP requests, dependencies, exceptions, performance
// Custom events:
public class TransactionService
{
    private readonly TelemetryClient _telemetry;

    public async Task CreateAsync(CreateTransactionCommand cmd, CancellationToken ct)
    {
        var sw = Stopwatch.StartNew();
        try
        {
            // ... create transaction
            
            _telemetry.TrackEvent("TransactionCreated", new Dictionary<string, string>
            {
                ["type"] = cmd.Type.ToString(),
                ["currency"] = cmd.Currency
            }, new Dictionary<string, double>
            {
                ["amount"] = (double)cmd.Amount
            });
        }
        catch (Exception ex)
        {
            _telemetry.TrackException(ex);
            throw;
        }
        finally
        {
            _telemetry.TrackMetric("TransactionCreateDuration", sw.ElapsedMilliseconds);
        }
    }
}
```

---

## Azure Functions — Serverless

```csharp
// Timer-triggered function (daily at midnight Manila time):
public class DailyReportFunction
{
    [Function("DailyReport")]
    public async Task Run(
        [TimerTrigger("0 0 0 * * *")] TimerInfo timer,
        FunctionContext context)
    {
        var logger = context.GetLogger<DailyReportFunction>();
        logger.LogInformation("Generating daily report at {Time}", DateTime.UtcNow);
        await _reportService.GenerateDailyAsync(CancellationToken.None);
    }
}

// HTTP-triggered function:
public class ProcessDocumentFunction
{
    [Function("ProcessDocument")]
    public async Task<HttpResponseData> Run(
        [HttpTrigger(AuthorizationLevel.Function, "post")] HttpRequestData req,
        FunctionContext context)
    {
        var dto = await req.ReadFromJsonAsync<ProcessDocumentDto>();
        await _documentService.ProcessAsync(dto!);
        
        var response = req.CreateResponse(HttpStatusCode.OK);
        await response.WriteAsJsonAsync(new { message = "Processing started" });
        return response;
    }
}

// Blob-triggered function (auto-process new files):
public class IndexNewResourceFunction
{
    [Function("IndexNewResource")]
    public async Task Run(
        [BlobTrigger("learning-resources/{name}", Connection = "AzureStorage")] byte[] blob,
        string name, FunctionContext context)
    {
        await _indexer.IndexFileAsync(blob, name, CancellationToken.None);
    }
}
```

---

## Azure Container Apps (Recommended for Containerized .NET)

```yaml
# container-app.yaml — deploy Docker container to Azure
properties:
  configuration:
    ingress:
      external: true
      targetPort: 8080
    secrets:
      - name: jwt-secret
        value: $(JWT_SECRET)  # from Key Vault reference
  template:
    containers:
      - name: api
        image: myacr.azurecr.io/aitutor-api:latest
        resources:
          cpu: 0.5
          memory: 1Gi
        env:
          - name: ASPNETCORE_ENVIRONMENT
            value: Production
          - name: JwtSettings__SecretKey
            secretRef: jwt-secret
    scale:
      minReplicas: 1
      maxReplicas: 10
      rules:
        - name: http-scaling
          http:
            metadata:
              concurrentRequests: "100"
```

---

## Interview Q&A

**Q1: What is Managed Identity in Azure and why use it?**
> Managed Identity gives Azure resources (App Service, Container Apps, Functions) an automatic identity without any credentials in code. Your app authenticates to Azure Key Vault, Storage, or SQL using this identity — no connection strings with passwords. `DefaultAzureCredential` automatically uses Managed Identity when running in Azure.

**Q2: What is the difference between Azure Queue Storage and Azure Service Bus?**
> Queue Storage — simple FIFO queue, cheapest, no ordering guarantees, no pub/sub. Service Bus — enterprise messaging with topics/subscriptions (pub/sub), dead-letter queues, message sessions (ordering), duplicate detection, large message support, scheduled delivery. Use Queue Storage for simple tasks; Service Bus for reliable event-driven architectures.

**Q3: What are the tiers of Azure App Service and when would you choose each?**
> Free/Shared — dev/test only, shared compute. Basic (B1-B3) — dev/test with dedicated compute, no auto-scale. Standard (S1-S3) — production, auto-scale, custom domains, SSL. Premium (P1V3+) — high-scale production, VNet integration, better hardware. Isolated — dedicated environment, network isolation.

**Q4: How do you handle secrets in a .NET Azure app without storing them in code?**
> Environment variables → Azure App Service Application Settings (encrypted at rest). Better: Azure Key Vault with Managed Identity — no credentials needed, secrets rotate independently of code. In dev: User Secrets (`dotnet user-secrets`). Never: `appsettings.json` or committed `.env` files.
