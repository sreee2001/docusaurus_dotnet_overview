# SentinelBridge

SentinelBridge automatically ingests corporate email (Microsoft 365 / Outlook), classifies requests, correlates conversations to ServiceNow incidents, and creates/updates tickets with configurable rules and notifications.

Key features

- Microsoft Graph (Azure AD SSO-ready) email ingestion
- Pluggable classifier (rule-based now; swap to Azure OpenAI later)
- ServiceNow integration (create/update/close incidents)
- Notification adapters (Teams, SMTP, SMS)
- DI, interfaces, small classes — designed for unit & integration testing

Getting started

1. Clone or create the repo: `git clone <your-repo-url> sentinel-bridge`
2. Build: `dotnet build`
3. Configure `appsettings.json` or environment variables (Azure AD, Graph, ServiceNow, notifications).
4. Run local dev with interactive auth (or use client credentials for daemon mode).

Repository layout

- `SentinelBridge.App` — host, worker, Graph auth, wiring
- `SentinelBridge.Core` — interfaces, models, orchestrator, rule-based classifier
- `SentinelBridge.Tests` — unit tests (xUnit)

Security & operations

- Use Azure Key Vault or environment variables for secrets.
- For production: run as an Azure WebJob / Azure Container Instance / AKS Job using an AAD app with least-privilege Graph permissions.

License: MIT
