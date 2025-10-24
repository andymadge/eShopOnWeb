# ADR-001: Adopt Clean Architecture Pattern

**Status**: Accepted
**Date**: October 2025 (Inferred from implementation)
**Deciders**: Architecture Team
**Technical Story**: Establish maintainable, testable architecture for e-commerce platform

---

## Context

The eShopOnWeb project requires an architecture that supports:
- **Long-term maintainability** - Code must be understandable and modifiable over years
- **Testability** - Business logic must be testable without infrastructure dependencies
- **Technology flexibility** - Ability to change UI frameworks, databases, or external services
- **Team scalability** - Clear boundaries for multiple developers/teams
- **Educational value** - Demonstrate industry best practices for .NET applications

Traditional layered architectures often lead to:
- Business logic scattered across UI and data layers
- Tight coupling to specific frameworks (EF Core, ASP.NET)
- Difficulty testing without full infrastructure
- Changes rippling across multiple layers

## Decision

We will implement **Clean Architecture** (also known as Onion Architecture or Hexagonal Architecture) with the following structure:

### Project Structure
```
ApplicationCore (Domain Layer)
    ↑ no dependencies
Infrastructure (Data/External Services)
    ↓ depends on ApplicationCore
Presentation (Web, PublicApi, BlazorAdmin)
    ↓ depends on ApplicationCore + Infrastructure
```

### Key Principles
1. **Dependency Inversion**: ApplicationCore defines interfaces, Infrastructure implements them
2. **Domain Independence**: ApplicationCore has zero infrastructure dependencies
3. **Framework Agnosticism**: Domain logic is pure C# (no EF, ASP.NET, etc.)
4. **Explicit Dependencies**: Constructor injection throughout, no static dependencies

### Implementation Details

**ApplicationCore Project**:
- Domain entities (Basket, Order, CatalogItem)
- Domain services (BasketService, OrderService)
- Interfaces (IRepository, IEmailSender, IAppLogger)
- Specifications (query patterns)
- **Dependencies**: None (only .NET BCL)

**Infrastructure Project**:
- EF Core DbContexts
- Repository implementations
- Identity services
- External service integrations
- **Dependencies**: ApplicationCore, EF Core, SQL Server provider

**Presentation Projects** (Web, PublicApi, BlazorAdmin):
- Controllers, Pages, Endpoints
- View Models
- UI-specific services
- **Dependencies**: ApplicationCore, Infrastructure

## Consequences

### Positive

1. **Testability Achieved**
   - ApplicationCore can be tested without any infrastructure
   - Mock implementations easy to create
   - Fast unit tests (no database required)
   - Evidence: 45 test files covering domain logic

2. **Maintainability Improved**
   - Clear separation of concerns
   - Easy to locate code by responsibility
   - Changes localized to specific layers
   - New developers onboard quickly

3. **Flexibility Enabled**
   - Can swap EF Core for another ORM
   - Can add new UI technologies (e.g., mobile)
   - Can change databases without touching domain
   - Framework upgrades isolated to presentation/infrastructure

4. **Business Logic Protected**
   - Domain rules encapsulated in entities
   - No accidental coupling to infrastructure
   - Clear boundaries enforce design

5. **Team Scalability**
   - Domain team can work independently
   - Infrastructure team owns data access
   - UI teams can work in parallel

### Negative

1. **Initial Complexity**
   - More projects (3+ instead of 1)
   - More interfaces to define
   - Steeper learning curve for junior developers
   - **Mitigation**: Comprehensive documentation, clear patterns

2. **Additional Abstractions**
   - Repository pattern adds indirection
   - Interface definitions increase code volume
   - More files to navigate
   - **Mitigation**: Consistent naming, good IDE support

3. **Potential Over-Engineering**
   - Simple CRUD operations may feel verbose
   - Interfaces for small services may seem unnecessary
   - **Mitigation**: Pragmatic application, not dogmatic

4. **Cross-Cutting Concerns**
   - Logging, validation span multiple layers
   - Need careful design to avoid duplication
   - **Mitigation**: Interface abstractions (IAppLogger), decorator pattern

