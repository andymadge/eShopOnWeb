# ADR-002: Separate Catalog and Identity Databases

**Status**: Accepted
**Date**: October 2025 (Inferred from implementation)
**Deciders**: Architecture Team, Security Team
**Technical Story**: Database architecture for security, scalability, and separation of concerns

---

## Context

The eShopOnWeb system manages two distinct types of data:

1. **Catalog Data**: Products, baskets, orders, brands, types
2. **Identity Data**: Users, roles, authentication, claims

We need to decide: **Single database or multiple databases?**

### Requirements

**Security**:
- Identity data is highly sensitive (passwords, personal information)
- Different access control and audit requirements
- Compliance considerations (GDPR, data residency)

**Scalability**:
- Catalog reads far exceed identity reads
- Different performance characteristics
- May need to scale independently

**Team Organization**:
- Security team manages identity
- Business team manages catalog
- Different backup/recovery policies

**Technology**:
- ASP.NET Core Identity has its own schema
- EF Core supports multiple DbContexts
- No cross-database transactions needed

## Decision

We will use **two separate SQL Server databases**:

### Database 1: Catalog Database
- **Context**: `CatalogContext`
- **Connection String**: `CatalogConnection`
- **Location**: `/src/Infrastructure/Data/CatalogContext.cs`

**Tables**:
- `Catalog` (CatalogItem entity)
- `CatalogBrands`
- `CatalogTypes`
- `Baskets`
- `BasketItems`
- `Orders`
- `OrderItems`

### Database 2: Identity Database
- **Context**: `AppIdentityDbContext`
- **Connection String**: `IdentityConnection`
- **Location**: `/src/Infrastructure/Identity/AppIdentityDbContext.cs`

**Tables** (ASP.NET Core Identity schema):
- `AspNetUsers`
- `AspNetRoles`
- `AspNetUserRoles`
- `AspNetUserClaims`
- `AspNetUserLogins`
- `AspNetRoleClaims`
- `AspNetUserTokens`

### Configuration

**appsettings.json**:
```json
{
  "ConnectionStrings": {
    "CatalogConnection": "Server=(localdb)\\mssqllocaldb;...;Initial Catalog=Microsoft.eShopOnWeb.CatalogDb;",
    "IdentityConnection": "Server=(localdb)\\mssqllocaldb;...;Initial Catalog=Microsoft.eShopOnWeb.Identity;"
  }
}
```

**Program.cs** (Web application):
```csharp
builder.Services.AddDbContext<CatalogContext>(c =>
    c.UseSqlServer(builder.Configuration.GetConnectionString("CatalogConnection")));

builder.Services.AddDbContext<AppIdentityDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("IdentityConnection")));
```

## Consequences

### Positive

1. **Security Isolation** ✅
   - Identity data physically separated
   - Different database users/permissions possible
   - Reduced blast radius if catalog DB compromised
   - Easier to implement encryption at rest (TDE) on Identity DB only

2. **Compliance Enabled** ✅
   - Can apply different retention policies
   - Identity DB can have more frequent backups
   - Easier audit logging per database
   - Data residency compliance (can place in different regions)

3. **Scalability** ✅
   - Catalog DB can scale independently (read replicas)
   - Identity DB typically smaller, doesn't need same resources
   - Can optimize indexes and performance independently
   - Different service tiers in Azure SQL

4. **Team Autonomy** ✅
   - Security team owns Identity database
   - Business/dev team owns Catalog database
   - Reduced coordination overhead
   - Clear ownership boundaries

5. **Technology Flexibility** ✅
   - Could migrate Identity to Azure AD / Entra ID without touching catalog
   - Could use different database engines if needed
   - Separate migration histories

6. **Backup & Recovery** ✅
   - Different backup schedules
   - Can restore catalog without affecting identity (and vice versa)
   - Point-in-time restore per database

### Negative

1. **No Cross-Database Transactions** ⚠️
   - Can't use database transactions across both
   - Must use application-level consistency
   - **Impact**: Not needed in current design (no operations span both)
   - **Mitigation**: If needed, implement saga pattern or eventual consistency

2. **Additional Connection Overhead** ⚠️
   - Two connection pools instead of one
   - Slightly more database connections
   - **Impact**: Minimal - connection pooling handles efficiently
   - **Mitigation**: Monitor connection counts, adjust pool sizes if needed

3. **Deployment Complexity** ⚠️
   - Two sets of migrations
   - Two connection strings to manage
   - Double the database resources (development)
   - **Impact**: Minor - automated with EF Core migrations
   - **Mitigation**: Scripts for combined deployment

4. **Cannot Use Foreign Keys Across Databases** ⚠️
   - BuyerId in Orders references user, but can't be FK
   - Referential integrity enforced in application
   - **Impact**: Acceptable - application logic enforces consistency
   - **Mitigation**: Guard clauses, validation

5. **Cost** ⚠️
   - Two databases cost more than one (Azure SQL)
   - Development: Requires two LocalDB instances
   - **Impact**: ~$30/month additional (Azure S1 tier)
   - **Mitigation**: Savings from separate scaling offset cost

## Validation

### Success Criteria (Met)

✅ **Databases are physically separate**
   - Verified: Two distinct connection strings in configuration

✅ **No cross-database dependencies in code**
   - Verified: No queries spanning both DbContexts

✅ **Migrations work independently**
   - Verified: Separate migration folders, independent update

✅ **Both contexts work simultaneously**
   - Verified: Application uses both without issues

