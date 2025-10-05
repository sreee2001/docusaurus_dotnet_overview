## 31. Azure Functions

### Short Introduction + Official Definition

Azure Functions is a serverless compute service that enables you to run event-triggered code without managing infrastructure. It automatically scales based on demand and you only pay for the compute time you consume.

**Official Definition**: "Azure Functions is a serverless solution that allows you to write less code, maintain less infrastructure, and save on costs. Instead of worrying about deploying and maintaining servers, the cloud infrastructure provides all the up-to-date resources needed to keep your applications running."

### Setup and Deployment Steps

**Azure CLI Setup**:

```bash
# Create storage account (required for Functions)
az storage account create --name mystorageaccount --location eastus --resource-group myResourceGroup --sku Standard_LRS

# Create Function App
az functionapp create --resource-group myResourceGroup --consumption-plan-location eastus --runtime dotnet-isolated --runtime-version 8 --functions-version 4 --name myFunctionApp --storage-account mystorageaccount
```

**Local Development Setup**:

```bash
# Install Azure Functions Core Tools
npm install -g azure-functions-core-tools@4 --unsafe-perm true

# Create new Functions project
func init MyFunctionProject --dotnet-isolated --target-framework net8.0
cd MyFunctionProject

# Create HTTP trigger function
func new --name HttpExample --template "HTTP trigger"
```

### Typical Usage and Integration with .NET Apps

**HTTP Trigger Function**:

```csharp
using Microsoft.Azure.Functions.Worker;
using Microsoft.Azure.Functions.Worker.Http;
using Microsoft.Extensions.Logging;
using System.Net;

namespace MyFunctionApp
{
    public class HttpExample
    {
        private readonly ILogger _logger;

        public HttpExample(ILoggerFactory loggerFactory)
        {
            _logger = loggerFactory.CreateLogger<HttpExample>();
        }

        [Function("HttpExample")]
        public async Task<HttpResponseData> Run(
            [HttpTrigger(AuthorizationLevel.Function, "get", "post")] HttpRequestData req)
        {
            _logger.LogInformation("C# HTTP trigger function processed a request.");

            var response = req.CreateResponse(HttpStatusCode.OK);
            response.Headers.Add("Content-Type", "text/plain; charset=utf-8");

            await response.WriteStringAsync("Welcome to Azure Functions!");

            return response;
        }
    }
}
```

**Timer Trigger Function**:

```csharp
[Function("TimerFunction")]
public void Run([TimerTrigger("0 */5 * * * *")] TimerInfo myTimer)
{
    _logger.LogInformation($"C# Timer trigger function executed at: {DateTime.Now}");

    if (myTimer.ScheduleStatus is not null)
    {
        _logger.LogInformation($"Next timer schedule at: {myTimer.ScheduleStatus.Next}");
    }
}
```

**Program.cs Configuration**:

```csharp
using Microsoft.Extensions.Hosting;
using Microsoft.Extensions.DependencyInjection;

var host = new HostBuilder()
    .ConfigureFunctionsWorkerDefaults()
    .ConfigureServices(services =>
    {
        // Add your services here
        services.AddScoped<IMyService, MyService>();
    })
    .Build();

host.Run();
```

### Use Cases

- Event-driven processing (HTTP requests, queue messages, file uploads)
- Scheduled tasks and background processing
- API backends for mobile and web applications
- Data processing pipelines
- IoT data ingestion and processing
- Webhook endpoints for third-party integrations

### When to Use vs Alternatives

**Use Azure Functions when**:

- You have intermittent or unpredictable workloads
- Event-driven architecture is suitable
- You want to minimize infrastructure costs
- Quick prototyping and development is needed

**Don't use when**:

- Long-running processes (15-minute timeout limit)
- High-frequency, consistent workloads
- Complex state management is required
- Low-latency requirements (cold start issues)

**Alternatives**:

- **AWS**: Lambda functions
- **GCP**: Cloud Functions, Cloud Run
- **Traditional**: App Service, Container Instances

### Market Pros/Cons and Cost Considerations

**Pros**:

- True pay-per-execution pricing
- Automatic scaling from zero to thousands
- Multiple trigger types (HTTP, Timer, Storage, Service Bus)
- Integrated with Azure ecosystem

**Cons**:

- Cold start latency
- 15-minute execution timeout
- Limited local state management
- Debugging can be challenging

**Cost Considerations**:

- Consumption plan: Pay per execution and resource consumption
- Premium plan: Pre-warmed instances, no cold starts (~$168/month base)
- Dedicated plan: Run on App Service plan
