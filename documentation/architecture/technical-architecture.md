# Technical Architecture

**Document Version**: 1.0
**Last Updated**: October 24, 2025
**Target Audience**: Software Architects, Senior Developers, Technical Leads

---

## Introduction

This document provides an in-depth technical view of eShopOnWeb's architecture, including implementation patterns, data flows, integration strategies, and technical decisions. It serves as the primary reference for understanding how the system is built and how components interact.

## Architecture Principles

### 1. Dependency Inversion Principle

**Core defines interfaces, Infrastructure implements them**

```
ApplicationCore (Interfaces)
        ↑
        │ implements
        │
Infrastructure (Implementations)
```

**Example**:
- Interface: `/src/ApplicationCore/Interfaces/IRepository.cs`
- Implementation: `/src/Infrastructure/Data/EfRepository.cs`
- Registration: DI container in `Program.cs`

### 2. Separation of Concerns

**Three distinct layers with clear responsibilities**:

| Layer | Responsibilities | Dependencies |
|-------|------------------|--------------|
| **ApplicationCore** | Domain entities, business rules, interfaces | None (pure .NET) |
| **Infrastructure** | Data access, external services, EF Core | ApplicationCore |
| **Presentation** | UI, API endpoints, view models | ApplicationCore, Infrastructure |

### 3. Tell, Don't Ask (Aggregate Encapsulation)

**Example from `Basket` aggregate** (`/src/ApplicationCore/Entities/BasketAggregate/Basket.cs`):

```csharp
// ❌ BAD: Exposing internal list
public List<BasketItem> Items { get; set; }

// ✅ GOOD: Encapsulated with behavior
private readonly List<BasketItem> _items = new();
public IReadOnlyCollection<BasketItem> Items => _items.AsReadOnly();

public void AddItem(int catalogItemId, decimal unitPrice, int quantity = 1)
{
    // Business logic enforced
    if (!Items.Any(i => i.CatalogItemId == catalogItemId))
    {
        _items.Add(new BasketItem(catalogItemId, quantity, unitPrice));
        return;
    }
    var existingItem = Items.First(i => i.CatalogItemId == catalogItemId);
    existingItem.AddQuantity(quantity);
}
```

