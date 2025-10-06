---
slug: azure_key_vault
title: Azure Key Vault
tags: [dotnet, azure, key, vault, secure, storage, secrets, certificates]
---

# Azure Key Vault

## Short Introduction

Azure Key Vault is a cloud service for securely storing and accessing secrets, keys, and certificates. It provides centralized storage for application secrets with hardware security module (HSM) protection and fine-grained access control.

## Official Definition

"Azure Key Vault is a tool for securely storing and accessing secrets. A secret is anything that you want to tightly control access to, such as API keys, passwords, certificates, or cryptographic keys."

## Setup and Deployment Steps

### Azure CLI Setup

```bash
# Create Key Vault
az keyvault create --name mykeyvault --resource-group myResourceGroup --location eastus --enabled-for-template-deployment

# Add secrets
az keyvault secret set --vault-name mykeyvault --name "ConnectionString" --value "Server=myserver;Database=mydb;User Id=myuser;Password=mypass;"

# Add certificate
az keyvault certificate import --vault-name mykeyvault --name "ssl-cert" --file certificate.pfx

# Create managed identity for app
az webapp identity assign --name mywebapp --resource-group myResourceGroup

# Grant access to Key Vault
az keyvault set-policy --name mykeyvault --object-id <managed-identity-object-id> --secret-permissions get list
```

### Bicep Template

```bicep
resource keyVault 'Microsoft.KeyVault/vaults@2023-02-01' = {
  name: 'mykeyvault'
  location: resourceGroup().location
  properties: {
    tenantId: subscription().tenantId
    sku: {
      family: 'A'
      name: 'standard'
    }
    accessPolicies: []
    enabledForTemplateDeployment: true
    enabledForDiskEncryption: true
    enabledForDeployment: true
  }
}

resource secret 'Microsoft.KeyVault/vaults/secrets@2023-02-01' = {
  parent: keyVault
  name: 'ConnectionString'
  properties: {
    value: 'Server=myserver;Database=mydb;Integrated Security=true;'
  }
}
```

## Typical Usage and Integration with .NET Apps

### NuGet Packages

```xml
<PackageReference Include="Azure.Extensions.AspNetCore.Configuration.Secrets" Version="1.3.0" />
<PackageReference Include="Azure.Identity" Version="1.10.3" />
<PackageReference Include="Azure.Security.KeyVault.Secrets" Version="4.5.0" />
```

### Configuration Integration

```csharp
// Program.cs
using Azure.Identity;
using Azure.Extensions.AspNetCore.Configuration.Secrets;

var builder = WebApplication.CreateBuilder(args);

// Add Key Vault configuration
if (!builder.Environment.IsDevelopment())
{
    var keyVaultUri = builder.Configuration["KeyVaultUri"];
    builder.Configuration.AddAzureKeyVault(
        new Uri(keyVaultUri),
        new DefaultAzureCredential());
}

// Alternative: Specific credential
// builder.Configuration.AddAzureKeyVault(
//     new Uri(keyVaultUri),
//     new ManagedIdentityCredential());

var app = builder.Build();
```

### Direct Key Vault Access Service

