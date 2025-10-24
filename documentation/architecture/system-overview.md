# System Architecture Overview

**Document Version**: 1.0
**Last Updated**: October 24, 2025
**System Version**: .NET 8.0

---

## Executive Summary

eShopOnWeb is a reference implementation demonstrating **Clean Architecture** and **Domain-Driven Design (DDD)** principles in a practical e-commerce context. Built on ASP.NET Core 8, it showcases enterprise patterns including CQRS, Repository with Specifications, and multiple presentation layers (MVC, Razor Pages, Blazor).

The system serves as both a functional e-commerce platform and an architectural blueprint for building maintainable, testable, and scalable .NET applications.

## System Purpose

### Primary Goals

1. **Demonstrate Clean Architecture**: Clear separation between domain logic, infrastructure, and presentation
2. **Showcase Modern .NET**: Leverage .NET 8 features and ASP.NET Core capabilities
3. **Provide Reference Implementation**: Production-quality code for learning and adaptation
4. **Support Multiple UIs**: MVC, Razor Pages, Blazor Server, and Blazor WebAssembly
5. **Enable API-First Development**: RESTful API with comprehensive documentation

### Business Capabilities

The system provides these core e-commerce capabilities:

| Capability | Description | User Type |
|------------|-------------|-----------|
| **Product Catalog** | Browse products with filtering and search | Customer |
| **Shopping Basket** | Add/remove items, manage quantities | Customer |
| **Order Management** | Place orders, view order history | Customer |
| **Catalog Administration** | CRUD operations on products, brands, types | Administrator |
| **User Management** | Registration, login, profile management | Customer |
| **API Access** | Programmatic access to catalog data | API Client |

## Architecture Style

### Clean Architecture (Layered Architecture)

eShopOnWeb implements **Clean Architecture** (also known as **Onion Architecture** or **Hexagonal Architecture**) with four distinct layers:

```
┌─────────────────────────────────────────────────────┐
│         Presentation Layer (UI/API)                  │
│  - Web (MVC, Razor Pages, Blazor Server)            │
│  - PublicApi (Minimal Endpoints)                    │
│  - BlazorAdmin (WebAssembly SPA)                    │
└─────────────────┬───────────────────────────────────┘
                  │ depends on ↓
┌─────────────────▼───────────────────────────────────┐
│         Application Core (Domain Logic)              │
│  - Domain Entities (Basket, Order, CatalogItem)     │
│  - Domain Services (BasketService, OrderService)    │
│  - Specifications (Query patterns)                  │
│  - Interfaces (Repository, Services)                │
└─────────────────┬───────────────────────────────────┘
                  │ depends on ↓
┌─────────────────▼───────────────────────────────────┐
│         Infrastructure (External Concerns)           │
│  - EF Core DbContexts (Catalog, Identity)           │
│  - Repository Implementation                         │
│  - Identity Services                                │
│  - External Service Integrations                    │
└─────────────────────────────────────────────────────┘
```

**Key Principle**: Dependencies point inward. The Application Core has no dependencies on Infrastructure or Presentation.

### Architecture Pattern: Monolithic with Container Separation

**Classification**: Monolithic application with logical separation into deployable containers

**Containers**:
1. **Web Application** - Customer-facing website
2. **Public API** - RESTful API for external integrations
3. **Blazor Admin SPA** - Admin dashboard (served by Web Application)
4. **SQL Server** - Data persistence layer

**Deployment Options**:
- **Development**: LocalDB or SQL Server in Docker
- **Production**: Azure App Services with Azure SQL Database
- **Docker**: Multi-container deployment via docker-compose

## Key Architectural Characteristics

### 1. Separation of Concerns

**Domain Logic Isolation**:
- Pure business logic in ApplicationCore project
- No infrastructure dependencies (EF, SQL, HTTP)
- Framework-agnostic domain model

**Infrastructure Abstraction**:
- Core defines interfaces
- Infrastructure implements interfaces
- Dependency Injection wires implementations

**Presentation Independence**:
- Multiple UI technologies supported
- UI communicates via domain services and repositories
- No UI logic in domain layer

### 2. Domain-Driven Design (DDD)

**Aggregates**:
- **Basket Aggregate**: Basket (root) + BasketItems
- **Order Aggregate**: Order (root) + OrderItems + Address
- **Catalog Items**: CatalogItem, CatalogBrand, CatalogType