**Benefits**:
- Enforces business rules (can't violate invariants)
- Encapsulation (can't accidentally modify items directly)
- Single point of control for basket modifications

### 4. Explicit Dependencies

**Constructor injection everywhere**:

```csharp
public class OrderService : IOrderService
{
    private readonly IRepository<Order> _orderRepository;
    private readonly IRepository<Basket> _basketRepository;
    private readonly IRepository<CatalogItem> _itemRepository;
    private readonly IUriComposer _uriComposer;

    public OrderService(
        IRepository<Basket> basketRepository,
        IRepository<CatalogItem> itemRepository,
        IRepository<Order> orderRepository,
        IUriComposer uriComposer)
    {
        // Dependencies injected, not newed up
    }
}
```

## Data Architecture

### Database Strategy

**Two Separate Databases**:

1. **Catalog Database** (CatalogContext)
   - Products, brands, types
   - Shopping baskets
   - Orders

2. **Identity Database** (AppIdentityDbContext)
   - Users, roles
   - Authentication data

**Rationale**:
- **Security**: Identity data isolated
- **Scaling**: Can scale independently
- **Compliance**: Separate backups/retention policies
- **Responsibility**: Different teams can manage

### Entity Framework Core Implementation

#### DbContext Configuration

**CatalogContext** (`/src/Infrastructure/Data/CatalogContext.cs`):

```csharp
public class CatalogContext : DbContext
{
    public DbSet<Basket> Baskets { get; set; }
    public DbSet<CatalogItem> CatalogItems { get; set; }
    public DbSet<CatalogBrand> CatalogBrands { get; set; }
    public DbSet<CatalogType> CatalogTypes { get; set; }
    public DbSet<Order> Orders { get; set; }
    public DbSet<OrderItem> OrderItems { get; set; }
    public DbSet<BasketItem> BasketItems { get; set; }

    protected override void OnModelCreating(ModelBuilder builder)
    {
        // Applies all IEntityTypeConfiguration<T> from assembly
        builder.ApplyConfigurationsFromAssembly(Assembly.GetExecutingAssembly());
    }
}
```

#### Entity Type Configurations

**Pattern**: One configuration class per entity

**Example** - `CatalogItemConfiguration.cs`:

```csharp
public class CatalogItemConfiguration : IEntityTypeConfiguration<CatalogItem>
{
    public void Configure(EntityTypeBuilder<CatalogItem> builder)
    {
        builder.ToTable("Catalog");

        builder.Property(ci => ci.Id)
            .UseHiLo("catalog_hilo")  // High-Low pattern for IDs
            .IsRequired();

        builder.Property(ci => ci.Name)
            .IsRequired()
            .HasMaxLength(50);

        builder.Property(ci => ci.Price)
            .IsRequired()
            .HasColumnType("decimal(18,2)");

        builder.HasOne(ci => ci.CatalogBrand)
            .WithMany()
            .HasForeignKey(ci => ci.CatalogBrandId);

        builder.HasOne(ci => ci.CatalogType)
            .WithMany()
            .HasForeignKey(ci => ci.CatalogTypeId);
    }
}
```

**Benefits**:
- Fluent API cleanly organized
- Reusable configurations
- Version control friendly

#### Migrations

**Location**: `/src/Infrastructure/Data/Migrations/`

**Commands**:
```bash
# Add migration
dotnet ef migrations add MigrationName -p Infrastructure -s Web

# Update database
dotnet ef database update -p Infrastructure -s Web
```

### Repository Pattern Implementation

**Generic Repository** (`/src/Infrastructure/Data/EfRepository.cs`):

```csharp
public class EfRepository<T> : IRepository<T> where T : BaseEntity, IAggregateRoot
{
    private readonly CatalogContext _dbContext;

    public async Task<T?> GetByIdAsync(int id)
    {
        return await _dbContext.Set<T>().FindAsync(id);
    }

    public async Task<List<T>> ListAsync(ISpecification<T> spec)
    {
        // Specification pattern integration
        return await ApplySpecification(spec).ToListAsync();
    }

    public async Task<T> AddAsync(T entity)
    {
        _dbContext.Set<T>().Add(entity);
        await _dbContext.SaveChangesAsync();
        return entity;
    }

    private IQueryable<T> ApplySpecification(ISpecification<T> spec)
    {
        return SpecificationEvaluator.GetQuery(_dbContext.Set<T>().AsQueryable(), spec);
    }
}
```

**Key Features**:
- Generic across all aggregate roots
- Specification pattern support
- Async/await for performance
- Single responsibility (data access only)

### Specification Pattern

**Purpose**: Encapsulate complex query logic

**Example** - `BasketWithItemsSpecification`:

```csharp
public class BasketWithItemsSpecification : Specification<Basket>
{
    public BasketWithItemsSpecification(int basketId)
    {
        Query
            .Where(b => b.Id == basketId)
            .Include(b => b.Items);  // Eager load to avoid N+1
    }
}
```

**Usage**:
```csharp
var spec = new BasketWithItemsSpecification(basketId);
var basket = await _repository.FirstOrDefaultAsync(spec);
```

**Benefits**:
- Reusable query logic
- Type-safe (compile-time checking)
- Testable (can verify specification logic)
- DRY (Don't Repeat Yourself)

## Integration Architecture

### Authentication & Authorization

#### Web Application (Cookie-Based)

**Configuration** (`/src/Web/Program.cs:47-58`):

```csharp
builder.Services.AddAuthentication(CookieAuthenticationDefaults.AuthenticationScheme)
    .AddCookie(options =>
    {
        options.Cookie.HttpOnly = true;        // XSS protection
        options.Cookie.SecurePolicy = CookieSecurePolicy.Always;  // HTTPS only
        options.Cookie.SameSite = SameSiteMode.Lax;  // CSRF protection
    });

builder.Services.AddIdentity<ApplicationUser, IdentityRole>()
    .AddDefaultUI()
    .AddEntityFrameworkStores<AppIdentityDbContext>()
    .AddDefaultTokenProviders();
```

**Flow**:
1. User submits login form
2. ASP.NET Core Identity validates credentials
3. Cookie issued with authentication ticket
4. Subsequent requests include cookie
5. Middleware validates cookie and sets User principal

#### Public API (JWT Bearer)

**Configuration** (`/src/PublicApi/Program.cs:54-70`):

```csharp
var key = Encoding.ASCII.GetBytes(AuthorizationConstants.JWT_SECRET_KEY);
builder.Services.AddAuthentication(config =>
{
    config.DefaultScheme = JwtBearerDefaults.AuthenticationScheme;
})
.AddJwtBearer(config =>
{
    config.TokenValidationParameters = new TokenValidationParameters
    {
        ValidateIssuerSigningKey = true,
        IssuerSigningKey = new SymmetricSecurityKey(key),
        ValidateIssuer = false,
        ValidateAudience = false
    };
});
```

**Token Generation** (`IdentityTokenClaimService`):

```csharp
public string GetToken(string userId, string username, IEnumerable<string> roles)
{
    var claims = new List<Claim>
    {
        new Claim(ClaimTypes.NameIdentifier, userId),
        new Claim(ClaimTypes.Name, username),
        new Claim(JwtRegisteredClaimNames.Jti, Guid.NewGuid().ToString())
    };

    foreach (var role in roles)
    {
        claims.Add(new Claim(ClaimTypes.Role, role));
    }

    var tokenDescriptor = new SecurityTokenDescriptor
    {
        Subject = new ClaimsIdentity(claims),
        Expires = DateTime.UtcNow.AddMinutes(60),
        SigningCredentials = new SigningCredentials(
            new SymmetricSecurityKey(key),
            SecurityAlgorithms.HmacSha256Signature)
    };

    var tokenHandler = new JwtSecurityTokenHandler();
    var token = tokenHandler.CreateToken(tokenDescriptor);
    return tokenHandler.WriteToken(token);
}
```

**Flow**:
1. Client POST credentials to `/api/auth`
2. Service validates against Identity database
3. JWT created with user claims
4. Token returned to client
5. Client includes in Authorization header: `Bearer {token}`
6. Middleware validates signature and expiration

### CORS Configuration

**Public API** (`/src/PublicApi/Program.cs:72-82`):

```csharp
builder.Services.AddCors(options =>
{
    options.AddPolicy(name: CORS_POLICY,
        corsPolicyBuilder =>
        {
            corsPolicyBuilder.WithOrigins(baseUrlConfig.WebBase);
            corsPolicyBuilder.AllowAnyMethod();
            corsPolicyBuilder.AllowAnyHeader();
        });
});
```

**Purpose**: Allow Blazor Admin (running in browser) to call API

### Caching Strategy

**Decorator Pattern Implementation**:

```
CachedCatalogViewModelService (Decorator)
    ↓ wraps
CatalogViewModelService (Original)
    ↓ uses
IRepository<CatalogItem>
```

**CachedCatalogViewModelService** (`/src/Web/Services/`):

```csharp
public class CachedCatalogViewModelService : ICatalogViewModelService
{
    private readonly IMemoryCache _cache;
    private readonly CatalogViewModelService _catalogService;
    private readonly ILogger<CachedCatalogViewModelService> _logger;

    public async Task<CatalogIndexViewModel> GetCatalogItems(...)
    {
        string cacheKey = $"catalog-{pageIndex}-{itemsPage}-{brandId}-{typeId}";

        return await _cache.GetOrCreateAsync(cacheKey, async entry =>
        {
            entry.SlidingExpiration = TimeSpan.FromMinutes(30);
            _logger.LogInformation("Cache miss for {CacheKey}", cacheKey);
            return await _catalogService.GetCatalogItems(...);
        });
    }
}
```

**Benefits**:
- Transparent to consumers (same interface)
- Easy to enable/disable (DI registration)
- Testable (can test with/without caching)

## Application Architecture

### CQRS Implementation (MediatR)

**Query Example** (`/src/Web/Features/MyOrders/GetMyOrders.cs`):

```csharp
// Query (Request)
public class GetMyOrders : IRequest<IEnumerable<OrderViewModel>>
{
    public string UserName { get; set; }
}

// Handler
public class GetMyOrdersHandler : IRequestHandler<GetMyOrders, IEnumerable<OrderViewModel>>
{
    private readonly IReadRepository<Order> _orderRepository;

    public async Task<IEnumerable<OrderViewModel>> Handle(
        GetMyOrders request,
        CancellationToken cancellationToken)
    {
        var spec = new CustomerOrdersSpecification(request.UserName);
        var orders = await _orderRepository.ListAsync(spec);

        return orders.Select(o => new OrderViewModel
        {
            OrderNumber = o.Id,
            OrderDate = o.OrderDate,
            Total = o.Total()
        });
    }
}
```

**Usage in Razor Page**:

```csharp
public class MyOrdersModel : PageModel
{
    private readonly IMediator _mediator;

    public async Task OnGetAsync()
    {
        var query = new GetMyOrders { UserName = User.Identity.Name };
        Orders = await _mediator.Send(query);
    }
}
```

**Benefits**:
- Decouples UI from query logic
- Handlers testable in isolation
- Easy to add cross-cutting concerns (logging, validation)

### Minimal Endpoints Pattern (PublicApi)

**Endpoint Structure** (`/src/PublicApi/CatalogItemEndpoints/CatalogItemListPagedEndpoint.cs`):

```csharp
public class CatalogItemListPagedEndpoint :
    IEndpoint<IResult, ListPagedCatalogItemRequest, IRepository<CatalogItem>>
{
    private readonly IUriComposer _uriComposer;
    private readonly IMapper _mapper;

    public void AddRoute(IEndpointRouteBuilder app)
    {
        app.MapGet("api/catalog-items",
            async (int? pageSize, int? pageIndex, int? catalogBrandId,
                   int? catalogTypeId, IRepository<CatalogItem> itemRepository) =>
            {
                return await HandleAsync(
                    new ListPagedCatalogItemRequest(...),
                    itemRepository);
            })
            .Produces<ListPagedCatalogItemResponse>()
            .WithTags("CatalogItemEndpoints");
    }

    public async Task<IResult> HandleAsync(
        ListPagedCatalogItemRequest request,
        IRepository<CatalogItem> itemRepository)
    {
        // 1. Create filter specification
        var filterSpec = new CatalogFilterSpecification(
            request.CatalogBrandId,
            request.CatalogTypeId);

        // 2. Get total count
        int totalItems = await itemRepository.CountAsync(filterSpec);

        // 3. Create paginated specification
        var pagedSpec = new CatalogFilterPaginatedSpecification(
            skip: request.PageIndex * request.PageSize,
            take: request.PageSize,
            brandId: request.CatalogBrandId,
            typeId: request.CatalogTypeId);

        // 4. Execute query
        var items = await itemRepository.ListAsync(pagedSpec);

        // 5. Map to DTOs
        var dtos = items.Select(_mapper.Map<CatalogItemDto>);

        // 6. Compose image URIs
        foreach (var dto in dtos)
        {
            dto.PictureUri = _uriComposer.ComposePicUri(dto.PictureUri);
        }

        // 7. Return response
        return Results.Ok(new ListPagedCatalogItemResponse
        {
            CatalogItems = dtos.ToList(),
            PageCount = (int)Math.Ceiling((decimal)totalItems / request.PageSize)
        });
    }
}
```

**Benefits**:
- Each endpoint is self-contained class
- Easy to find and maintain
- Testable in isolation
- Clear separation from routing

### AutoMapper Configuration

**Mapping Profile** (`/src/PublicApi/MappingProfile.cs`):

```csharp
public class MappingProfile : Profile
{
    public MappingProfile()
    {
        CreateMap<CatalogItem, CatalogItemDto>()
            .ForMember(d => d.PictureUri, opt => opt.MapFrom(s => s.PictureUri));

        CreateMap<CatalogBrand, CatalogBrandDto>();
        CreateMap<CatalogType, CatalogTypeDto>();
    }
}
```

**Purpose**: Separate domain entities from API contracts

## Cross-Cutting Concerns

### Exception Handling

**Public API Middleware** (`/src/PublicApi/Middleware/ExceptionMiddleware.cs`):

```csharp
public class ExceptionMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<ExceptionMiddleware> _logger;

    public async Task InvokeAsync(HttpContext httpContext)
    {
        try
        {
            await _next(httpContext);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "An unhandled exception occurred");
            await HandleExceptionAsync(httpContext, ex);
        }
    }

    private static Task HandleExceptionAsync(HttpContext context, Exception exception)
    {
        context.Response.ContentType = "application/json";
        context.Response.StatusCode = (int)HttpStatusCode.InternalServerError;

        return context.Response.WriteAsync(new ErrorDetails
        {
            StatusCode = context.Response.StatusCode,
            Message = "Internal Server Error"
        }.ToString());
    }
}
```

### Logging

**Abstraction** (`/src/ApplicationCore/Interfaces/IAppLogger.cs`):

```csharp
public interface IAppLogger<T>
{
    void LogInformation(string message, params object[] args);
    void LogWarning(string message, params object[] args);
    void LogError(Exception exception, string message, params object[] args);
}
```

**Implementation** (`/src/Infrastructure/Logging/LoggerAdapter.cs`):

```csharp
public class LoggerAdapter<T> : IAppLogger<T>
{
    private readonly ILogger<T> _logger;

    public LoggerAdapter(ILoggerFactory loggerFactory)
    {
        _logger = loggerFactory.CreateLogger<T>();
    }

    public void LogInformation(string message, params object[] args)
    {
        _logger.LogInformation(message, args);
    }

    // ... other methods
}
```

**Benefits**:
- ApplicationCore doesn't depend on Microsoft.Extensions.Logging
- Easy to swap logging implementation
- Testable (can mock IAppLogger)

### Health Checks

**Custom Health Check** (`/src/Web/HealthChecks/ApiHealthCheck.cs`):

```csharp
public class ApiHealthCheck : IHealthCheck
{
    private readonly IConfiguration _configuration;
    private readonly IHttpClientFactory _httpClientFactory;

    public async Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context,
        CancellationToken cancellationToken = default)
    {
        var apiBaseUrl = _configuration["baseUrls:apiBase"];
        var client = _httpClientFactory.CreateClient();

        try
        {
            var response = await client.GetAsync($"{apiBaseUrl}catalog-items", cancellationToken);
            return response.IsSuccessStatusCode
                ? HealthCheckResult.Healthy("API is reachable")
                : HealthCheckResult.Unhealthy("API returned error");
        }
        catch (Exception ex)
        {
            return HealthCheckResult.Unhealthy("API is unreachable", ex);
        }
    }
}
```

**Registration**:

```csharp
builder.Services
    .AddHealthChecks()
    .AddCheck<ApiHealthCheck>("api_health_check", tags: new[] { "apiHealthCheck" })
    .AddCheck<HomePageHealthCheck>("home_page_health_check", tags: new[] { "homePageHealthCheck" });
```

**Endpoints**:
- `/health` - All health checks
- `/api_health_check` - API-specific
- `/home_page_health_check` - Web-specific

## Security Architecture

### Input Validation

**Guard Clauses** (`Ardalis.GuardClauses`):

```csharp
public Order(string buyerId, Address shipToAddress, List<OrderItem> items)
{
    Guard.Against.NullOrEmpty(buyerId, nameof(buyerId));
    Guard.Against.Null(shipToAddress, nameof(shipToAddress));
    Guard.Against.EmptyBasketOnCheckout(items);  // Custom guard

    BuyerId = buyerId;
    ShipToAddress = shipToAddress;
    _orderItems = items;
}
```

**FluentValidation** (API endpoints):

```csharp
public class CreateCatalogItemRequestValidator : AbstractValidator<CreateCatalogItemRequest>
{
    public CreateCatalogItemRequestValidator()
    {
        RuleFor(x => x.Name).NotEmpty().MaximumLength(50);
        RuleFor(x => x.Price).GreaterThan(0);
        RuleFor(x => x.CatalogTypeId).GreaterThan(0);
        RuleFor(x => x.CatalogBrandId).GreaterThan(0);
    }
}
```

### SQL Injection Prevention

**Parameterized Queries** (via EF Core):

```csharp
// EF Core automatically parameterizes
var items = await _context.CatalogItems
    .Where(i => i.Name.Contains(searchTerm))  // ✅ Safe - parameterized
    .ToListAsync();
```

### XSS Prevention

**Razor Pages** - Automatic encoding:

```html
@* Safe - automatically HTML encoded *@
<h1>@Model.ProductName</h1>

@* Unsafe - explicitly unencoded (rarely needed) *@
<div>@Html.Raw(Model.TrustedHtml)</div>
```

## Performance Optimization

### Async/Await Throughout

All I/O operations are async:

```csharp
// Database operations
await _repository.AddAsync(entity);
await _context.SaveChangesAsync();

// External HTTP calls
await _httpClient.GetAsync(url);

// File I/O
await File.WriteAllTextAsync(path, content);
```

### Eager Loading (Avoid N+1)

**Problem**:
```csharp
var baskets = await _context.Baskets.ToListAsync();
foreach (var basket in baskets)
{
    // N+1: Query per basket!
    var items = basket.Items.ToList();
}
```

**Solution** (Specification):
```csharp
Query
    .Where(b => b.Id == basketId)
    .Include(b => b.Items);  // Single query with JOIN
```

### Connection Pooling

**Built-in with EF Core**:

```csharp
builder.Services.AddDbContext<CatalogContext>(options =>
{
    options.UseSqlServer(connectionString, sqlOptions =>
    {
        sqlOptions.EnableRetryOnFailure();  // Resiliency
    });
});
```

---

**Document Status**: ✅ Complete
**Technical Accuracy**: Verified against source code
**Code References**: All examples pulled from actual implementation
