# Technology Stack

This document provides a comprehensive inventory of all technologies, frameworks, libraries, and tools used in the eShopOnWeb application.

---

## Table of Contents

- [Platform & Runtime](#platform--runtime)
- [Web Frameworks & UI](#web-frameworks--ui)
- [Data Access & Persistence](#data-access--persistence)
- [Authentication & Security](#authentication--security)
- [Architecture & Patterns](#architecture--patterns)
- [API & Documentation](#api--documentation)
- [Mapping & Validation](#mapping--validation)
- [Cloud & Azure Services](#cloud--azure-services)
- [Development & Build Tools](#development--build-tools)
- [Testing Frameworks](#testing-frameworks)
- [Package Management](#package-management)
- [Version Summary](#version-summary)

---

## Platform & Runtime

### .NET Platform

| Technology | Version | Purpose | License |
|------------|---------|---------|---------|
| **.NET** | 8.0 | Cross-platform application framework | MIT |
| **C#** | 12.0 (latest) | Primary programming language | MIT |

**Key Features Used**:
- Nullable reference types
- Record types
- Init-only properties
- Pattern matching
- Async/await
- LINQ

**Target Framework**: `net8.0`

**Runtime Identifiers**:
- `win-x64` - Windows 64-bit
- `linux-x64` - Linux 64-bit
- `osx-x64` - macOS 64-bit

---

## Web Frameworks & UI

### ASP.NET Core

| Technology | Version | Purpose | License |
|------------|---------|---------|---------|
| **ASP.NET Core** | 8.0.2 | Web application framework | MIT |
| **ASP.NET Core MVC** | 8.0 | Model-View-Controller pattern | MIT |
| **Razor Pages** | 8.0 | Page-based web UI | MIT |
| **Minimal APIs** | 8.0 | Lightweight HTTP APIs | MIT |

**Packages**:
```xml
<PackageReference Include="Microsoft.AspNetCore.Mvc" Version="2.2.0" />
<PackageReference Include="Microsoft.AspNetCore.Components.WebAssembly.Server" Version="8.0.2" />
```

### Blazor

| Technology | Version | Purpose | License |
|------------|---------|---------|---------|
| **Blazor WebAssembly** | 8.0.2 | Client-side SPA framework | MIT |
| **Blazor Components** | 8.0.2 | Component model | MIT |

**Packages**:
```xml
<PackageReference Include="Microsoft.AspNetCore.Components.WebAssembly" Version="8.0.2" />
<PackageReference Include="Microsoft.AspNetCore.Components.WebAssembly.DevServer" Version="8.0.2" />
<PackageReference Include="Microsoft.AspNetCore.Components.Authorization" Version="8.0.2" />
<PackageReference Include="Microsoft.AspNetCore.Components.WebAssembly.Authentication" Version="8.0.2" />
```

### Blazor Libraries

| Technology | Version | Purpose | License |
|------------|---------|---------|---------|
| **Blazored.LocalStorage** | 4.5.0 | Browser local storage access | MIT |
| **BlazorInputFile** | 0.2.0 | File upload component | MIT |

---

## Data Access & Persistence

### Entity Framework Core

| Technology | Version | Purpose | License |
|------------|---------|---------|---------|
| **EF Core** | 8.0.2 | Object-relational mapper | MIT |
| **EF Core SQL Server** | 8.0.2 | SQL Server database provider | MIT |
| **EF Core InMemory** | 8.0.2 | In-memory database for testing | MIT |
| **EF Core Tools** | 8.0.2 | Design-time tools & migrations | MIT |

**Packages**:
```xml
<PackageReference Include="Microsoft.EntityFrameworkCore.SqlServer" Version="8.0.2" />
<PackageReference Include="Microsoft.EntityFrameworkCore.InMemory" Version="8.0.2" />
<PackageReference Include="Microsoft.EntityFrameworkCore.Tools" Version="8.0.2" />
<PackageReference Include="Microsoft.AspNetCore.Diagnostics.EntityFrameworkCore" Version="8.0.2" />
```

**EF Core Features Used**:
- Code-First migrations
- Fluent API configuration
- HiLo ID generation
- Owned entities (value objects)
- Shadow properties
- Specification pattern support

### Database

| Technology | Version | Purpose | License |
|------------|---------|---------|---------|
| **SQL Server** | 2019+ | Primary database (production) | Proprietary |
| **SQL Server Express** | 2019+ | Development database | Free |
| **LocalDB** | 2019+ | Developer local database | Free |

**Connection String Example**:
```json
"ConnectionStrings": {
  "CatalogConnection": "Server=(localdb)\\mssqllocaldb;Database=Microsoft.eShopOnWeb.CatalogDb;Trusted_Connection=True;"
}
```

---

## Authentication & Security

### ASP.NET Core Identity

| Technology | Version | Purpose | License |
|------------|---------|---------|---------|
| **ASP.NET Core Identity** | 8.0.2 | User management & authentication | MIT |
| **Identity EntityFrameworkCore** | 8.0.2 | EF Core storage for Identity | MIT |
| **Identity UI** | 8.0.2 | Pre-built Identity pages | MIT |

**Packages**:
```xml
<PackageReference Include="Microsoft.AspNetCore.Identity.EntityFrameworkCore" Version="8.0.2" />
<PackageReference Include="Microsoft.AspNetCore.Identity.UI" Version="8.0.2" />
<PackageReference Include="Microsoft.Extensions.Identity.Core" Version="8.0.2" />
```

### JWT Authentication

| Technology | Version | Purpose | License |
|------------|---------|---------|---------|
| **JWT Bearer** | 8.0.2 | JWT token authentication | MIT |
| **System.IdentityModel.Tokens.Jwt** | 7.3.1 | JWT token handling | MIT |

**Packages**:
```xml
<PackageReference Include="Microsoft.AspNetCore.Authentication.JwtBearer" Version="8.0.2" />
<PackageReference Include="System.IdentityModel.Tokens.Jwt" Version="7.3.1" />
```

### Security Libraries

| Technology | Version | Purpose | License |
|------------|---------|---------|---------|
| **System.Security.Claims** | 4.3.0 | Claims-based security | MIT |

---

## Architecture & Patterns

### Ardalis Libraries

| Technology | Version | Purpose | License |
|------------|---------|---------|---------|
| **Ardalis.Specification** | 7.0.0 | Specification pattern | MIT |
| **Ardalis.Specification.EntityFrameworkCore** | 7.0.0 | EF Core specification evaluator | MIT |
| **Ardalis.Result** | 7.0.0 | Result pattern for error handling | MIT |
| **Ardalis.GuardClauses** | 4.0.1 | Guard clause extension methods | MIT |
| **Ardalis.ApiEndpoints** | 4.1.0 | Endpoint organization | MIT |
| **Ardalis.ListStartupServices** | 1.1.4 | Display registered services | MIT |

**Usage**:
- **Specification Pattern**: Encapsulate query logic in reusable specifications
- **Result Pattern**: Return success/failure without exceptions
- **Guard Clauses**: Input validation with readable syntax
- **API Endpoints**: Organize minimal API endpoints

**Examples**:
```csharp
// Guard Clauses
Guard.Against.Null(basket, nameof(basket));
Guard.Against.NegativeOrZero(price, nameof(price));

// Specification
var spec = new CatalogFilterSpecification(brandId, typeId);
var items = await _repository.ListAsync(spec);

// Result Pattern
var result = await _service.GetItemAsync(id);
if (result.IsSuccess) { /* use result.Value */ }
```

### MediatR

| Technology | Version | Purpose | License |
|------------|---------|---------|---------|
| **MediatR** | 12.0.1 | Mediator pattern implementation | Apache 2.0 |

**Purpose**:
- Decouples request handling
- Supports CQRS pattern
- Pipeline behaviors for cross-cutting concerns

---

## API & Documentation

### OpenAPI / Swagger

| Technology | Version | Purpose | License |
|------------|---------|---------|---------|
| **Swashbuckle.AspNetCore** | 6.5.0 | OpenAPI/Swagger generation | MIT |
| **Swashbuckle.AspNetCore.SwaggerUI** | 6.5.0 | Interactive API documentation | MIT |
| **Swashbuckle.AspNetCore.Annotations** | 6.5.0 | API endpoint annotations | MIT |

**Packages**:
```xml
<PackageReference Include="Swashbuckle.AspNetCore" Version="6.5.0" />
<PackageReference Include="Swashbuckle.AspNetCore.SwaggerUI" Version="6.5.0" />
<PackageReference Include="Swashbuckle.AspNetCore.Annotations" Version="6.5.0" />
```

**Features**:
- Automatic API documentation generation
- Interactive Swagger UI at `/swagger`
- OpenAPI 3.0 specification
- JWT Bearer authentication support

### Minimal API Extensions

| Technology | Version | Purpose | License |
|------------|---------|---------|---------|
| **MinimalApi.Endpoint** | 1.3.0 | Endpoint interface and organization | MIT |

**Purpose**:
- Organize minimal API endpoints in classes
- Dependency injection support
- Route configuration

---

## Mapping & Validation

### AutoMapper

| Technology | Version | Purpose | License |
|------------|---------|---------|---------|
| **AutoMapper** | 12.0.1 | Object-to-object mapping | MIT |
| **AutoMapper.Extensions.Microsoft.DependencyInjection** | 12.0.1 | DI integration | MIT |

**Packages**:
```xml
<PackageReference Include="AutoMapper.Extensions.Microsoft.DependencyInjection" Version="12.0.1" />
```

**Usage**:
- Map entities to DTOs
- Map entities to view models
- Profile-based configuration

**Example**:
```csharp
CreateMap<CatalogItem, CatalogItemDto>();
var dto = _mapper.Map<CatalogItemDto>(catalogItem);
```

### FluentValidation

| Technology | Version | Purpose | License |
|------------|---------|---------|---------|
| **FluentValidation** | 11.9.0 | Validation library | Apache 2.0 |

**Purpose**:
- Strongly-typed validation rules
- Separation of validation logic
- Testable validators

---

## Cloud & Azure Services

### Azure Integration

| Technology | Version | Purpose | License |
|------------|---------|---------|---------|
| **Azure.Extensions.AspNetCore.Configuration.Secrets** | 1.3.1 | Azure Key Vault configuration | MIT |
| **Azure.Identity** | 1.10.4 | Azure authentication | MIT |

**Packages**:
```xml
<PackageReference Include="Azure.Extensions.AspNetCore.Configuration.Secrets" Version="1.3.1" />
<PackageReference Include="Azure.Identity" Version="1.10.4" />
```

**Features**:
- Store secrets in Azure Key Vault
- Managed Identity authentication
- Configuration provider integration

**Usage**:
```csharp
builder.Configuration.AddAzureKeyVault(
    new Uri(keyVaultUri),
    new DefaultAzureCredential());
```

### Container & DevOps

| Technology | Version | Purpose | License |
|------------|---------|---------|---------|
| **Docker** | Latest | Containerization | Apache 2.0 |
| **Visual Studio Container Tools** | 1.19.6 | VS Docker integration | Proprietary |

**Packages**:
```xml
<PackageReference Include="Microsoft.VisualStudio.Azure.Containers.Tools.Targets" Version="1.19.6" />
```

**Containerization**:
- Dockerfile for each service
- Docker Compose for multi-container
- Linux-based containers

---

## Development & Build Tools

### Microsoft Extensions

| Technology | Version | Purpose | License |
|------------|---------|---------|---------|
| **Microsoft.Extensions.Logging.Configuration** | 8.0.0 | Logging configuration | MIT |
| **System.Net.Http.Json** | 8.0.0 | JSON HTTP extensions | MIT |
| **System.Text.Json** | 8.0.3 | JSON serialization | MIT |

### Code Generation

| Technology | Version | Purpose | License |
|------------|---------|---------|---------|
| **Microsoft.VisualStudio.Web.CodeGeneration.Design** | 8.0.0 | Scaffolding tool | MIT |

### Client-Side Tools

| Technology | Version | Purpose | License |
|------------|---------|---------|---------|
| **Microsoft.Web.LibraryManager.Build** | 2.1.175 | LibMan package manager | MIT |
| **BuildBundlerMinifier** | 3.2.449 | CSS/JS bundling and minification | Apache 2.0 |

**Packages**:
```xml
<PackageReference Include="Microsoft.Web.LibraryManager.Build" Version="2.1.175" />
<PackageReference Include="BuildBundlerMinifier" Version="3.2.449" Condition="'$(Configuration)'=='Release'" />
```

---

## Testing Frameworks

### Unit Testing

| Technology | Version | Purpose | License |
|------------|---------|---------|---------|
| **xUnit** | 2.7.0 | Unit testing framework | Apache 2.0 |
| **xunit.runner.visualstudio** | 2.5.6 | Visual Studio test runner | Apache 2.0 |
| **xunit.runner.console** | 2.7.0 | Console test runner | Apache 2.0 |

**Packages**:
```xml
<PackageReference Include="xunit" Version="2.7.0" />
<PackageReference Include="xunit.runner.visualstudio" Version="2.5.6" />
<PackageReference Include="xunit.runner.console" Version="2.7.0" />
```

### Alternative Testing (Some projects)

| Technology | Version | Purpose | License |
|------------|---------|---------|---------|
| **MSTest** | 3.2.2 | Microsoft test framework | MIT |
| **MSTest.TestAdapter** | 3.2.2 | Test adapter | MIT |

### Mocking & Assertions

| Technology | Version | Purpose | License |
|------------|---------|---------|---------|
| **NSubstitute** | 5.1.0 | Mocking framework | BSD |
| **NSubstitute.Analyzers.CSharp** | 1.0.17 | Roslyn analyzers | BSD |

**Packages**:
```xml
<PackageReference Include="NSubstitute" Version="5.1.0" />
<PackageReference Include="NSubstitute.Analyzers.CSharp" Version="1.0.17" />
```

### Integration Testing

| Technology | Version | Purpose | License |
|------------|---------|---------|---------|
| **Microsoft.AspNetCore.Mvc.Testing** | 8.0.2 | Integration test utilities | MIT |
| **Microsoft.NET.Test.Sdk** | 17.9.0 | Test SDK | MIT |

**Packages**:
```xml
<PackageReference Include="Microsoft.AspNetCore.Mvc.Testing" Version="8.0.2" />
<PackageReference Include="Microsoft.NET.Test.Sdk" Version="17.9.0" />
```

### Code Coverage

| Technology | Version | Purpose | License |
|------------|---------|---------|---------|
| **coverlet.collector** | 6.0.2 | Code coverage collector | MIT |

**Package**:
```xml
<PackageReference Include="coverlet.collector" Version="6.0.2" />
```

---

## Package Management

### Central Package Management

The solution uses **Central Package Management** (CPM) to manage NuGet package versions centrally.

**Configuration File**: `Directory.Packages.props`

**Benefits**:
- Single source of truth for versions
- Easier version updates across solution
- Prevents version conflicts

**Structure**:
```xml
<Project>
  <PropertyGroup>
    <ManagePackageVersionsCentrally>true</ManagePackageVersionsCentrally>
    <TargetFramework>net8.0</TargetFramework>
  </PropertyGroup>
  <ItemGroup>
    <PackageVersion Include="PackageName" Version="x.y.z" />
  </ItemGroup>
</Project>
```

### NuGet Feeds

**Primary Feed**: nuget.org (default)

**Restore Sources**:
- https://api.nuget.org/v3/index.json

---

## Version Summary

### Framework Versions

| Component | Version |
|-----------|---------|
| .NET | 8.0 |
| ASP.NET Core | 8.0.2 |
| Entity Framework Core | 8.0.2 |
| C# Language | 12.0 |

### Key Library Versions

| Library | Version |
|---------|---------|
| Ardalis.Specification | 7.0.0 |
| AutoMapper | 12.0.1 |
| MediatR | 12.0.1 |
| Swashbuckle | 6.5.0 |
| xUnit | 2.7.0 |
| NSubstitute | 5.1.0 |

### Version Variables

Defined in `Directory.Packages.props`:

```xml
<AspNetVersion>8.0.2</AspNetVersion>
<SystemExtensionVersion>8.0.0</SystemExtensionVersion>
<EntityFramworkCoreVersion>8.0.2</EntityFramworkCoreVersion>
<VSCodeGeneratorVersion>8.0.0</VSCodeGeneratorVersion>
```

---

## Solution Projects

### Project Structure

| Project | Type | Purpose |
|---------|------|---------|
| **ApplicationCore** | Class Library | Domain entities and interfaces |
| **Infrastructure** | Class Library | Data access and external services |
| **Web** | ASP.NET Core Web App | MVC/Razor Pages front-end |
| **PublicApi** | ASP.NET Core Web API | REST API |
| **BlazorAdmin** | Blazor WebAssembly | Admin SPA |
| **BlazorShared** | Class Library | Shared DTOs and models |
| **UnitTests** | xUnit Test Project | Unit tests |
| **IntegrationTests** | xUnit Test Project | Integration tests |
| **FunctionalTests** | xUnit Test Project | End-to-end tests |
| **PublicApiIntegrationTests** | xUnit Test Project | API integration tests |

### Project Dependencies

```
ApplicationCore (no dependencies)
    ↑
Infrastructure → ApplicationCore
    ↑
Web → ApplicationCore, Infrastructure, BlazorAdmin, BlazorShared
PublicApi → ApplicationCore, Infrastructure
BlazorAdmin → BlazorShared
```

---

## IDE & Development Environment

### Supported IDEs

| IDE | Version | Notes |
|-----|---------|-------|
| **Visual Studio 2022** | 17.8+ | Recommended for Windows |
| **Visual Studio Code** | Latest | Cross-platform |
| **JetBrains Rider** | 2023.3+ | Alternative IDE |

### Required Workloads

**Visual Studio**:
- ASP.NET and web development
- .NET desktop development
- Azure development (optional)

### Extensions & Tools

**Recommended VS Extensions**:
- ReSharper or CodeMaid
- Markdown Editor
- Docker Desktop
- SQL Server Management Studio (SSMS)

**VS Code Extensions**:
- C# (OmniSharp)
- C# Dev Kit
- Azure Tools
- Docker
- REST Client

---

## Runtime Requirements

### Development

| Requirement | Version/Details |
|-------------|----------------|
| .NET SDK | 8.0 or later |
| Node.js | 18 LTS or later (for npm scripts) |
| SQL Server | 2019+ or LocalDB |
| Docker Desktop | Latest (optional) |

### Production

| Requirement | Version/Details |
|-------------|----------------|
| .NET Runtime | 8.0 or later |
| ASP.NET Core Runtime | 8.0 or later |
| SQL Server | 2019+ |
| IIS | 10+ (Windows) or Nginx/Apache (Linux) |
| OS | Windows Server 2019+, Linux (Ubuntu 20.04+), or Docker |

---

## Build & Deployment

### Build Tools

| Tool | Purpose |
|------|---------|
| **MSBuild** | Build orchestration |
| **dotnet CLI** | Build, test, publish commands |
| **NuGet** | Package restore |

### Deployment Options

**Traditional Deployment**:
- IIS (Windows)
- Kestrel behind reverse proxy (Linux)
- Azure App Service
- Azure Container Instances

**Container Deployment**:
- Docker
- Kubernetes
- Azure Container Apps
- Azure Kubernetes Service (AKS)

### CI/CD

**Compatible Platforms**:
- Azure DevOps Pipelines
- GitHub Actions
- Jenkins
- GitLab CI/CD

**Typical Pipeline**:
1. Restore packages
2. Build solution
3. Run unit tests
4. Run integration tests
5. Publish artifacts
6. Deploy to environment

---

## Configuration Management

### Configuration Sources

| Source | Priority | Purpose |
|--------|----------|---------|
| appsettings.json | Low | Default settings |
| appsettings.{Environment}.json | Medium | Environment-specific settings |
| User Secrets | High | Local development secrets |
| Environment Variables | High | Container/server configuration |
| Azure Key Vault | High | Production secrets |
| Command Line | Highest | Override any setting |

### Environment Names

- **Development**: Local developer machines
- **Staging**: Pre-production testing
- **Production**: Live environment

---

## Logging & Monitoring

### Logging

| Technology | Purpose |
|------------|---------|
| **Microsoft.Extensions.Logging** | Logging abstraction |
| **Serilog** | Structured logging (optional) |
| **Application Insights** | Azure monitoring (optional) |

### Logging Providers

- Console (Development)
- Debug (Development)
- EventLog (Windows Production)
- Azure Application Insights (Azure Production)
- File logging (optional, via Serilog)

---

## Security & Compliance

### Security Libraries

| Library | Purpose |
|---------|---------|
| ASP.NET Core Identity | User authentication |
| JWT Bearer | Token authentication |
| Data Protection API | Encrypt sensitive data |

### Security Features

- HTTPS enforcement
- CSRF protection
- XSS protection (Razor encoding)
- SQL injection protection (EF Core parameterization)
- Password hashing (PBKDF2)
- Account lockout
- Role-based authorization

### Compliance

**GDPR Considerations**:
- User data deletion support via Identity
- Privacy policy pages
- Cookie consent

---

## Performance & Scalability

### Performance Libraries

| Library | Purpose |
|---------|---------|
| **Response Caching** | HTTP response caching |
| **Memory Cache** | In-memory caching |
| **Response Compression** | Gzip/Brotli compression |

### Scalability Features

- Stateless architecture
- Horizontal scaling ready
- Database connection pooling
- HiLo ID generation (reduces DB round trips)
- Specification pattern (efficient queries)

---

## Browser Support

### Web Application (MVC/Razor)

| Browser | Minimum Version |
|---------|----------------|
| Chrome | Latest 2 versions |
| Firefox | Latest 2 versions |
| Edge | Latest 2 versions |
| Safari | Latest 2 versions |

### Blazor WebAssembly

| Browser | Minimum Version |
|---------|----------------|
| Chrome | 90+ |
| Firefox | 88+ |
| Edge | 90+ |
| Safari | 14+ |

**Note**: Blazor requires WebAssembly support.

---

## License Summary

### Open Source Licenses

Most technologies used are under permissive open-source licenses:

- **MIT License**: .NET, ASP.NET Core, EF Core, most NuGet packages
- **Apache 2.0**: MediatR, FluentValidation, Docker
- **BSD**: NSubstitute

### Proprietary Components

- SQL Server (requires license for production)
- Visual Studio (requires license, Community edition free for qualifying users)
- Azure services (pay-as-you-go)

---

## Upgrade Path

### Current Version

- .NET 8.0 (LTS until November 2026)

### Future Upgrades

**To .NET 9.0** (November 2024):
- Standard Term Support (STS)
- Support until May 2026
- Recommended for early adopters

**To .NET 10.0** (November 2025):
- Long Term Support (LTS)
- Support until November 2028
- Recommended for production systems

### Upgrade Checklist

1. Update `TargetFramework` in .csproj files
2. Update package versions in `Directory.Packages.props`
3. Review breaking changes documentation
4. Update Docker base images
5. Test thoroughly before deploying

---

## Related Documentation

- [Component Catalog](./component-catalog.md)
- [API Documentation](./api-documentation.md)
- [Data Dictionary](./data-dictionary.md)
- [System Overview](../architecture/system-overview.md)
- [Technical Architecture](../architecture/technical-architecture.md)

---

## Additional Resources

### Official Documentation

- [.NET Documentation](https://docs.microsoft.com/dotnet/)
- [ASP.NET Core Documentation](https://docs.microsoft.com/aspnet/core/)
- [Entity Framework Core Documentation](https://docs.microsoft.com/ef/core/)
- [Blazor Documentation](https://docs.microsoft.com/aspnet/core/blazor/)

### Reference Architectures

- [eShopOnWeb on GitHub](https://github.com/dotnet-architecture/eShopOnWeb)
- [.NET Microservices Architecture eBook](https://docs.microsoft.com/dotnet/architecture/microservices/)
- [ASP.NET Core Architecture eBook](https://docs.microsoft.com/dotnet/architecture/modern-web-apps-azure/)

### Community & Support

- [.NET Foundation](https://dotnetfoundation.org/)
- [ASP.NET Community Standup](https://dotnet.microsoft.com/live/community-standup)
- [Stack Overflow - .NET tag](https://stackoverflow.com/questions/tagged/.net)
