---
slug: docker
title: Docker
tags: [dotnet, containers, docker]
---

# Docker

## Short Introduction

Docker is a containerization platform that packages applications and their dependencies into lightweight, portable containers. For .NET applications, Docker enables consistent deployment across different environments, from development to production, ensuring "it works on my machine" becomes "it works everywhere."

## Official Definition

Docker is an open platform for developing, shipping, and running applications using container virtualization. Docker containers wrap up software in a complete filesystem that contains everything needed to run: code, runtime, system tools, system libraries, and settings.

## Setup/Usage with .NET 8+ Code

### Basic Dockerfile for .NET 8

```dockerfile
# Use the official .NET 8 runtime as base image
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS base
WORKDIR /app
EXPOSE 8080
EXPOSE 8081

# Use SDK image for building
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
ARG BUILD_CONFIGURATION=Release
WORKDIR /src
COPY ["HotelManagement.WebAPI/HotelManagement.WebAPI.csproj", "HotelManagement.WebAPI/"]
COPY ["HotelManagement.Core/HotelManagement.Core.csproj", "HotelManagement.Core/"]
RUN dotnet restore "./HotelManagement.WebAPI/HotelManagement.WebAPI.csproj"
COPY . .
WORKDIR "/src/HotelManagement.WebAPI"
RUN dotnet build "./HotelManagement.WebAPI.csproj" -c $BUILD_CONFIGURATION -o /app/build

FROM build AS publish
ARG BUILD_CONFIGURATION=Release
RUN dotnet publish "./HotelManagement.WebAPI.csproj" -c $BUILD_CONFIGURATION -o /app/publish /p:UseAppHost=false

FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "HotelManagement.WebAPI.dll"]
```

### Docker Compose for multi-service setup:

```yaml
# docker-compose.yml
services:
  hotel-api:
    build:
      context: .
      dockerfile: HotelManagement.WebAPI/Dockerfile
    ports:
      - "5000:8080"
      - "5001:8081"
    environment:
      - ASPNETCORE_ENVIRONMENT=Development
      - ConnectionStrings__DefaultConnection=Server=hotel-db;Database=HotelManagement;User Id=sa;Password=YourPassword123!;TrustServerCertificate=true
    depends_on:
      - hotel-db
    networks:
      - hotel-network

  hotel-db:
    image: mcr.microsoft.com/mssql/server:2022-latest
    environment:
      - ACCEPT_EULA=Y
      - SA_PASSWORD=YourPassword123!
      - MSSQL_PID=Express
    ports:
      - "1433:1433"
    volumes:
      - hotel-db-data:/var/opt/mssql
    networks:
      - hotel-network

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis-data:/data
    networks:
      - hotel-network

volumes:
  hotel-db-data:
  redis-data:

networks:
  hotel-network:
    driver: bridge
```

## Use Cases

- **Consistent Environments**: Eliminate "works on my machine" issues
- **Microservices Deployment**: Package individual services independently
- **CI/CD Pipelines**: Standardized build and deployment process
- **Development Environment**: Quick setup for new team members
- **Cloud Migration**: Easier transition between cloud providers
- **Scaling**: Horizontal scaling with container orchestration

## When to Use vs When Not to Use

### Use Docker when

- Building microservices or distributed applications
- Need consistent deployment across environments
- Working with complex dependency requirements
- Implementing CI/CD pipelines
- Scaling applications horizontally
- Working in team environments

### Consider alternatives when

- Simple single-server applications
- Windows-specific applications requiring full OS features
- Applications with extreme performance requirements
- Limited storage or bandwidth constraints
- Team lacks containerization knowledge

## Market Alternatives & Pros/Cons

### Alternatives:

- **Podman**: Daemonless container engine
- **containerd**: Industry-standard container runtime
- **LXC/LXD**: System containers for Linux
- **Windows Containers**: Native Windows containerization
- **Virtual Machines**: Traditional virtualization

