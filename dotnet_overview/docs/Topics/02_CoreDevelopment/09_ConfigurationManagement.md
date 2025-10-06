---
slug: configuration_management
title: Configuration Management
tags: [dotnet, core, configuration, management]
---

# Configuration Management

## Short Introduction

Configuration management in .NET Core provides a flexible system for managing application settings from various sources like files, environment variables, and cloud services.

## Official Definition

The .NET configuration system provides a key-value based configuration API that works with a variety of configuration providers to supply configuration data to applications.

## Usage

```csharp
// appsettings.json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "ConnectionStrings": {
    "DefaultConnection": "Server=(localdb)\\mssqllocaldb;Database=MyApp;Trusted_Connection=true"
  },
  "EmailSettings": {
    "SmtpServer": "smtp.gmail.com",
    "Port": 587,
    "Username": "user@example.com",
    "Password": "password"
  }
}

// Configuration classes
public class EmailSettings
{
    public string SmtpServer { get; set; } = string.Empty;
    public int Port { get; set; }
    public string Username { get; set; } = string.Empty;
    public string Password { get; set; } = string.Empty;
}

// Program.cs
builder.Services.Configure<EmailSettings>(
    builder.Configuration.GetSection("EmailSettings"));
```

## Configuration Providers

- JSON files (appsettings.json)
- Environment variables
- Command line arguments
- Azure Key Vault
- User secrets (development)

## Use Cases

- Application settings
- Connection strings
- Feature flags
- Environment-specific configuration
- Secrets management

## Sample Usage

```csharp
// Using IOptions pattern
public class EmailService : IEmailService
{
    private readonly EmailSettings _emailSettings;
    private readonly ILogger<EmailService> _logger;

    public EmailService(IOptions<EmailSettings> emailSettings, ILogger<EmailService> logger)
    {
        _emailSettings = emailSettings.Value;
        _logger = logger;
    }

    public async Task SendEmailAsync(string to, string subject, string body)
    {
        using var client = new SmtpClient(_emailSettings.SmtpServer, _emailSettings.Port);
        // Configure and send email
    }
}

// Direct configuration access
public class DatabaseService
{
    private readonly IConfiguration _configuration;

    public DatabaseService(IConfiguration configuration)
    {
        _configuration = configuration;
    }

    public string GetConnectionString()
    {
        return _configuration.GetConnectionString("DefaultConnection")
               ?? throw new InvalidOperationException("Connection string not found");
    }
}
```