**Aggregate Roots**:
- Mark consistency boundaries
- Implement `IAggregateRoot` interface
- Child entities accessed only through root

**Value Objects**:
- `Address` - Shipping address (immutable)
- `OrderItem` - Order line item (immutable)
- `CatalogItemOrdered` - Product snapshot

**Specifications** (Query Pattern):
- Encapsulate complex queries
- Reusable across different consumers
- Type-safe query building

**Domain Services**:
- `BasketService` - Basket operations
- `OrderService` - Order creation
- Coordinate multiple aggregates

**Repository Pattern**:
- Abstract data access
- Generic repository `IRepository<T>`
- Specification-based queries

### 3. CQRS (Command Query Responsibility Segregation)

**Queries** (via MediatR):
- `GetMyOrders` - Retrieve user's order history
- `GetOrderDetails` - Get specific order details
- Read-only operations

**Commands** (via Services):
- `BasketService.AddItemToBasket()`
- `OrderService.CreateOrderAsync()`
- Write operations

**Separation Benefits**:
- Optimize reads and writes independently
- Different models for display vs. persistence
- Scalability (can separate read/write databases)

### 4. Dependency Injection

**Container**: Built-in ASP.NET Core DI container

**Lifetimes**:
- **Scoped**: Repositories, Services, DbContexts (per request)
- **Singleton**: Configuration objects, UriComposer
- **Transient**: Rarely used (most services are stateless scoped)

**Configuration**:
- Extension methods in `/src/Web/Configuration/`
- `ConfigureCoreServices()` - Domain services
- `ConfigureWebServices()` - Web-specific services

**Testing Benefits**:
- Easy to mock dependencies
- Supports constructor injection throughout

## Technology Stack

### Core Platform
- **.NET 8.0** - Latest LTS release
- **C# 12** - With nullable reference types enabled
- **ASP.NET Core 8** - Web framework

### Presentation Layer
| Technology | Usage |
|------------|-------|
| **ASP.NET Core MVC** | Traditional controller-based views |
| **Razor Pages** | Page-based routing (catalog, basket) |
| **Blazor Server** | Server-side interactive components |
| **Blazor WebAssembly** | Client-side SPA (admin dashboard) |

### Data Access
- **Entity Framework Core 8** - ORM
- **SQL Server** - Primary database
- **EF Core Migrations** - Schema versioning
- **In-Memory Database** - Development/testing

### Architecture Patterns (Libraries)
| Library | Version | Purpose |
|---------|---------|---------|
| **Ardalis.Specification** | 7.0.0 | Specification pattern for queries |
| **MediatR** | 12.0.1 | CQRS implementation |
| **AutoMapper** | 12.0.1 | Object-to-object mapping |
| **Ardalis.GuardClauses** | 4.0.1 | Input validation |
| **Ardalis.Result** | 7.0.0 | Result pattern for error handling |

### Authentication & Security
- **ASP.NET Core Identity** - User management
- **Cookie Authentication** - Web application
- **JWT Bearer Tokens** - API authentication
- **Azure Key Vault** - Production secrets (via Azure.Identity)

### API & Documentation
- **Minimal Endpoints** - API route definitions
- **Swagger/Swashbuckle** (6.5.0) - OpenAPI documentation
- **FluentValidation** (11.9.0) - Input validation

### DevOps & Cloud
- **Docker** - Containerization
- **Azure SQL** - Cloud database
- **Azure Bicep** - Infrastructure as Code
- **GitHub Actions** - CI/CD pipelines

## Quality Attributes

### Maintainability

**Score**: ⭐⭐⭐⭐⭐ (Excellent)

**Evidence**:
- Clear separation of concerns across projects
- SOLID principles followed consistently
- Well-organized folder structure
- Comprehensive test coverage (45 test files)
- Consistent coding style via .editorconfig

**Benefits**:
- Easy to locate code by responsibility
- Changes localized to specific layers
- New developers onboard quickly

### Testability

**Score**: ⭐⭐⭐⭐⭐ (Excellent)

**Evidence**:
- Dependency injection throughout
- Interface-based abstractions
- Pure domain logic (no infrastructure coupling)
- 4 test projects covering different layers
- 45 test files (unit, integration, functional)

