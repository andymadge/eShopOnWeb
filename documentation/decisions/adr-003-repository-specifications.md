# ADR-003: Repository Pattern with Specification Pattern

**Status**: Accepted
**Date**: October 2025 (Inferred from implementation)
**Deciders**: Architecture Team
**Technical Story**: Data access abstraction and query encapsulation

---

## Context

The system needs a way to access data that:
- Abstracts Entity Framework Core implementation details
- Enables unit testing without a database
- Encapsulates complex query logic
- Remains type-safe and compile-time checked
- Supports DDD principles (aggregate roots)
- Avoids the "leaky abstraction" problem of many repository implementations

### Problems with Direct DbContext Usage

**Coupling to EF Core**:
```csharp
// ❌ Domain service directly uses EF Core
public class OrderService
{
    private readonly CatalogContext _context;

    public async Task<Order> GetOrderAsync(int id)
    {
        return await _context.Orders
            .Include(o => o.OrderItems)
            .FirstOrDefaultAsync(o => o.Id == id);
    }
}
```

**Issues**:
- Service knows about EF Core (Include, FirstOrDefaultAsync)
- Can't test without database
- Query logic mixed with business logic
- Violates Clean Architecture (domain depends on infrastructure)

### Problems with Traditional Generic Repository

```csharp
// ❌ Basic repository with fixed methods
public interface IRepository<T>
{
    Task<T> GetByIdAsync(int id);
    Task<List<T>> GetAllAsync();
    Task<List<T>> FindAsync(Expression<Func<T, bool>> predicate);
}
```

**Issues**:
- `FindAsync` exposes EF Core Expression trees (leaky abstraction)
- Can't encapsulate eager loading (Include)
- Every new query needs new repository method
- Repository interfaces become bloated
- Hard to reuse query logic

## Decision

We will implement **Generic Repository Pattern with Specification Pattern**:

### 1. Generic Repository Interfaces

**Location**: `/src/ApplicationCore/Interfaces/`

```csharp
public interface IReadRepository<T> where T : BaseEntity, IAggregateRoot
{
    Task<T?> GetByIdAsync(int id, CancellationToken cancellationToken = default);
    Task<List<T>> ListAsync(CancellationToken cancellationToken = default);
    Task<List<T>> ListAsync(ISpecification<T> spec, CancellationToken cancellationToken = default);
    Task<int> CountAsync(ISpecification<T> spec, CancellationToken cancellationToken = default);
    Task<T?> FirstOrDefaultAsync(ISpecification<T> spec, CancellationToken cancellationToken = default);
}

public interface IRepository<T> : IReadRepository<T> where T : BaseEntity, IAggregateRoot
{
    Task<T> AddAsync(T entity, CancellationToken cancellationToken = default);
    Task UpdateAsync(T entity, CancellationToken cancellationToken = default);
    Task DeleteAsync(T entity, CancellationToken cancellationToken = default);
    Task DeleteRangeAsync(IEnumerable<T> entities, CancellationToken cancellationToken = default);
}
```

**Key Design Choices**:
- Separate read and write interfaces (CQRS-friendly)
- Generic across all aggregate roots
- Takes `ISpecification<T>` for complex queries
- No EF Core types exposed

### 2. Specification Pattern

**Base Specification** (Ardalis.Specification library):

```csharp
public abstract class Specification<T> : ISpecification<T>
{
    protected ISpecificationBuilder<T> Query { get; }

    // Builder pattern for queries
    protected Specification()
    {
        Query = new SpecificationBuilder<T>(this);
    }
}
```

**Concrete Specification Example**:

**Location**: `/src/ApplicationCore/Specifications/BasketWithItemsSpecification.cs`

```csharp
public class BasketWithItemsSpecification : Specification<Basket>
{
    public BasketWithItemsSpecification(int basketId)
    {
        Query
            .Where(b => b.Id == basketId)
            .Include(b => b.Items);  // Eager loading
    }
}

// Overload for querying by buyer ID
public class BasketWithItemsSpecification : Specification<Basket>
{
    public BasketWithItemsSpecification(string buyerId)
    {
        Query
            .Where(b => b.BuyerId == buyerId)
            .Include(b => b.Items);
    }
}
```

**Benefits**:
- Query logic encapsulated in specification
- Reusable across application
- Type-safe (compile-time checked)
- Testable (can verify specification logic)
- No EF Core in domain layer

### 3. Repository Implementation

**Location**: `/src/Infrastructure/Data/EfRepository.cs`

