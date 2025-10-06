---
slug: azure_cosmos_database
title: Azure Cosmos DB
tags: [dotnet, azure, cosmos, database, nosql]
---

# Azure Cosmos DB

## Short Introduction

Azure Cosmos DB is Microsoft's globally distributed, multi-model NoSQL database service designed for mission-critical applications requiring low latency, elastic scale, and high availability across multiple geographic regions.

## Official Definition

"Azure Cosmos DB is a fully managed NoSQL database for modern app development. Single-digit millisecond response times, and automatic and instant scalability, guarantee speed at any scale."

## Setup and Deployment Steps

### Azure CLI Setup

```bash
# Create Cosmos DB account
az cosmosdb create --name mycosmosdb --resource-group myResourceGroup --kind GlobalDocumentDB --locations regionName="East US" failoverPriority=0 isZoneRedundant=False

# Create database
az cosmosdb sql database create --account-name mycosmosdb --resource-group myResourceGroup --name ToDoList

# Create container
az cosmosdb sql container create --account-name mycosmosdb --resource-group myResourceGroup --database-name ToDoList --name Items --partition-key-path "/category" --throughput 400
```

### Bicep Template

```bicep
resource cosmosAccount 'Microsoft.DocumentDB/databaseAccounts@2023-04-15' = {
  name: 'mycosmosdb'
  location: resourceGroup().location
  kind: 'GlobalDocumentDB'
  properties: {
    consistencyPolicy: {
      defaultConsistencyLevel: 'Session'
    }
    locations: [
      {
        locationName: resourceGroup().location
        failoverPriority: 0
        isZoneRedundant: false
      }
    ]
    databaseAccountOfferType: 'Standard'
  }
}

resource database 'Microsoft.DocumentDB/databaseAccounts/sqlDatabases@2023-04-15' = {
  parent: cosmosAccount
  name: 'ToDoList'
  properties: {
    resource: {
      id: 'ToDoList'
    }
  }
}
```

## Typical Usage and Integration with .NET Apps

### NuGet Package Installation

```xml
<PackageReference Include="Microsoft.Azure.Cosmos" Version="3.35.4" />
```

### Configuration and Service Registration

```csharp
// appsettings.json
{
  "CosmosDb": {
    "Account": "https://mycosmosdb.documents.azure.com:443/",
    "Key": "your-primary-key-here",
    "DatabaseName": "ToDoList",
    "ContainerName": "Items"
  }
}

// Program.cs
using Microsoft.Azure.Cosmos;

var builder = WebApplication.CreateBuilder(args);

// Register Cosmos DB client
builder.Services.AddSingleton<CosmosClient>(serviceProvider =>
{
    var configuration = serviceProvider.GetRequiredService<IConfiguration>();
    var account = configuration["CosmosDb:Account"];
    var key = configuration["CosmosDb:Key"];
    return new CosmosClient(account, key);
});

builder.Services.AddScoped<ICosmosDbService, CosmosDbService>();
```

### Service Implementation

```csharp
public interface ICosmosDbService
{
    Task<T> GetItemAsync<T>(string id, string partitionKey);
    Task<T> CreateItemAsync<T>(T item, string partitionKey);
    Task<T> UpdateItemAsync<T>(string id, T item, string partitionKey);
    Task DeleteItemAsync(string id, string partitionKey);
    Task<IEnumerable<T>> GetItemsAsync<T>(string queryString);
}

public class CosmosDbService : ICosmosDbService
{
    private readonly Container _container;

    public CosmosDbService(CosmosClient cosmosClient, IConfiguration configuration)
    {
        var databaseName = configuration["CosmosDb:DatabaseName"];
        var containerName = configuration["CosmosDb:ContainerName"];
        _container = cosmosClient.GetContainer(databaseName, containerName);
    }

    public async Task<T> GetItemAsync<T>(string id, string partitionKey)
    {
        try
        {
            var response = await _container.ReadItemAsync<T>(id, new PartitionKey(partitionKey));
            return response.Resource;
        }
        catch (CosmosException ex) when (ex.StatusCode == System.Net.HttpStatusCode.NotFound)
        {
            return default(T);
        }
    }

    public async Task<T> CreateItemAsync<T>(T item, string partitionKey)
    {
        var response = await _container.CreateItemAsync(item, new PartitionKey(partitionKey));
        return response.Resource;
    }

    public async Task<IEnumerable<T>> GetItemsAsync<T>(string queryString)
    {
        var query = _container.GetItemQueryIterator<T>(new QueryDefinition(queryString));
        var results = new List<T>();

        while (query.HasMoreResults)
        {
            var response = await query.ReadNextAsync();
            results.AddRange(response.ToList());
        }

        return results;
    }
}

// Model example
public class ToDoItem
{
    [JsonPropertyName("id")]
    public string Id { get; set; } = Guid.NewGuid().ToString();

    [JsonPropertyName("category")]
    public string Category { get; set; }

    [JsonPropertyName("name")]
    public string Name { get; set; }

    [JsonPropertyName("description")]
    public string Description { get; set; }

    [JsonPropertyName("isComplete")]
    public bool IsComplete { get; set; }
}
```

## Use Cases

- Globally distributed applications requiring low latency
- IoT applications with high-volume data ingestion
- Real-time personalization and recommendations
- Content management and catalogs
- Gaming leaderboards and user profiles
- Financial services requiring multi-region compliance

## When to Use vs Alternatives

### Use Azure Cosmos DB when

- Global distribution with low latency is critical
- Elastic scaling from zero to unlimited is needed
- Multiple data models (document, key-value, graph, column) required
- 99.999% availability SLA is important
- Multi-master replication is beneficial

### Don't use when

- Simple relational queries are primary requirement
- Cost optimization is the main concern
- ACID transactions across multiple partitions are critical
- Complex joins and aggregations are common

### Alternatives

- **Azure**: SQL Database, Table Storage
- **AWS**: DynamoDB, DocumentDB
- **GCP**: Firestore, Bigtable
- **Open Source**: MongoDB, Cassandra

## Market Pros/Cons and Cost Considerations

### Pros

- Multi-model database (SQL API, MongoDB API, Cassandra API, etc.)
- Global distribution with automatic failover
- Automatic indexing and schema-agnostic
- Multiple consistency levels
- Serverless and autoscale options

### Cons

- Can be expensive for high-throughput scenarios
- Learning curve for partition key design
- Limited cross-partition transactions
- Query optimization requires understanding of RU consumption

### Cost Considerations

- Charged based on Request Units (RU/s) and storage
- Minimum 400 RU/s for containers (~$24/month)
- Serverless option: Pay per request (good for development/testing)
- Multi-region replication multiplies costs
- Autoscale available to optimize costs for variable workloads
