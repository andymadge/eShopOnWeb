# Deployment Architecture

**Document Version**: 1.0
**Last Updated**: October 24, 2025

---

## Overview

eShopOnWeb supports multiple deployment models optimized for different environments: local development, Docker containers, and Azure cloud. This document details each deployment strategy, infrastructure requirements, and operational considerations.

## Deployment Models

### 1. Local Development

#### **Option A: Visual Studio / Rider IDE**

**Requirements**:
- .NET 8 SDK
- SQL Server LocalDB (Windows) or SQL Server (Mac/Linux)
- Visual Studio 2022 / JetBrains Rider / VS Code

**Configuration**:
- Connection strings in `appsettings.Development.json`
- Two startup projects: Web + PublicApi
- Ports:
  - Web: https://localhost:44315
  - API: https://localhost:5099

**appsettings.Development.json** (Web):
```json
{
  "ConnectionStrings": {
    "CatalogConnection": "Server=(localdb)\\mssqllocaldb;Integrated Security=true;Initial Catalog=Microsoft.eShopOnWeb.CatalogDb;",
    "IdentityConnection": "Server=(localdb)\\mssqllocaldb;Integrated Security=true;Initial Catalog=Microsoft.eShopOnWeb.Identity;"
  }
}
```

**Database Initialization**:
- Automatic on application startup
- Seeding via `CatalogContextSeed` and `AppIdentityDbContextSeed`
- Default admin user created

#### **Option B: .NET CLI**

```bash
# Terminal 1 - Web Application
cd src/Web
dotnet run

# Terminal 2 - Public API
cd src/PublicApi
dotnet run
```

**Database Setup**:
```bash
# Apply migrations
dotnet ef database update -p src/Infrastructure -s src/Web
```

### 2. Docker Deployment

#### **Docker Compose Configuration**

**File**: `docker-compose.yml`

```yaml
version: '3.4'

services:
  eshopwebmvc:
    image: ${DOCKER_REGISTRY-}eshopwebmvc
    build:
      context: .
      dockerfile: src/Web/Dockerfile
    environment:
      - ASPNETCORE_ENVIRONMENT=Docker
    ports:
      - "5106:80"
      - "5107:443"
    depends_on:
      - sqlserver

  eshoppublicapi:
    image: ${DOCKER_REGISTRY-}eshoppublicapi
    build:
      context: .
      dockerfile: src/PublicApi/Dockerfile
    environment:
      - ASPNETCORE_ENVIRONMENT=Docker
    ports:
      - "5200:80"
      - "5201:443"
    depends_on:
      - sqlserver

  sqlserver:
    image: mcr.microsoft.com/azure-sql-edge
    ports:
      - "1433:1433"
    environment:
      - SA_PASSWORD=@someThingComplicated1234
      - ACCEPT_EULA=Y
```

#### **Docker Commands**

**Build and Run**:
```bash
docker-compose build
docker-compose up
```

**Teardown**:
```bash
docker-compose down
docker-compose down -v  # Include volumes
```

**URLs**:
- Web: http://localhost:5106
- API: http://localhost:5200
- SQL Server: localhost:1433

**appsettings.Docker.json** (Web):
```json
{
  "ConnectionStrings": {
    "CatalogConnection": "Server=sqlserver;Database=Microsoft.eShopOnWeb.CatalogDb;User Id=sa;Password=@someThingComplicated1234;TrustServerCertificate=true",
    "IdentityConnection": "Server=sqlserver;Database=Microsoft.eShopOnWeb.Identity;User Id=sa;Password=@someThingComplicated1234;TrustServerCertificate=true"
  },
  "baseUrls": {
    "apiBase": "http://host.docker.internal:5200/api/",
    "webBase": "http://host.docker.internal:5106/"
  }
}
```

#### **Dockerfile (Web Application)**

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /app

COPY *.sln .
COPY . .
RUN dotnet restore

WORKDIR /app/src/Web
RUN dotnet build -c Release -o /app/build

FROM build AS publish
RUN dotnet publish -c Release -o /app/publish

FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "Web.dll"]
```

**Multi-stage Benefits**:
- Smaller final image (runtime only)
- Build artifacts not included
- Faster deployment

### 3. Azure Cloud Deployment

#### **Infrastructure Components**

**Provisioned Resources**:
1. **Azure App Service (Web)** - Customer-facing application
2. **Azure App Service (API)** - Public API
3. **Azure SQL Database (Catalog)** - Product/order data
4. **Azure SQL Database (Identity)** - User/authentication data
5. **Azure Key Vault** - Secrets management
6. **App Service Plan** - Hosting plan (S1 or higher recommended)
7. **Application Insights** - Monitoring (recommended)

#### **Infrastructure as Code (Bicep)**

**File**: `/infra/main.bicep`

Key resources defined:
- App Service Plans
- App Services (Web + API)
- SQL Server
- SQL Databases (2)
- Key Vault
- Managed Identity assignments
- Connection string secrets

**Deployment**:
```bash
# Login to Azure
az login

# Deploy infrastructure
az deployment sub create \
  --location eastus \
  --template-file infra/main.bicep \
  --parameters infra/main.parameters.json
```

#### **Application Configuration (Azure)**

**Connection Strings**:
- Stored in Azure Key Vault
- Accessed via Managed Identity (no credentials in code)

**Program.cs** (Production configuration):
```csharp
if (!builder.Environment.IsDevelopment())
{
    var credential = new ChainedTokenCredential(
        new AzureDeveloperCliCredential(),
        new DefaultAzureCredential());

    builder.Configuration.AddAzureKeyVault(
        new Uri(builder.Configuration["AZURE_KEY_VAULT_ENDPOINT"]),
        credential);

    builder.Services.AddDbContext<CatalogContext>(c =>
    {
        var connectionString = builder.Configuration[
            builder.Configuration["AZURE_SQL_CATALOG_CONNECTION_STRING_KEY"]];
        c.UseSqlServer(connectionString, sqlOptions =>
        {
            sqlOptions.EnableRetryOnFailure();  // Transient fault handling
        });
    });
}
```

**Key Vault Secrets**:
- `catalog-connection-string` - Catalog database
- `identity-connection-string` - Identity database
- `jwt-secret-key` - API authentication secret

#### **Managed Identity Setup**

**Purpose**: Secure access to Key Vault without storing credentials

**Configuration**:
1. Enable System-Assigned Managed Identity on App Services
2. Grant "Key Vault Secrets User" role to identities
3. Application authenticates automatically

**Benefits**:
- No credentials in application settings
- Automatic credential rotation
- Azure RBAC integration

#### **CI/CD Pipeline (GitHub Actions)**

**File**: `.github/workflows/dotnetcore.yml`

**Workflow**:
1. Checkout code
2. Setup .NET 8 SDK
3. Restore NuGet packages
4. Build solution
5. Run tests
6. Publish artifacts
7. Deploy to Azure App Services

**Example**:
```yaml
name: .NET Core Build and Test

on: [push, pull_request]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Setup .NET
        uses: actions/setup-dotnet@v3
        with:
          dotnet-version: '8.0.x'
      - name: Restore
        run: dotnet restore
      - name: Build
        run: dotnet build --configuration Release
      - name: Test
        run: dotnet test --no-build --verbosity normal
