---
slug: azure_storage
title: Azure Storage (Blob, Queue, Table)
tags:
  [
    dotnet,
    azure,
    storage,
    blob,
    queue,
    table,
    secure,
    highavailability,
    scalable,
    redundant,
    nosql,
    data_lake,
  ]
---

# Azure Storage (Blob, Queue, Table)

## Short Introduction

Azure Storage is Microsoft's cloud storage solution providing highly available, secure, durable, scalable, and redundant storage. It includes Blob storage for unstructured data, Queue storage for messaging, and Table storage for NoSQL structured data.

## Official Definition

"Azure Storage is a Microsoft-managed service providing cloud storage that is highly available, secure, durable, scalable, and redundant. Azure Storage includes Azure Blobs (objects), Azure Data Lake Storage Gen2, Azure Files, Azure Queues, and Azure Tables."

## Setup and Deployment Steps

### Azure CLI Setup

```bash
# Create storage account
az storage account create --name mystorageaccount --resource-group myResourceGroup --location eastus --sku Standard_LRS --kind StorageV2

# Get connection string
az storage account show-connection-string --name mystorageaccount --resource-group myResourceGroup

# Create blob container
az storage container create --name mycontainer --account-name mystorageaccount

# Create queue
az storage queue create --name myqueue --account-name mystorageaccount

# Create table
az storage table create --name mytable --account-name mystorageaccount
```

## Typical Usage and Integration with .NET Apps

### NuGet Packages

```xml
<PackageReference Include="Azure.Storage.Blobs" Version="12.19.1" />
<PackageReference Include="Azure.Storage.Queues" Version="12.17.1" />
<PackageReference Include="Azure.Data.Tables" Version="12.8.2" />
```

### Blob Storage Service

```csharp
using Azure.Storage.Blobs;
using Azure.Storage.Blobs.Models;

public interface IBlobStorageService
{
    Task<string> UploadBlobAsync(string containerName, string blobName, Stream data);
    Task<Stream> DownloadBlobAsync(string containerName, string blobName);
    Task DeleteBlobAsync(string containerName, string blobName);
    Task<List<string>> ListBlobsAsync(string containerName);
}

public class BlobStorageService : IBlobStorageService
{
    private readonly BlobServiceClient _blobServiceClient;

    public BlobStorageService(BlobServiceClient blobServiceClient)
    {
        _blobServiceClient = blobServiceClient;
    }

    public async Task<string> UploadBlobAsync(string containerName, string blobName, Stream data)
    {
        var containerClient = _blobServiceClient.GetBlobContainerClient(containerName);
        await containerClient.CreateIfNotExistsAsync(PublicAccessType.None);

        var blobClient = containerClient.GetBlobClient(blobName);
        await blobClient.UploadAsync(data, overwrite: true);

        return blobClient.Uri.ToString();
    }

    public async Task<Stream> DownloadBlobAsync(string containerName, string blobName)
    {
        var blobClient = _blobServiceClient.GetBlobContainerClient(containerName).GetBlobClient(blobName);
        var response = await blobClient.DownloadStreamingAsync();
        return response.Value.Content;
    }

    public async Task<List<string>> ListBlobsAsync(string containerName)
    {
        var containerClient = _blobServiceClient.GetBlobContainerClient(containerName);
        var blobs = new List<string>();

        await foreach (var blobItem in containerClient.GetBlobsAsync())
        {
            blobs.Add(blobItem.Name);
        }

        return blobs;
    }
}
```

### Queue Storage Service

```csharp
using Azure.Storage.Queues;
using Azure.Storage.Queues.Models;

public class QueueStorageService
{
    private readonly QueueServiceClient _queueServiceClient;

    public QueueStorageService(QueueServiceClient queueServiceClient)
    {
        _queueServiceClient = queueServiceClient;
    }

    public async Task SendMessageAsync(string queueName, string message)
    {
        var queueClient = _queueServiceClient.GetQueueClient(queueName);
        await queueClient.CreateIfNotExistsAsync();
        await queueClient.SendMessageAsync(message);
    }

    public async Task<QueueMessage[]> ReceiveMessagesAsync(string queueName, int maxMessages = 10)
    {
        var queueClient = _queueServiceClient.GetQueueClient(queueName);
        var response = await queueClient.ReceiveMessagesAsync(maxMessages);
        return response.Value;
    }

    public async Task DeleteMessageAsync(string queueName, string messageId, string popReceipt)
    {
        var queueClient = _queueServiceClient.GetQueueClient(queueName);
        await queueClient.DeleteMessageAsync(messageId, popReceipt);
    }
}
```