**Test Coverage**:
- **Unit Tests**: Domain entities, services, specifications
- **Integration Tests**: Repository operations with real database
- **Functional Tests**: End-to-end scenarios (Web + API)
- **API Integration Tests**: REST endpoint validation

### Scalability

**Score**: ⭐⭐⭐⭐ (Very Good)

**Strengths**:
- Stateless design (no in-process session state)
- Caching strategy (MemoryCache with decorator pattern)
- Async/await throughout
- Container-ready (Docker)
- Database connection pooling via EF Core

**Limitations**:
- Monolithic deployment (single scale unit)
- Shared database (can become bottleneck)
- In-memory cache (not distributed)

**Scale-Out Path**:
1. Deploy multiple Web/API instances behind load balancer
2. Use distributed cache (Redis)
3. Separate read replicas for queries
4. Consider extracting high-volume services (future microservices)

### Performance

**Score**: ⭐⭐⭐⭐ (Very Good)

**Optimizations**:
- Caching decorator for catalog queries
- Eager loading with specifications (avoid N+1)
- Async database operations
- Minimal endpoints (low overhead)
- Static file caching (wwwroot)

**Monitoring**:
- Health check endpoints
- Structured logging
- EF Core query logging (development)

### Security

**Score**: ⭐⭐⭐⭐⭐ (Excellent)

**Implementations**:
- HTTPS enforcement
- Secure cookies (HttpOnly, Secure, SameSite=Lax)
- ASP.NET Core Identity with password policies
- JWT tokens for API
- Input validation (Guard clauses, FluentValidation)
- SQL injection prevention (EF Core parameterization)
- Azure Key Vault for production secrets
- CORS policy (restricted origins)

### Reliability

**Score**: ⭐⭐⭐⭐ (Very Good)

**Resilience Features**:
- Exception handling middleware (API)
- Retry logic for Azure SQL (EnableRetryOnFailure)
- Health checks (database, API, web)
- Guard clauses prevent invalid state
- Transaction support via EF Core

**Monitoring**:
- `/health` endpoint for overall health
- `/api_health_check` for API availability
- `/home_page_health_check` for web availability

### Observability

**Score**: ⭐⭐⭐ (Good)

**Current State**:
- Structured logging via ILogger
- Health check endpoints
- Exception logging
- Development exception page

**Improvement Opportunities**:
- Add distributed tracing (OpenTelemetry)
- Implement Application Insights
- Add metrics collection (Prometheus)
- Enhanced log correlation

## System Context

### Users

**Customer** (Primary Actor)
- Browses product catalog
- Manages shopping basket
- Places orders
- Views order history
- Registers and logs in

**Administrator** (Secondary Actor)
- Manages catalog items (CRUD)
- Uploads product images
- Manages brands and categories
- Monitors system health

**API Client** (System Actor)
- External systems integrating with catalog
- Automated scripts for catalog management
- Mobile applications (future)

### External Systems

**SQL Server Database**
- Two separate databases:
  - **Catalog Database**: Products, baskets, orders
  - **Identity Database**: Users, roles, authentication
- Accessed via Entity Framework Core
- Connection resiliency for Azure SQL

**Azure Key Vault** (Production)
- Stores connection strings securely
- Accessed via Managed Identity
- No credentials in application code

**Email Service** (Abstracted)
- Interface: `IEmailSender`
- Purpose: Order confirmations, notifications
- Implementation: Configured per environment

## Data Flow Overview

### Customer Places Order

```
1. Customer adds items to basket
   → Web UI calls BasketService.AddItemToBasket()
   → Service saves to database via IRepository<Basket>

2. Customer proceeds to checkout
   → Web UI displays basket items
   → Customer provides shipping address

3. Customer confirms order
   → Web UI calls OrderService.CreateOrderAsync()
   → Service loads basket with specification
   → Service loads catalog items for current prices
   → Service creates Order aggregate
   → Service persists order via IRepository<Order>
   → (Optional) Email confirmation sent

4. Customer views order
   → Web UI sends GetOrderDetails query (MediatR)
   → Handler uses CustomerOrdersSpecification
   → Handler retrieves order via IReadRepository<Order>
   → Order details displayed
```

### Admin Updates Catalog