```

### 4. Kubernetes (Future/Optional)

**Not currently implemented, but architecture supports it**

**Manifests would include**:
- Deployments for Web and API
- Services (LoadBalancer/ClusterIP)
- ConfigMaps for configuration
- Secrets for sensitive data
- Ingress for routing
- HPA (Horizontal Pod Autoscaler)

**Considerations**:
- Stateless design ready for scaling
- Would need distributed cache (Redis)
- Database would remain external (Azure SQL)

## Environment Configuration

### Development
- **Database**: LocalDB or SQL Server Express
- **Secrets**: appsettings.Development.json
- **Debugging**: Full symbols, exception details
- **Logging**: Console, Debug output

### Docker
- **Database**: Azure SQL Edge container
- **Secrets**: appsettings.Docker.json (environment variables)
- **Networking**: Docker bridge network
- **Logging**: Console (captured by Docker)

### Azure (Production)
- **Database**: Azure SQL Database with geo-replication
- **Secrets**: Azure Key Vault
- **Identity**: Managed Identity
- **Logging**: Application Insights
- **Monitoring**: Azure Monitor
- **SSL**: Automatic via App Service

## Scaling Strategy

### Vertical Scaling
- **App Service**: Scale up to higher tier (P1V2, P2V2, etc.)
- **Database**: Increase DTUs or move to elastic pool

### Horizontal Scaling
- **App Services**: Scale out to multiple instances
- **Requirements**:
  - Use distributed cache (Redis) instead of MemoryCache
  - Sticky sessions not required (stateless design)
  - Database can handle connection pool from multiple instances

**Configuration**:
```bash
az webapp update --resource-group <rg> --name <app-name> --set numberOfWorkers=3
```

### Database Scaling
- **Read Replicas**: Use for read-heavy queries
- **Elastic Pool**: Share resources across databases
- **Sharding**: Partition by tenant (future consideration)

## Monitoring & Observability

### Health Checks
**Endpoints**:
- `/health` - Overall application health
- `/api_health_check` - API connectivity
- `/home_page_health_check` - Web responsiveness

**Azure Health Probe**:
- Configured on App Service
- Auto-restarts unhealthy instances

### Application Insights Integration

**Configuration**:
```csharp
builder.Services.AddApplicationInsightsTelemetry();
```

**Telemetry Collected**:
- Request/response times
- Dependency calls (SQL, HTTP)
- Exceptions
- Custom events
- User sessions

### Logging

**Development**: Console logger
**Azure**: Application Insights

**Log Levels**:
- **Information**: Startup, key events
- **Warning**: Degraded performance, recoverable errors
- **Error**: Exceptions, failures

## Security Considerations

### Network Security
- **HTTPS Enforced**: All traffic redirected to HTTPS
- **TLS 1.2+**: Minimum TLS version
- **Private Endpoints** (optional): Isolate database from internet

### Secrets Management
- **Never in source control**
- **Development**: User Secrets or appsettings.Development.json (gitignored)
- **Production**: Azure Key Vault

### Database Security
- **Firewall Rules**: Restrict to Azure services
- **Encrypted Connections**: TrustServerCertificate=false (prod)
- **Azure AD Authentication**: Recommended for production
- **Transparent Data Encryption (TDE)**: Enabled on Azure SQL

### Application Security
- **Managed Identity**: No credentials in configuration
- **Key Rotation**: Automatic via Key Vault
- **Principle of Least Privilege**: App only has necessary permissions

## Backup & Disaster Recovery

### Database Backups (Azure SQL)
- **Automated**: Point-in-time restore (7-35 days)
- **Geo-Redundant**: Replicated across regions
- **Long-term Retention**: Up to 10 years

### Application Deployment
- **Blue-Green Deployment**: Use deployment slots
- **Rollback**: Quick revert to previous version
- **Source Control**: Git history for code recovery

### Recovery Scenarios

| Scenario | RTO | RPO | Recovery Method |
|----------|-----|-----|-----------------|
| App Service failure | <5 min | 0 | Auto-restart / failover |
| Database corruption | <1 hour | <5 min | Point-in-time restore |
| Region outage | <4 hours | <15 min | Geo-failover (if configured) |
| Deployment bug | <10 min | 0 | Swap deployment slots |

## Cost Optimization

### Azure Cost Breakdown (Estimated)

| Resource | Tier | Monthly Cost (USD) |
|----------|------|-------------------|
| App Service Plan (S1) | Standard | ~$73 |
| Azure SQL (S1) x2 | Standard | ~$30 x 2 |
| Key Vault | Standard | ~$0.03/operation |
| Application Insights | Basic | Pay-per-GB |
| **Total** | | **~$135-150/month** |

### Optimization Tips
1. **Auto-scale**: Scale down during off-peak hours
2. **Reserved Instances**: Save up to 72% with 1-3 year commitment
3. **Development/Testing**: Use lower tiers (B1)
4. **Database**: Use elastic pool for multiple databases

## Deployment Checklist

### Pre-Deployment
- [ ] Infrastructure provisioned (Bicep)
- [ ] Key Vault secrets configured
- [ ] Managed Identity assigned and permissions granted
- [ ] Connection strings tested
- [ ] SSL certificate configured (automatic on App Service)
- [ ] Database migrations run

### Deployment
- [ ] Build passes
- [ ] All tests pass
- [ ] Publish to staging slot
- [ ] Smoke tests on staging
- [ ] Swap to production
- [ ] Verify health checks

### Post-Deployment
- [ ] Monitor Application Insights for errors
- [ ] Verify health check endpoints
- [ ] Test critical user flows
- [ ] Check database performance
- [ ] Review logs for warnings

---

**Document Status**: ✅ Complete
**Deployment Models**: Local, Docker, Azure
**Production Ready**: Yes (with proper configuration)