### Pros:

- Lightweight compared to VMs
- Fast startup times
- Consistent environments
- Easy scaling and orchestration
- Large ecosystem and community support
- Efficient resource utilization

### Cons:

- Learning curve for teams
- Additional complexity in simple scenarios
- Storage overhead for images
- Security considerations with shared kernel
- Windows containers have limitations

## Complete Runnable Sample

### Multi-stage Production Dockerfile:

```dockerfile
# HotelManagement.WebAPI/Dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS base
USER app
WORKDIR /app
EXPOSE 8080
EXPOSE 8081

FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
ARG BUILD_CONFIGURATION=Release
WORKDIR /src

# Copy project files and restore dependencies
COPY ["HotelManagement.WebAPI/HotelManagement.WebAPI.csproj", "HotelManagement.WebAPI/"]
COPY ["HotelManagement.Core/HotelManagement.Core.csproj", "HotelManagement.Core/"]
COPY ["HotelManagement.Infrastructure/HotelManagement.Infrastructure.csproj", "HotelManagement.Infrastructure/"]

RUN dotnet restore "HotelManagement.WebAPI/HotelManagement.WebAPI.csproj"

# Copy source code and build
COPY . .
WORKDIR "/src/HotelManagement.WebAPI"
RUN dotnet build "HotelManagement.WebAPI.csproj" -c $BUILD_CONFIGURATION -o /app/build

FROM build AS publish
ARG BUILD_CONFIGURATION=Release
RUN dotnet publish "HotelManagement.WebAPI.csproj" -c $BUILD_CONFIGURATION -o /app/publish /p:UseAppHost=false

FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:8080/health || exit 1

ENTRYPOINT ["dotnet", "HotelManagement.WebAPI.dll"]
```

### Docker Commands

```bash
# Build the image
docker build -t hotel-management-api:latest -f HotelManagement.WebAPI/Dockerfile .

# Run container
docker run -d \
  --name hotel-api \
  -p 5000:8080 \
  -p 5001:8081 \
  -e ASPNETCORE_ENVIRONMENT=Production \
  -e ConnectionStrings__DefaultConnection="Server=host.docker.internal;Database=HotelManagement;Integrated Security=true;TrustServerCertificate=true" \
  hotel-management-api:latest

# View logs
docker logs hotel-api

# Execute commands in container
docker exec -it hotel-api bash

# Build and run with compose
docker-compose up -d

# Scale services
docker-compose up -d --scale hotel-api=3
```

### .dockerignore

```
# Ignore unnecessary files
**/.dockerignore
**/.env
**/.git
**/.gitignore
**/.project
**/.settings
**/.toolstarget
**/.vs
**/.vscode
**/*.*proj.user
**/*.dbmdl
**/*.jfm
**/azds.yaml
**/bin
**/charts
**/docker-compose*
**/Dockerfile*
**/node_modules
**/npm-debug.log
**/obj
**/secrets.dev.yaml
**/values.dev.yaml
LICENSE
README.md
```

### Production-ready docker-compose.override.yml

```yaml
# docker-compose.override.yml (for production)
services:
  hotel-api:
    environment:
      - ASPNETCORE_ENVIRONMENT=Production
      - ASPNETCORE_URLS=https://+:8081;http://+:8080
    restart: unless-stopped
    deploy:
      replicas: 2
      resources:
        limits:
          cpus: "0.5"
          memory: 512M
        reservations:
          cpus: "0.25"
          memory: 256M

  hotel-db:
    restart: unless-stopped
    deploy:
      resources:
        limits:
          cpus: "1.0"
          memory: 2G
        reservations:
          cpus: "0.5"
          memory: 1G

  redis:
    restart: unless-stopped
    command: redis-server --appendonly yes --requirepass yourredispassword
    deploy:
      resources:
        limits:
          cpus: "0.25"
          memory: 256M
```