```csharp
using Azure.Security.KeyVault.Secrets;
using Azure.Identity;

public interface IKeyVaultService
{
    Task<string> GetSecretAsync(string secretName);
    Task SetSecretAsync(string secretName, string secretValue);
    Task<IEnumerable<string>> ListSecretsAsync();
    Task DeleteSecretAsync(string secretName);
}

public class KeyVaultService : IKeyVaultService
{
    private readonly SecretClient _secretClient;
    private readonly ILogger<KeyVaultService> _logger;

    public KeyVaultService(IConfiguration configuration, ILogger<KeyVaultService> logger)
    {
        var keyVaultUri = configuration["KeyVaultUri"];
        _secretClient = new SecretClient(new Uri(keyVaultUri), new DefaultAzureCredential());
        _logger = logger;
    }

    public async Task<string> GetSecretAsync(string secretName)
    {
        try
        {
            var secret = await _secretClient.GetSecretAsync(secretName);
            return secret.Value.Value;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, $"Failed to retrieve secret: {secretName}");
            throw;
        }
    }

    public async Task SetSecretAsync(string secretName, string secretValue)
    {
        try
        {
            await _secretClient.SetSecretAsync(secretName, secretValue);
            _logger.LogInformation($"Secret {secretName} updated successfully");
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, $"Failed to set secret: {secretName}");
            throw;
        }
    }

    public async Task<IEnumerable<string>> ListSecretsAsync()
    {
        var secretNames = new List<string>();
        await foreach (var secret in _secretClient.GetPropertiesOfSecretsAsync())
        {
            secretNames.Add(secret.Name);
        }
        return secretNames;
    }

    public async Task DeleteSecretAsync(string secretName)
    {
        await _secretClient.StartDeleteSecretAsync(secretName);
        _logger.LogInformation($"Secret {secretName} deleted");
    }
}
```

### Using Secrets in Controllers

```csharp
[ApiController]
[Route("api/[controller]")]
public class ConfigController : ControllerBase
{
    private readonly IConfiguration _configuration;
    private readonly IKeyVaultService _keyVaultService;

    public ConfigController(IConfiguration configuration, IKeyVaultService keyVaultService)
    {
        _configuration = configuration;
        _keyVaultService = keyVaultService;
    }

    [HttpGet("database-connection")]
    public async Task<IActionResult> GetDatabaseConnection()
    {
        // This automatically comes from Key Vault if configured
        var connectionString = _configuration.GetConnectionString("DefaultConnection");

        // Or get directly from Key Vault service
        var directSecret = await _keyVaultService.GetSecretAsync("ConnectionString");

        return Ok(new { HasConnection = !string.IsNullOrEmpty(connectionString) });
    }
}
```

### Certificate Management

```csharp
using Azure.Security.KeyVault.Certificates;

public class CertificateService
{
    private readonly CertificateClient _certificateClient;

    public CertificateService(string keyVaultUri)
    {
        _certificateClient = new CertificateClient(new Uri(keyVaultUri), new DefaultAzureCredential());
    }

    public async Task<X509Certificate2> GetCertificateAsync(string certificateName)
    {
        var certificate = await _certificateClient.DownloadCertificateAsync(certificateName);
        return certificate.Value;
    }
}
```

## Use Cases

- Secure storage of connection strings and API keys
- Certificate management for SSL/TLS
- Encryption key management
- Database passwords and service credentials
- OAuth client secrets
- Configuration secrets for different environments

## When to Use vs Alternatives

### Use Azure Key Vault when

- Centralized secret management across multiple applications
- Compliance requirements for secret storage
- Integration with Azure services and managed identities
- Hardware security module (HSM) protection needed
- Audit logging of secret access required

### Don't use when

- Simple applications with few secrets
- Cost optimization is critical for small projects
- Secrets don't need centralized management
- Non-Azure environments primarily

### Alternatives

- **Azure**: App Configuration (for non-secret config), Azure Managed HSM
- **AWS**: AWS Secrets Manager, AWS Systems Manager Parameter Store
- **GCP**: Secret Manager
- **Open Source**: HashiCorp Vault, Kubernetes Secrets

## Market Pros/Cons and Cost Considerations

### Pros

- Hardware security module (HSM) backing
- Integration with Azure Active Directory
- Automatic secret rotation capabilities
- Comprehensive audit logging
- Managed identities eliminate credential management

### Cons

- Additional complexity for simple scenarios
- Network dependency for secret retrieval
- Cost for high-volume secret operations
- Learning curve for proper implementation

### Cost Considerations

- Standard tier: ~$0.03 per 10,000 transactions
- Premium tier (HSM): ~$1.00 per 10,000 transactions + $5/hour per HSM
- Certificate operations: ~$3.00 per certificate request
- No charge for storage of secrets, keys, and certificates
