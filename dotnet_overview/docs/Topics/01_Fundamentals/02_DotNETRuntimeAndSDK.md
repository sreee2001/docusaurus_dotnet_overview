---
slug: runtime_vs_sdk
title: .NET Runtime and SDK
tags: [dotnet, runtime, sdk]
---

# .NET Runtime and SDK

## Short Introduction

The .NET Runtime executes .NET applications, while the SDK provides tools for developing, building, and publishing .NET applications.

## Official Definition

- **.NET Runtime**: The execution environment that runs .NET applications
- **.NET SDK**: Software Development Kit containing compiler, tools, and runtime for development

## Usage

```bash
# Check installed versions
dotnet --version
dotnet --list-runtimes
dotnet --list-sdks

# Create new project
dotnet new webapi -n MyApi
dotnet new mvc -n MyWebApp
dotnet new console -n MyConsoleApp
```

## Use Cases

- **Runtime**: Production servers running .NET applications
- **SDK**: Development machines, CI/CD pipelines, build servers

## When to Use / When Not to Use

**Runtime only:**

- Production deployments
- Containers running applications
- Minimal footprint requirements

**SDK:**

- Development environments
- Build servers
- CI/CD pipelines

## Market Alternatives

- Java JDK/JRE
- Node.js runtime
- Python interpreter
- Go compiler/runtime

## Sample Usage

```dockerfile
# Multi-stage Docker build
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /app
COPY *.csproj ./
RUN dotnet restore
COPY . ./
RUN dotnet publish -c Release -o out

FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS runtime
WORKDIR /app
COPY --from=build /app/out .
ENTRYPOINT ["dotnet", "MyApp.dll"]
```