```csharp
public class EfRepository<T> : IRepository<T> where T : BaseEntity, IAggregateRoot
{
    private readonly CatalogContext _dbContext;

    public EfRepository(CatalogContext dbContext)
    {
        _dbContext = dbContext;
    }

    public async Task<T?> GetByIdAsync(int id, CancellationToken cancellationToken = default)
    {
        return await _dbContext.Set<T>().FindAsync(new object[] { id }, cancellationToken);
    }

    public async Task<List<T>> ListAsync(ISpecification<T> spec, CancellationToken cancellationToken = default)
    {
        return await ApplySpecification(spec).ToListAsync(cancellationToken);
    }

    public async Task<int> CountAsync(ISpecification<T> spec, CancellationToken cancellationToken = default)
    {
        return await ApplySpecification(spec).CountAsync(cancellationToken);
    }

    public async Task<T> AddAsync(T entity, CancellationToken cancellationToken = default)
    {
        await _dbContext.Set<T>().AddAsync(entity, cancellationToken);
        await _dbContext.SaveChangesAsync(cancellationToken);
        return entity;
    }

    private IQueryable<T> ApplySpecification(ISpecification<T> spec)
    {
        return SpecificationEvaluator.GetQuery(_dbContext.Set<T>().AsQueryable(), spec);
    }
}
```

**Key Points**:
- Single implementation for all aggregate roots
- Applies specifications to generate EF queries
- All EF Core details contained in Infrastructure

### 4. Usage in Domain Services

**Location**: `/src/ApplicationCore/Services/OrderService.cs`

```csharp
public class OrderService : IOrderService
{
    private readonly IRepository<Order> _orderRepository;
    private readonly IRepository<Basket> _basketRepository;
    private readonly IRepository<CatalogItem> _itemRepository;

    public async Task CreateOrderAsync(int basketId, Address shippingAddress)
    {
        // Use specification to load basket with items
        var basketSpec = new BasketWithItemsSpecification(basketId);
        var basket = await _basketRepository.FirstOrDefaultAsync(basketSpec);

        Guard.Against.Null(basket, nameof(basket));
        Guard.Against.EmptyBasketOnCheckout(basket.Items);

        // Load catalog items for prices
        var catalogItemsSpec = new CatalogItemsSpecification(
            basket.Items.Select(item => item.CatalogItemId).ToArray());
        var catalogItems = await _itemRepository.ListAsync(catalogItemsSpec);

        // Create order
        var items = basket.Items.Select(basketItem =>
        {
            var catalogItem = catalogItems.First(c => c.Id == basketItem.CatalogItemId);
            var itemOrdered = new CatalogItemOrdered(
                catalogItem.Id,
                catalogItem.Name,
                _uriComposer.ComposePicUri(catalogItem.PictureUri));
            return new OrderItem(itemOrdered, basketItem.UnitPrice, basketItem.Quantity);
        }).ToList();

        var order = new Order(basket.BuyerId, shippingAddress, items);
        await _orderRepository.AddAsync(order);
    }
}
```

**Benefits**:
- No EF Core in domain service
- Clear intent (BasketWithItemsSpecification)
- Testable (can mock IRepository)
- Reusable specifications

## Consequences

### Positive

1. **Clean Architecture Maintained** ✅
   - ApplicationCore has no EF Core dependency
   - Domain services depend only on interfaces
   - Infrastructure details isolated

2. **High Testability** ✅
   - Can mock IRepository for unit tests
   - Can test specifications independently
   - Fast tests without database