## Validation

### Success Criteria (Met)

✅ **Domain logic has no infrastructure dependencies**
   - Verified: ApplicationCore.csproj has no EF Core or ASP.NET references

✅ **High test coverage without infrastructure**
   - Verified: Unit tests in UnitTests project test domain without database

✅ **Easy to swap implementations**
   - Verified: Can use in-memory database for tests, SQL Server for production

✅ **Clear code organization**
   - Verified: Each project has single, well-defined responsibility

### Metrics

| Metric | Target | Actual | Status |
|--------|--------|--------|--------|
| **Projects with clear boundaries** | 3+ | 6 | ✅ Exceeded |
| **Domain dependencies** | 0 | 0 | ✅ Met |
| **Test projects** | 2+ | 4 | ✅ Exceeded |
| **Interface abstractions** | 10+ | 10+ | ✅ Met |

### Code Evidence

**ApplicationCore Independence** (`ApplicationCore.csproj`):
```xml
<ItemGroup>
  <PackageReference Include="Ardalis.GuardClauses" Version="4.0.1" />
  <PackageReference Include="Ardalis.Result" Version="7.0.0" />
  <PackageReference Include="Ardalis.Specification" Version="7.0.0" />
  <!-- NO EF Core, NO ASP.NET Core -->
</ItemGroup>
```

**Dependency Injection** (Program.cs):
```csharp
// Infrastructure implements ApplicationCore interfaces
builder.Services.AddScoped(typeof(IRepository<>), typeof(EfRepository<>));
builder.Services.AddScoped<IBasketService, BasketService>();
builder.Services.AddScoped<IOrderService, OrderService>();
```

## Alternatives Considered

### Alternative 1: Traditional N-Tier Architecture

**Approach**: Presentation → Business Logic → Data Access (each dependent on the one below)

**Pros**:
- Simpler, fewer projects
- Familiar to most developers
- Less abstraction overhead

**Cons**:
- Business logic often ends up in UI or data layer
- Hard to test (requires database)
- Coupling to specific frameworks
- Changes ripple through layers

**Decision**: Rejected - Long-term maintainability more important than initial simplicity

### Alternative 2: Single-Project Modular Monolith

**Approach**: All code in one project, organized by folders

**Pros**:
- Simplest structure
- Fast to navigate
- No project reference issues

**Cons**:
- No enforcement of boundaries (easy to violate)
- Can't separate dependencies
- Hard to extract modules later
- Not clear for demonstration purposes

**Decision**: Rejected - Lack of enforced boundaries would lead to coupling

### Alternative 3: Microservices from Day 1

**Approach**: Separate services for Catalog, Basket, Orders

**Pros**:
- Maximum isolation
- Independent deployment
- Technology flexibility per service

**Cons**:
- Massive complexity overhead
- Distributed system challenges (latency, consistency)
- Overkill for reference implementation
- Harder to understand

**Decision**: Rejected - Complexity not justified for current scale; Clean Architecture provides migration path if needed

## Related Decisions

- [ADR-002: Separate Catalog and Identity Databases](./adr-002-separate-databases.md) - Logical separation of concerns
- [ADR-003: Repository Pattern with Specifications](./adr-003-repository-specifications.md) - Data access abstraction
- [ADR-006: Dependency Injection Throughout](./adr-006-dependency-injection.md) - Enables Clean Architecture

## References

- **Clean Architecture** by Robert C. Martin (Uncle Bob)
- **Onion Architecture** by Jeffrey Palermo
- **Hexagonal Architecture** by Alistair Cockburn
- **Microsoft eShopOnWeb** (original reference implementation)

## Notes

This decision establishes the foundation for all other architectural decisions. The success of this approach is evident in:
- Comprehensive test coverage (45 test files)
- Clear code organization (6 well-defined projects)
- Educational value (demonstrates best practices)

The slight increase in initial complexity is more than offset by long-term benefits in maintainability, testability, and flexibility.

---

**Reviewed**: October 24, 2025
**Status**: Validated against implementation
**Outcome**: Successful - Architecture achieves stated goals