```
1. Admin opens Blazor Admin dashboard
   → Blazor SPA loads from Web Application (static files)
   → SPA fetches data from Public API

2. Admin edits catalog item
   → Blazor SPA sends PUT request to Public API
   → API endpoint validates JWT token
   → Endpoint receives CatalogItemDto
   → AutoMapper maps to CatalogItem entity
   → Entity updated via IRepository<CatalogItem>
   → Changes committed to database

3. Changes visible immediately
   → Customer browsing catalog sees updated data
   → Cache invalidated (if caching enabled)
```

### API Client Integration

```
1. Client authenticates
   → POST /api/auth with username/password
   → API validates credentials against Identity database
   → IdentityTokenClaimService generates JWT
   → Token returned to client

2. Client requests catalog data
   → GET /api/catalog-items with Authorization header
   → JWT middleware validates token
   → Endpoint creates CatalogFilterPaginatedSpecification
   → Repository applies specification to DbContext
   → EF Core generates SQL query
   → Results mapped to CatalogItemDto
   → JSON response returned

3. Client creates catalog item
   → POST /api/catalog-items with JWT
   → Endpoint validates input
   → New CatalogItem entity created
   → Saved via IRepository<CatalogItem>
   → Success response returned
```

## Project Structure

### Solution Organization

```
eShopOnWeb.sln
├── src/
│   ├── ApplicationCore/          # Domain layer (no dependencies)
│   ├── Infrastructure/           # Data access, external services
│   ├── Web/                      # Customer-facing web app
│   ├── PublicApi/                # REST API
│   ├── BlazorAdmin/              # Admin SPA
│   └── BlazorShared/             # Shared Blazor components
├── tests/
│   ├── UnitTests/                # Domain logic tests
│   ├── IntegrationTests/         # Repository tests
│   ├── FunctionalTests/          # End-to-end tests
│   └── PublicApiIntegrationTests/ # API tests
├── docker-compose.yml            # Container orchestration
└── infra/                        # Azure Bicep IaC
```

### Dependency Graph

```
Web ──────────┐
PublicApi ────┼──→ ApplicationCore ──→ BlazorShared
BlazorAdmin ──┘           ↑
                          │
                   Infrastructure
```

**Rules**:
- ApplicationCore has NO dependencies on other projects
- Infrastructure depends on ApplicationCore (implements interfaces)
- Presentation projects depend on ApplicationCore + Infrastructure
- No circular dependencies

## Deployment Models

### Local Development

**Option 1: Visual Studio / Rider**
- SQL Server LocalDB (Windows)
- Multiple startup projects (Web + PublicApi)
- Port 44315 (Web), 5099 (API)

**Option 2: Docker Compose**
```bash
docker-compose up
```
- Azure SQL Edge container
- Web application container
- Public API container
- All connected via Docker network

### Azure Production

**Infrastructure**:
- **Web Application**: Azure App Service (Web tier)
- **Public API**: Azure App Service (API tier)
- **Databases**: Azure SQL Database (2 databases)
- **Secrets**: Azure Key Vault
- **Identity**: Managed Identity for Key Vault access

**Configuration**:
- Connection strings from Key Vault
- Environment-specific appsettings
- HTTPS enforced
- Monitoring via Application Insights (recommended)

## Architectural Decisions Summary

Key decisions (see [ADRs](../decisions/) for full rationale):

1. **Clean Architecture** - Maintainability and testability over simplicity
2. **Separate Databases** - Security isolation (Catalog vs Identity)
3. **Multiple UI Technologies** - Demonstrate flexibility and modern approaches
4. **Minimal Endpoints for API** - Cleaner than traditional controllers
5. **Specification Pattern** - Type-safe query encapsulation
6. **MediatR for CQRS** - Decouple queries from presentation
7. **Repository Pattern** - Abstract data access
8. **Guard Clauses** - Explicit validation at boundaries

## Next Steps for Understanding

1. **For Developers**: Review [C4 Container Diagram](../diagrams/c4-container.mermaid) and [Component Catalog](../reference/component-catalog.md)
2. **For Architects**: Read [Technical Architecture](./technical-architecture.md) and [ADRs](../decisions/)
3. **For DevOps**: Check [Deployment Architecture](./deployment-architecture.md)
4. **For API Consumers**: See [API Documentation](../reference/api-documentation.md)

---

**Document Status**: ✅ Complete
**Confidence Level**: High (verified against codebase)
**Last Verified**: October 24, 2025