3. **Query Reusability** ✅
   - BasketWithItemsSpecification used by multiple consumers
   - CatalogFilterSpecification reused for listing and counting
   - DRY principle (Don't Repeat Yourself)

4. **Type Safety** ✅
   - Compile-time checking of queries
   - No magic strings
   - Refactoring-friendly

5. **Explicit Query Logic** ✅
   - Query intent clear from specification name
   - Easy to find where queries are defined
   - Self-documenting

6. **Performance Optimization** ✅
   - Eager loading explicitly defined (Include)
   - Can add AsNoTracking for read-only queries
   - Pagination support (Skip/Take)

### Negative

1. **Additional Classes** ⚠️
   - One specification class per query pattern
   - More files to navigate
   - **Impact**: Project has 7 specifications (manageable)
   - **Mitigation**: Clear naming, organized folders

2. **Learning Curve** ⚠️
   - Developers must understand specification pattern
   - Not as straightforward as direct queries
   - **Impact**: Requires training for new developers
   - **Mitigation**: Documentation, examples, pair programming

3. **Library Dependency** ⚠️
   - Depends on Ardalis.Specification
   - Tied to specific implementation
   - **Impact**: Minimal - stable, well-maintained library
   - **Mitigation**: Interface is standard, could swap implementation if needed

4. **Overhead for Simple Queries** ⚠️
   - GetById is straightforward, doesn't need specification
   - Some queries feel over-engineered
   - **Impact**: Minor - methods like GetByIdAsync don't require specification
   - **Mitigation**: Provide non-specification methods for simple cases

## Validation

### Success Criteria (Met)

✅ **Domain services have no EF Core dependency**
   - Verified: ApplicationCore project references only Ardalis.Specification

✅ **Queries are reusable**
   - Verified: BasketWithItemsSpecification used in 3+ places

✅ **Unit tests don't require database**
   - Verified: UnitTests project mocks IRepository

✅ **Specifications are type-safe**
   - Verified: Compilation fails if entity properties change

### Example Specifications

| Specification | Purpose | Consumers |
|---------------|---------|-----------|
| `BasketWithItemsSpecification` | Load basket with items | OrderService, BasketService, Web Pages |
| `CatalogFilterSpecification` | Filter by brand/type | CatalogViewModelService, API |
| `CatalogFilterPaginatedSpecification` | Paginated catalog | Web Pages, API |
| `CustomerOrdersSpecification` | User's order history | MyOrders page, API |
| `CatalogItemsSpecification` | Load items by IDs | OrderService |

### Test Example

**Unit Test** (`/tests/UnitTests/ApplicationCore/Services/OrderServiceTests.cs`):

```csharp
[Fact]
public async Task CreateOrder_GivenValidBasket_CreatesOrder()
{
    // Arrange
    var mockBasketRepo = Substitute.For<IRepository<Basket>>();
    var mockOrderRepo = Substitute.For<IRepository<Order>>();
    var mockItemRepo = Substitute.For<IRepository<CatalogItem>>();

    var basket = new Basket("buyer1");
    basket.AddItem(1, 10.00m, 2);

    mockBasketRepo.FirstOrDefaultAsync(Arg.Any<BasketWithItemsSpecification>())
        .Returns(basket);

    var orderService = new OrderService(
        mockBasketRepo,
        mockItemRepo,
        mockOrderRepo,
        mockUriComposer);

    // Act
    await orderService.CreateOrderAsync(1, new Address("123 Main St", "City", "State", "US", "12345"));

    // Assert
    await mockOrderRepo.Received(1).AddAsync(Arg.Any<Order>());
}
```

**Benefits**:
- No database required
- Fast execution
- Easy to set up test data

## Alternatives Considered

### Alternative 1: Direct DbContext Injection

**Approach**: Inject CatalogContext into domain services

**Pros**:
- Simpler, fewer abstractions
- Full EF Core power available
- No additional library

**Cons**:
- Violates Clean Architecture
- Domain coupled to EF Core
- Can't unit test without database
- Hard to swap ORM

**Decision**: Rejected - Violates core architectural principles

### Alternative 2: Specific Repository per Entity

**Approach**: IBasketRepository, IOrderRepository, ICatalogItemRepository

**Pros**:
- Explicit contracts per entity
- Can add entity-specific methods

**Cons**:
- Lots of repetitive interfaces
- Repetitive implementations
- Violates DRY
- More code to maintain

**Decision**: Rejected - Generic repository with specifications more elegant

### Alternative 3: CQRS with MediatR for All Data Access

**Approach**: Every query is a MediatR request

**Pros**:
- Maximum decoupling
- Easy to add cross-cutting concerns
- Very testable

**Cons**:
- Overhead for simple CRUD
- Many more classes
- Learning curve steep

**Decision**: Partially adopted - MediatR used for complex queries in Web project, repository for simple operations

### Alternative 4: Query Objects (Without Specification Library)

**Approach**: Custom query object pattern

**Pros**:
- No external library
- Full control

**Cons**:
- Must implement specification evaluator
- Reinventing the wheel
- Less features than established library

**Decision**: Rejected - Ardalis.Specification is mature, well-tested, open-source

## Implementation Guidelines

### Creating a New Specification

1. **Identify the query need**
2. **Create specification class** in `/src/ApplicationCore/Specifications/`
3. **Use builder pattern** to define query
4. **Include eager loading** if needed
5. **Add unit tests** for specification logic

**Template**:
```csharp
public class MyEntitySpecification : Specification<MyEntity>
{
    public MyEntitySpecification(int someFilter)
    {
        Query
            .Where(e => e.SomeProperty == someFilter)
            .Include(e => e.RelatedEntities)
            .OrderBy(e => e.Name);
    }
}
```

### Using a Specification

```csharp
var spec = new MyEntitySpecification(filterValue);
var entities = await _repository.ListAsync(spec);
var count = await _repository.CountAsync(spec);
var firstMatch = await _repository.FirstOrDefaultAsync(spec);
```

## Related Decisions

- [ADR-001: Clean Architecture](./adr-001-clean-architecture.md) - Repository enables domain independence
- [ADR-004: Domain-Driven Design](./adr-004-domain-driven-design.md) - Repository works with aggregate roots
- [ADR-005: CQRS with MediatR](./adr-005-cqrs-mediatr.md) - Repository complements MediatR queries

## References

- **Repository Pattern**: Martin Fowler, Patterns of Enterprise Application Architecture
- **Specification Pattern**: Eric Evans, Domain-Driven Design
- **Ardalis.Specification**: https://github.com/ardalis/Specification
- **Microsoft Docs**: https://docs.microsoft.com/en-us/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/infrastructure-persistence-layer-design

---

**Reviewed**: October 24, 2025
**Status**: Validated against implementation
**Outcome**: Successful - Pattern enables clean, testable data access
**Evidence**: 7 specifications, 0 EF Core references in ApplicationCore