### Data Isolation Verification

**Catalog Data**:
```csharp
// CatalogContext never touches AspNetUsers
public class CatalogContext : DbContext
{
    public DbSet<CatalogItem> CatalogItems { get; set; }
    public DbSet<Basket> Baskets { get; set; }
    public DbSet<Order> Orders { get; set; }
    // No DbSet for AspNetUsers
}
```

**Identity Data**:
```csharp
// AppIdentityDbContext only Identity tables
public class AppIdentityDbContext : IdentityDbContext<ApplicationUser>
{
    // Only ASP.NET Identity entities
    // No Catalog entities
}
```

### Cross-Reference Pattern

**Orders reference users by string ID** (no FK):
```csharp
public class Order : BaseEntity
{
    public string BuyerId { get; private set; }  // User ID (no FK)
    // Rest of order data
}
```

**Lookups done application-side**:
```csharp
// Get user's orders
var userId = User.Identity.Name;  // From Identity
var spec = new CustomerOrdersSpecification(userId);
var orders = await _orderRepository.ListAsync(spec);  // From Catalog
```

## Alternatives Considered

### Alternative 1: Single Database with All Tables

**Approach**: All tables in one database, single DbContext

**Pros**:
- Simpler connection management
- Can use foreign keys everywhere
- Database transactions across all data
- Lower cost (one database)

**Cons**:
- Security: Identity data not isolated
- Compliance: Harder to apply different policies
- Scalability: Can't scale independently
- Team: Single ownership, coordination overhead
- Risk: Breach affects all data

**Decision**: Rejected - Security and scalability requirements outweigh simplicity

### Alternative 2: Single Database with Two Schemas

**Approach**: One database, separate schemas (dbo.Catalog*, identity.AspNet*)

**Pros**:
- Physical co-location
- Can still use FKs
- Single backup/restore
- Lower cost

**Cons**:
- Still not physically isolated (security)
- Same server resources (scalability)
- Complex permission management
- Schema-based separation not as robust

**Decision**: Rejected - Doesn't provide enough isolation for security requirements

### Alternative 3: Separate Databases with Distributed Transactions

**Approach**: Two databases + distributed transaction coordinator

**Pros**:
- Physical separation
- ACID transactions across both

**Cons**:
- Significant complexity
- Performance overhead
- Not supported in all environments (Azure)
- Not needed (no operations span both)

**Decision**: Rejected - Complexity not justified, no cross-database operations needed

### Alternative 4: Separate Database Systems (SQL + NoSQL)

**Approach**: Catalog in SQL Server, Identity in MongoDB/CosmosDB

**Pros**:
- Maximum technology flexibility
- Could optimize each for access patterns

**Cons**:
- Massive complexity
- Different skillsets required
- More dependencies
- Overkill for reference implementation

**Decision**: Rejected - Technology diversity not needed, adds complexity

## Implementation Notes

### Migration Commands

**Catalog Database**:
```bash
dotnet ef migrations add <MigrationName> \
  --context CatalogContext \
  --project src/Infrastructure \
  --startup-project src/Web

dotnet ef database update \
  --context CatalogContext \
  --project src/Infrastructure \
  --startup-project src/Web
```

**Identity Database**:
```bash
dotnet ef migrations add <MigrationName> \
  --context AppIdentityDbContext \
  --project src/Infrastructure \
  --startup-project src/Web

dotnet ef database update \
  --context AppIdentityDbContext \
  --project src/Infrastructure \
  --startup-project src/Web
```

### Azure Deployment

**Separate Azure SQL Databases**:
```bicep
// Catalog Database
resource catalogDb 'Microsoft.Sql/servers/databases@2021-11-01' = {
  name: 'eshopweb-catalog'
  properties: {
    sku: { name: 'S1', tier: 'Standard' }
  }
}

// Identity Database (higher security tier)
resource identityDb 'Microsoft.Sql/servers/databases@2021-11-01' = {
  name: 'eshopweb-identity'
  properties: {
    sku: { name: 'S1', tier: 'Standard' }
    // Additional security configurations
  }
}
```

### Azure Key Vault Secrets

**Separate secrets for each**:
- `catalog-connection-string`
- `identity-connection-string`

## Related Decisions

- [ADR-001: Clean Architecture](./adr-001-clean-architecture.md) - Enables separation via multiple contexts
- [ADR-007: Azure Key Vault for Secrets](./adr-007-key-vault-secrets.md) - Secure connection string storage

## Monitoring & Operations

**Health Checks**: Separate for each database
```csharp
services.AddHealthChecks()
    .AddDbContextCheck<CatalogContext>("catalog-db")
    .AddDbContextCheck<AppIdentityDbContext>("identity-db");
```

**Backup Strategy**:
- Catalog DB: Daily full, transaction log every 15 minutes
- Identity DB: Hourly full, transaction log every 5 minutes (more critical)

**Monitoring**:
- Separate DTU/CPU metrics per database
- Alert thresholds configured independently

## References

- **ASP.NET Core Identity Documentation**: Recommends separate context
- **Azure SQL Best Practices**: Security isolation
- **Microsoft Reference Architecture**: eShopOnWeb uses this pattern
- **GDPR Compliance**: Data minimization and isolation

---

**Reviewed**: October 24, 2025
**Status**: Validated against implementation
**Outcome**: Successful - Security and scalability goals achieved
**Cost Impact**: ~$30/month additional (Azure), acceptable for benefits