### Table Storage Service

```csharp
using Azure.Data.Tables;

public class Customer : ITableEntity
{
    public string PartitionKey { get; set; } = default!;
    public string RowKey { get; set; } = default!;
    public string Name { get; set; } = default!;
    public string Email { get; set; } = default!;
    public DateTimeOffset? Timestamp { get; set; }
    public ETag ETag { get; set; }
}

public class TableStorageService
{
    private readonly TableServiceClient _tableServiceClient;

    public TableStorageService(TableServiceClient tableServiceClient)
    {
        _tableServiceClient = tableServiceClient;
    }

    public async Task<Customer> CreateCustomerAsync(Customer customer)
    {
        var tableClient = _tableServiceClient.GetTableClient("Customers");
        await tableClient.CreateIfNotExistsAsync();

        await tableClient.AddEntityAsync(customer);
        return customer;
    }

    public async Task<Customer> GetCustomerAsync(string partitionKey, string rowKey)
    {
        var tableClient = _tableServiceClient.GetTableClient("Customers");
        var response = await tableClient.GetEntityAsync<Customer>(partitionKey, rowKey);
        return response.Value;
    }

    public async Task<List<Customer>> QueryCustomersAsync(string partitionKey)
    {
        var tableClient = _tableServiceClient.GetTableClient("Customers");
        var customers = new List<Customer>();

        await foreach (var customer in tableClient.QueryAsync<Customer>(filter: $"PartitionKey eq '{partitionKey}'"))
        {
            customers.Add(customer);
        }

        return customers;
    }
}
```

### Service Registration

```csharp
// Program.cs
using Azure.Storage.Blobs;
using Azure.Storage.Queues;
using Azure.Data.Tables;

var builder = WebApplication.CreateBuilder(args);

var connectionString = builder.Configuration.GetConnectionString("AzureStorage");

builder.Services.AddSingleton(x => new BlobServiceClient(connectionString));
builder.Services.AddSingleton(x => new QueueServiceClient(connectionString));
builder.Services.AddSingleton(x => new TableServiceClient(connectionString));

builder.Services.AddScoped<IBlobStorageService, BlobStorageService>();
builder.Services.AddScoped<QueueStorageService>();
builder.Services.AddScoped<TableStorageService>();
```

## Use Cases

### Blob Storage

- Static website hosting
- Document and media storage
- Backup and archival
- Data lakes for analytics
- Content distribution

### Queue Storage

- Decoupling application components
- Background job processing
- Load leveling
- Reliable messaging between services

### Table Storage

- Semi-structured NoSQL data
- Web application data storage
- User data and metadata
- Device information storage
- Logging and telemetry data

## When to Use vs Alternatives

### Use Azure Storage when

- Cost-effective storage for large amounts of data
- Integration with Azure ecosystem is important
- Simple key-value or blob storage requirements
- High durability and availability needed

### Don't use when

- Complex relational queries required
- Strong consistency across partitions needed
- Real-time analytics requirements
- ACID transactions required

### Alternatives

- **AWS**: S3 (Blob), SQS (Queue), DynamoDB (Table)
- **GCP**: Cloud Storage, Cloud Tasks, Firestore
- **Azure alternatives**: Cosmos DB (more features), Service Bus (advanced messaging)

## Market Pros/Cons and Cost Considerations

### Pros

- Very cost-effective storage solution
- High durability (99.999999999% for LRS)
- Multiple access tiers (Hot, Cool, Archive)
- Strong integration with Azure services
- REST API access from any platform

### Cons

- Limited query capabilities (especially Table Storage)
- No ACID transactions across partitions
- Table Storage has limited secondary indexes
- Queue Storage lacks advanced messaging features

### Cost Considerations

- Blob Storage: ~$0.018/GB/month (Hot), ~$0.01/GB/month (Cool), ~$0.002/GB/month (Archive)
- Queue Storage: ~$0.50 per million transactions
- Table Storage: ~$0.50 per million transactions + ~$0.045/GB/month
- Additional charges for operations, bandwidth, and geo-replication
