# Component Catalog

This document provides a comprehensive catalog of all major components in the eShopOnWeb application, organized by layer and responsibility.

---

## Table of Contents

- [ApplicationCore Components](#applicationcore-components)
  - [Domain Entities](#domain-entities)
  - [Services](#services)
  - [Specifications](#specifications)
  - [Interfaces](#interfaces)
- [Infrastructure Components](#infrastructure-components)
  - [Data Access](#data-access)
  - [Identity & Security](#identity--security)
  - [External Services](#external-services)
- [Web Application Components](#web-application-components)
  - [View Model Services](#view-model-services)
  - [Controllers](#controllers)
  - [Pages](#pages)
- [PublicApi Components](#publicapi-components)
  - [Catalog Item Endpoints](#catalog-item-endpoints)
  - [Catalog Brand Endpoints](#catalog-brand-endpoints)
  - [Catalog Type Endpoints](#catalog-type-endpoints)
  - [Authentication Endpoints](#authentication-endpoints)
- [BlazorAdmin Components](#blazoradmin-components)
  - [Admin Services](#admin-services)

---

## ApplicationCore Components

### Domain Entities

#### CatalogItem

**Purpose**: Represents a product in the catalog with pricing, description, and classification.

**Location**: `/src/ApplicationCore/Entities/CatalogItem.cs`

**Responsibilities**:
- Store product information (name, description, price, picture URI)
- Maintain relationships to brand and type classifications
- Enforce business rules for product updates
- Validate price and description requirements

**Dependencies**:
- Internal: `BaseEntity`, `CatalogBrand`, `CatalogType`
- External: Ardalis.GuardClauses

**Public APIs**:
- `UpdateDetails(CatalogItemDetails details)` - Updates name, description, and price with validation
- `UpdateBrand(int catalogBrandId)` - Changes the product's brand
- `UpdateType(int catalogTypeId)` - Changes the product's type
- `UpdatePictureUri(string pictureName)` - Updates the product image path

**Data Managed**:
- Product properties: Name, Description, Price, PictureUri
- Classification: CatalogTypeId, CatalogBrandId
- Navigation properties to CatalogBrand and CatalogType

**Notes**:
- Uses private setters and methods to enforce encapsulation
- Implements IAggregateRoot marker interface for DDD pattern
- Guards against null/empty values and negative prices

---

#### CatalogBrand

**Purpose**: Represents a product brand/manufacturer in the catalog.

**Location**: `/src/ApplicationCore/Entities/CatalogBrand.cs`

**Responsibilities**:
- Store brand name information
- Serve as a classification dimension for catalog items

**Dependencies**:
- Internal: `BaseEntity`, `IAggregateRoot`

**Public APIs**:
- Constructor: `CatalogBrand(string brand)` - Creates new brand

**Data Managed**:
- Brand name (single string property)

**Notes**:
- Simple value object with minimal behavior
- Immutable after creation (private setter)
- Used for filtering and organizing products

---

#### CatalogType

**Purpose**: Represents a product category/type in the catalog.

**Location**: `/src/ApplicationCore/Entities/CatalogType.cs`

**Responsibilities**:
- Store product type/category information
- Serve as a classification dimension for catalog items

**Dependencies**:
- Internal: `BaseEntity`, `IAggregateRoot`

**Public APIs**:
- Constructor: `CatalogType(string type)` - Creates new type

**Data Managed**:
- Type name (single string property)

**Notes**:
- Simple value object with minimal behavior
- Immutable after creation (private setter)
- Used for filtering and organizing products

---

#### Basket

**Purpose**: Shopping cart aggregate root managing items a customer intends to purchase.

**Location**: `/src/ApplicationCore/Entities/BasketAggregate/Basket.cs`

**Responsibilities**:
- Maintain collection of basket items
- Calculate total item count
- Add items with quantity management
- Remove empty items
- Transfer basket ownership between users

**Dependencies**:
- Internal: `BaseEntity`, `BasketItem`, `IAggregateRoot`
- External: Ardalis.GuardClauses

**Public APIs**:
- `AddItem(int catalogItemId, decimal unitPrice, int quantity = 1)` - Adds or increments item
- `RemoveEmptyItems()` - Cleans up items with zero quantity
- `SetNewBuyerId(string buyerId)` - Changes basket ownership

**Data Managed**:
- BuyerId (username or anonymous ID)
- Collection of BasketItem entities
- Computed TotalItems count

**Notes**:
- Implements aggregate pattern with private collection
- Automatically consolidates duplicate items
- Protects Items collection with IReadOnlyCollection

---

#### BasketItem

**Purpose**: Line item within a shopping basket representing a catalog item with quantity and price.

**Location**: `/src/ApplicationCore/Entities/BasketAggregate/BasketItem.cs`

**Responsibilities**:
- Store catalog item reference, quantity, and unit price
- Validate and update quantities
- Maintain relationship to parent Basket

**Dependencies**:
- Internal: `BaseEntity`
- External: Ardalis.GuardClauses

**Public APIs**:
- `AddQuantity(int quantity)` - Increments quantity
- `SetQuantity(int quantity)` - Sets absolute quantity

**Data Managed**:
- CatalogItemId, UnitPrice, Quantity, BasketId

**Notes**:
- Guards against negative or excessive quantities
- Price stored as snapshot (doesn't change if catalog item price changes)

---

#### Order

**Purpose**: Represents a completed purchase order with shipping details and line items.

**Location**: `/src/ApplicationCore/Entities/OrderAggregate/Order.cs`

**Responsibilities**:
- Store order details (buyer, date, shipping address)
- Maintain collection of order items
- Calculate order total
- Enforce aggregate boundaries

**Dependencies**:
- Internal: `BaseEntity`, `OrderItem`, `Address`, `IAggregateRoot`
- External: Ardalis.GuardClauses

**Public APIs**:
- `Total()` - Calculates sum of all line items

**Data Managed**:
- BuyerId, OrderDate, ShipToAddress
- Collection of OrderItem entities

**Notes**:
- Pure DDD aggregate with encapsulated collection
- Immutable after creation (no update methods)
- Uses private parameterless constructor for EF Core

---

#### OrderItem

**Purpose**: Line item within an order representing a purchased catalog item.

**Location**: `/src/ApplicationCore/Entities/OrderAggregate/OrderItem.cs`

**Responsibilities**:
- Store ordered item details (snapshot of catalog item)
- Maintain unit price and quantity at time of order
- Reference original catalog item

**Dependencies**:
- Internal: `BaseEntity`, `CatalogItemOrdered`

**Public APIs**:
- Constructor only (immutable after creation)

**Data Managed**:
- ItemOrdered (CatalogItemOrdered value object)
- UnitPrice, Units

**Notes**:
- Immutable to preserve order history
- Uses value object for catalog item snapshot

---

#### Address

**Purpose**: Value object representing a shipping address.

**Location**: `/src/ApplicationCore/Entities/OrderAggregate/Address.cs`

**Responsibilities**:
- Store complete mailing address
- Provide value semantics for addresses

**Dependencies**:
- None (pure value object)

**Public APIs**:
- Constructor only (immutable)

**Data Managed**:
- Street, City, State, Country, ZipCode

**Notes**:
- Value object (no identity)
- Owned by Order entity in database

---

### Services

#### BasketService

**Purpose**: Application service managing basket operations and business logic.

**Location**: `/src/ApplicationCore/Services/BasketService.cs`

**Responsibilities**:
- Add items to baskets
- Update quantities
- Delete baskets
- Transfer anonymous baskets to authenticated users

**Dependencies**:
- Internal: `IRepository<Basket>`, `IAppLogger<BasketService>`, specifications
- External: Ardalis.GuardClauses, Ardalis.Result

**Public APIs**:
- `AddItemToBasket(string username, int catalogItemId, decimal price, int quantity = 1)` - Adds item
- `DeleteBasketAsync(int basketId)` - Removes basket
- `SetQuantities(int basketId, Dictionary<string, int> quantities)` - Bulk quantity update
- `TransferBasketAsync(string anonymousId, string userName)` - Migrates basket on login

**Data Managed**:
- Orchestrates basket aggregate operations

**Notes**:
- Creates basket on-demand if not exists
- Implements transfer logic for anonymous-to-authenticated flow

---

#### OrderService

**Purpose**: Application service managing order creation from baskets.

**Location**: `/src/ApplicationCore/Services/OrderService.cs`

**Responsibilities**:
- Create orders from basket contents
- Validate basket state
- Retrieve current catalog item data
- Construct order aggregates

**Dependencies**:
- Internal: `IRepository<Order>`, `IRepository<Basket>`, `IRepository<CatalogItem>`, `IUriComposer`
- External: Ardalis.GuardClauses

**Public APIs**:
- `CreateOrderAsync(int basketId, Address shippingAddress)` - Creates order from basket

**Data Managed**:
- Orchestrates order creation workflow

**Notes**:
- Validates non-empty basket
- Takes snapshot of current catalog item prices
- Uses specifications for efficient data retrieval

---

#### UriComposer

**Purpose**: Service for composing URI paths for catalog item images.

**Location**: `/src/ApplicationCore/Services/UriComposer.cs`

**Responsibilities**:
- Generate absolute URIs for product images
- Handle base URI configuration

**Dependencies**:
- Internal: `IUriComposer` interface

**Public APIs**:
- `ComposePicUri(string uriTemplate)` - Converts relative path to absolute URI

**Data Managed**:
- Base URI configuration

**Notes**:
- Centralizes URI generation logic
- Supports CDN or local image storage

---

### Specifications

#### CatalogFilterSpecification

**Purpose**: Query specification for filtering catalog items by brand and/or type.

**Location**: `/src/ApplicationCore/Specifications/CatalogFilterSpecification.cs`

**Responsibilities**:
- Define filtering criteria for catalog queries
- Include related entities (Brand, Type)

**Dependencies**:
- External: Ardalis.Specification

**Public APIs**:
- Constructor with optional brandId and typeId parameters

**Notes**:
- Used for counting total filtered items
- Companion to CatalogFilterPaginatedSpecification

---

#### CatalogFilterPaginatedSpecification

**Purpose**: Extends filtering with pagination support.

**Location**: `/src/ApplicationCore/Specifications/CatalogFilterPaginatedSpecification.cs`

**Responsibilities**:
- Add skip/take pagination to filtered catalog queries
- Maintain same filtering logic as CatalogFilterSpecification

**Dependencies**:
- External: Ardalis.Specification

**Public APIs**:
- Constructor with pagination parameters (skip, take, brandId, typeId)

**Notes**:
- Implements efficient pagination
- Always use with CatalogFilterSpecification for accurate page counts

---

#### BasketWithItemsSpecification

**Purpose**: Query specification for retrieving basket with all items loaded.

**Location**: `/src/ApplicationCore/Specifications/BasketWithItemsSpecification.cs`

**Responsibilities**:
- Eager load basket items collection
- Filter by buyer ID or basket ID

**Dependencies**:
- External: Ardalis.Specification

**Public APIs**:
- Constructor accepting either username or basketId

**Notes**:
- Prevents N+1 query problems
- Essential for basket operations

---

#### CatalogItemsSpecification

**Purpose**: Query specification for retrieving multiple catalog items by ID.

**Location**: `/src/ApplicationCore/Specifications/CatalogItemsSpecification.cs`

**Responsibilities**:
- Filter catalog items by array of IDs
- Optimize bulk retrieval

**Dependencies**:
- External: Ardalis.Specification

**Public APIs**:
- Constructor accepting int[] of catalog item IDs

**Notes**:
- Used when processing baskets or orders
- Efficient bulk query

---

### Interfaces

#### IRepository<T>

**Purpose**: Generic repository interface for aggregate root persistence.

**Location**: `/src/ApplicationCore/Interfaces/IRepository.cs`

**Responsibilities**:
- Define contract for CRUD operations
- Support specification pattern queries

**Public APIs**:
- `GetByIdAsync(int id)` - Retrieve by primary key
- `ListAsync()` - List all entities
- `ListAsync(ISpecification<T> spec)` - Query by specification
- `AddAsync(T entity)` - Create entity
- `UpdateAsync(T entity)` - Update entity
- `DeleteAsync(T entity)` - Remove entity
- `CountAsync(ISpecification<T> spec)` - Count matching entities
- `FirstOrDefaultAsync(ISpecification<T> spec)` - Get single matching entity

**Notes**:
- Implemented by EfRepository in Infrastructure layer
- Only works with IAggregateRoot entities

---

#### IBasketService

**Purpose**: Service interface for basket operations.

**Location**: `/src/ApplicationCore/Interfaces/IBasketService.cs`

**Public APIs**:
- `AddItemToBasket(string username, int catalogItemId, decimal price, int quantity)` - Add item
- `SetQuantities(int basketId, Dictionary<string, int> quantities)` - Update quantities
- `DeleteBasketAsync(int basketId)` - Remove basket
- `TransferBasketAsync(string anonymousId, string userName)` - Transfer basket

---

#### IOrderService

**Purpose**: Service interface for order operations.

**Location**: `/src/ApplicationCore/Interfaces/IOrderService.cs`

**Public APIs**:
- `CreateOrderAsync(int basketId, Address shippingAddress)` - Create order

---

## Infrastructure Components

### Data Access

#### CatalogContext

**Purpose**: Entity Framework DbContext for all application data.

**Location**: `/src/Infrastructure/Data/CatalogContext.cs`

**Responsibilities**:
- Define database schema via DbSet properties
- Configure entity relationships and mappings
- Provide data access interface

**Dependencies**:
- External: Entity Framework Core

**Public APIs**:
- DbSet properties for each entity type

**Data Managed**:
- All application entities: Baskets, CatalogItems, CatalogBrands, CatalogTypes, Orders, OrderItems, BasketItems

**Notes**:
- Uses Fluent API configurations from separate configuration classes
- Applies all IEntityTypeConfiguration classes from assembly

---

#### EfRepository<T>

**Purpose**: Generic Entity Framework implementation of IRepository.

**Location**: `/src/Infrastructure/Data/EfRepository.cs`

**Responsibilities**:
- Implement repository pattern over EF Core
- Execute specification queries
- Provide transactional data access

**Dependencies**:
- Internal: `CatalogContext`, `IRepository<T>`
- External: Entity Framework Core, Ardalis.Specification.EntityFrameworkCore

**Public APIs**:
- Implements all IRepository<T> methods

**Notes**:
- Uses Ardalis.Specification evaluator for query building
- All IAggregateRoot entities can use this repository

---

#### Entity Configurations

**Purpose**: Fluent API configuration classes for entity mappings.

**Location**: `/src/Infrastructure/Data/Config/*.cs`

**Responsibilities**:
- Define table names and schema
- Configure primary keys and identity strategies
- Set up relationships and foreign keys
- Define constraints (required, max length, data types)
- Configure value objects and owned entities

**Components**:
- `CatalogItemConfiguration` - Maps CatalogItem to "Catalog" table, configures HiLo identity
- `CatalogBrandConfiguration` - Maps CatalogBrand entity
- `CatalogTypeConfiguration` - Maps CatalogType entity
- `BasketConfiguration` - Maps Basket with items collection
- `BasketItemConfiguration` - Maps BasketItem entity
- `OrderConfiguration` - Maps Order with owned Address
- `OrderItemConfiguration` - Maps OrderItem with owned CatalogItemOrdered

**Notes**:
- Separates mapping configuration from entity classes
- Uses HiLo pattern for high-performance ID generation
- Configures owned types for value objects

---

#### CatalogContextSeed

**Purpose**: Database seeding class for initial/demo data.

**Location**: `/src/Infrastructure/Data/CatalogContextSeed.cs`

**Responsibilities**:
- Populate initial catalog brands and types
- Create sample catalog items with images
- Ensure database is ready for demo/development

**Notes**:
- Idempotent (safe to run multiple times)
- Used during application startup

---

### Identity & Security

#### IdentityTokenClaimService

**Purpose**: Service for generating JWT tokens for authenticated users.

**Location**: `/src/Infrastructure/Identity/` (inferred)

**Responsibilities**:
- Generate JWT tokens with user claims
- Include role information in tokens
- Set token expiration

**Dependencies**:
- Internal: `ITokenClaimsService`
- External: System.IdentityModel.Tokens.Jwt, ASP.NET Core Identity

**Public APIs**:
- `GetTokenAsync(string username)` - Generates JWT for user

**Notes**:
- Used by authentication endpoints
- Tokens include role claims for authorization

---

### External Services

#### EmailSender

**Purpose**: Service for sending email notifications.

**Location**: `/src/Infrastructure/Services/EmailSender.cs`

**Responsibilities**:
- Send transactional emails
- Implement IEmailSender interface

**Dependencies**:
- Internal: `IEmailSender`

**Notes**:
- May be stubbed in development
- Production implementation would use SendGrid, SMTP, or similar

---

## Web Application Components

### View Model Services

#### CatalogViewModelService

**Purpose**: Transforms domain entities into view models for catalog display.

**Location**: `/src/Web/Services/CatalogViewModelService.cs`

**Responsibilities**:
- Retrieve paginated catalog items
- Build filter dropdowns (brands, types)
- Compose image URIs
- Create pagination metadata

**Dependencies**:
- Internal: `IRepository<CatalogItem>`, `IRepository<CatalogBrand>`, `IRepository<CatalogType>`, `IUriComposer`
- External: Microsoft.Extensions.Logging

**Public APIs**:
- `GetCatalogItems(int pageIndex, int itemsPage, int? brandId, int? typeId)` - Returns CatalogIndexViewModel
- `GetBrands()` - Returns brand dropdown options
- `GetTypes()` - Returns type dropdown options

**Data Managed**:
- View models and display data

**Notes**:
- UI-specific service (not business logic)
- Belongs in Web project, not ApplicationCore
- Uses specifications for efficient queries

---

#### BasketViewModelService

**Purpose**: Transforms basket entities into view models for UI display.

**Location**: `/src/Web/Services/BasketViewModelService.cs`

**Responsibilities**:
- Retrieve or create basket for user
- Map basket items with catalog details
- Compose image URIs for basket items
- Count total basket items

**Dependencies**:
- Internal: `IRepository<Basket>`, `IRepository<CatalogItem>`, `IUriComposer`, `IBasketQueryService`

**Public APIs**:
- `GetOrCreateBasketForUser(string userName)` - Returns BasketViewModel
- `Map(Basket basket)` - Converts entity to view model
- `CountTotalBasketItems(string username)` - Returns item count

**Notes**:
- Creates basket on-demand if doesn't exist
- Enriches basket items with current catalog data

---

#### CachedCatalogViewModelService

**Purpose**: Decorator adding caching to CatalogViewModelService.

**Location**: `/src/Web/Services/CachedCatalogViewModelService.cs`

**Responsibilities**:
- Cache catalog query results
- Improve performance for repeated queries
- Delegate to wrapped service on cache miss

**Dependencies**:
- Internal: `ICatalogViewModelService`, `IMemoryCache`

**Notes**:
- Implements decorator pattern
- Configurable cache duration
- Cache invalidation on catalog changes

---

### Controllers

#### CatalogController

**Purpose**: MVC controller serving catalog browse and detail pages.

**Location**: Inferred from typical MVC structure

**Responsibilities**:
- Serve catalog index with filtering and pagination
- Display individual product details
- Handle catalog browsing UI

---

#### BasketController

**Purpose**: MVC controller managing basket operations via web UI.

**Location**: Inferred from typical MVC structure

**Responsibilities**:
- Display basket contents
- Add items to basket
- Update quantities
- Remove items
- Checkout initiation

---

#### OrderController

**Purpose**: MVC controller handling order placement and history.

**Location**: `/src/Web/Controllers/OrderController.cs`

**Responsibilities**:
- Display order history
- Show order details
- Create new orders

---

### Pages

#### Razor Pages

**Location**: `/src/Web/Pages/` (typical structure)

**Components**:
- Basket page - Shopping cart UI
- Checkout page - Order creation workflow
- Order confirmation page - Post-purchase display

**Notes**:
- Uses Razor Pages pattern
- Server-side rendering with MVVM pattern

---

## PublicApi Components

All Public API endpoints use the Minimal API pattern with the MinimalApi.Endpoint library.

### Catalog Item Endpoints

#### CatalogItemListPagedEndpoint

**Purpose**: REST endpoint for paginated catalog item listing with filtering.

**Location**: `/src/PublicApi/CatalogItemEndpoints/CatalogItemListPagedEndpoint.cs`

**Responsibilities**:
- Accept pagination and filter parameters
- Query catalog with specifications
- Return paged results with metadata
- Compose image URIs

**Dependencies**:
- Internal: `IRepository<CatalogItem>`, `IUriComposer`, `IMapper`
- External: AutoMapper

**Route**: `GET /api/catalog-items`

**Query Parameters**:
- `pageSize` (optional, int)
- `pageIndex` (optional, int)
- `catalogBrandId` (optional, int)
- `catalogTypeId` (optional, int)

**Response**: `ListPagedCatalogItemResponse` with CatalogItems array and PageCount

**Notes**:
- Includes intentional 1-second delay (likely for demo purposes)
- Returns 200 OK with empty list if no items match

---

#### CatalogItemGetByIdEndpoint

**Purpose**: REST endpoint for retrieving single catalog item by ID.

**Location**: `/src/PublicApi/CatalogItemEndpoints/CatalogItemGetByIdEndpoint.cs`

**Responsibilities**:
- Retrieve catalog item by ID
- Return 404 if not found
- Compose image URI

**Dependencies**:
- Internal: `IRepository<CatalogItem>`, `IUriComposer`

**Route**: `GET /api/catalog-items/{catalogItemId}`

**Path Parameters**:
- `catalogItemId` (required, int)

**Responses**:
- 200 OK with `GetByIdCatalogItemResponse`
- 404 Not Found

---

#### CreateCatalogItemEndpoint

**Purpose**: REST endpoint for creating new catalog items.

**Location**: `/src/PublicApi/CatalogItemEndpoints/CreateCatalogItemEndpoint.cs`

**Responsibilities**:
- Validate uniqueness of item name
- Create new catalog item
- Set default picture URI
- Return created item

**Dependencies**:
- Internal: `IRepository<CatalogItem>`, `IUriComposer`

**Route**: `POST /api/catalog-items`

**Authorization**: Requires Administrator role with JWT Bearer token

**Request Body**: `CreateCatalogItemRequest` (name, description, price, brand ID, type ID, picture URI)

**Responses**:
- 201 Created with location header and `CreateCatalogItemResponse`
- 400 Bad Request if duplicate name

**Notes**:
- Enforces unique item names
- Sets placeholder image for security reasons
- Requires authentication and admin role

---

#### UpdateCatalogItemEndpoint

**Purpose**: REST endpoint for updating existing catalog items.

**Location**: `/src/PublicApi/CatalogItemEndpoints/UpdateCatalogItemEndpoint.cs`

**Responsibilities**:
- Retrieve existing item
- Update details via domain methods
- Return updated item

**Dependencies**:
- Internal: `IRepository<CatalogItem>`, `IUriComposer`

**Route**: `PUT /api/catalog-items`

**Authorization**: Requires Administrator role with JWT Bearer token

**Request Body**: `UpdateCatalogItemRequest` (id, name, description, price, brand ID, type ID)

**Responses**:
- 200 OK with `UpdateCatalogItemResponse`
- 404 Not Found

**Notes**:
- Uses domain entity update methods
- Validates via guard clauses in entity
- Picture URI not updated via this endpoint

---

#### DeleteCatalogItemEndpoint

**Purpose**: REST endpoint for deleting catalog items.

**Location**: `/src/PublicApi/CatalogItemEndpoints/DeleteCatalogItemEndpoint.cs`

**Responsibilities**:
- Retrieve and delete catalog item
- Return success status

**Route**: `DELETE /api/catalog-items/{catalogItemId}`

**Authorization**: Requires Administrator role with JWT Bearer token

**Path Parameters**:
- `catalogItemId` (required, int)

**Responses**:
- 200 OK with `DeleteCatalogItemResponse`
- 404 Not Found

---

### Catalog Brand Endpoints

#### CatalogBrandListEndpoint

**Purpose**: REST endpoint for listing all catalog brands.

**Location**: `/src/PublicApi/CatalogBrandEndpoints/CatalogBrandListEndpoint.cs`

**Responsibilities**:
- Retrieve all brands
- Map to DTOs
- Return brand list

**Dependencies**:
- Internal: `IRepository<CatalogBrand>`, `IMapper`

**Route**: `GET /api/catalog-brands`

**Response**: `ListCatalogBrandsResponse` with array of CatalogBrandDto

**Notes**:
- No pagination (assumes small dataset)
- No authentication required (read-only)

---

### Catalog Type Endpoints

#### CatalogTypeListEndpoint

**Purpose**: REST endpoint for listing all catalog types.

**Location**: `/src/PublicApi/CatalogTypeEndpoints/CatalogTypeListEndpoint.cs`

**Responsibilities**:
- Retrieve all types
- Map to DTOs
- Return type list

**Dependencies**:
- Internal: `IRepository<CatalogType>`, `IMapper`

**Route**: `GET /api/catalog-types`

**Response**: `ListCatalogTypesResponse` with array of CatalogTypeDto

**Notes**:
- No pagination (assumes small dataset)
- No authentication required (read-only)

---

### Authentication Endpoints

#### AuthenticateEndpoint

**Purpose**: REST endpoint for user authentication and JWT token generation.

**Location**: `/src/PublicApi/AuthEndpoints/AuthenticateEndpoint.cs`

**Responsibilities**:
- Validate username and password
- Generate JWT token on success
- Return authentication result with status flags

**Dependencies**:
- Internal: `SignInManager<ApplicationUser>`, `ITokenClaimsService`
- External: ASP.NET Core Identity

**Route**: `POST /api/authenticate`

**Request Body**: `AuthenticateRequest` (username, password)

**Response**: `AuthenticateResponse` with:
- `Result` (bool) - Whether authentication succeeded
- `Token` (string) - JWT token if successful
- `IsLockedOut` (bool)
- `IsNotAllowed` (bool)
- `RequiresTwoFactor` (bool)
- `Username` (string)

**Notes**:
- Uses ASP.NET Core Identity
- Lockout enabled on failures
- Returns descriptive status even on failure

---

## BlazorAdmin Components

### Admin Services

#### CatalogItemService

**Purpose**: HTTP client service for catalog item CRUD via Public API.

**Location**: `/src/BlazorAdmin/Services/CatalogItemService.cs`

**Responsibilities**:
- Consume Public API endpoints
- Provide strongly-typed methods for catalog operations
- Handle HTTP communication

**Dependencies**:
- Internal: HttpService
- External: System.Net.Http

**Public APIs**:
- CRUD methods for catalog items
- List, Get, Create, Update, Delete operations

---

#### CachedCatalogItemServiceDecorator

**Purpose**: Decorator adding client-side caching to CatalogItemService.

**Location**: `/src/BlazorAdmin/Services/CachedCatalogItemServiceDecorator.cs`

**Responsibilities**:
- Cache catalog item queries
- Invalidate cache on mutations
- Improve Blazor app performance

**Dependencies**:
- Internal: CatalogItemService wrapper

**Notes**:
- Decorator pattern for cross-cutting concern
- In-memory cache with expiration

---

#### CatalogLookupDataService

**Purpose**: Service for retrieving catalog lookup data (brands, types).

**Location**: `/src/BlazorAdmin/Services/CatalogLookupDataService.cs`

**Responsibilities**:
- Load brands and types for dropdowns
- Provide reference data to admin UI

**Dependencies**:
- Internal: HttpService

---

#### CachedCatalogLookupDataServiceDecorator

**Purpose**: Caching decorator for lookup data service.

**Location**: `/src/BlazorAdmin/Services/CachedCatalogLookupDataServiceDecorator.cs`

**Responsibilities**:
- Cache brands and types (changes infrequently)
- Reduce API calls for reference data

---

#### ToastService

**Purpose**: UI notification service for Blazor admin.

**Location**: `/src/BlazorAdmin/Services/ToastService.cs`

**Responsibilities**:
- Display success/error notifications
- Provide user feedback for operations

---

#### HttpService

**Purpose**: Base HTTP client wrapper for authenticated API calls.

**Location**: `/src/BlazorAdmin/Services/HttpService.cs`

**Responsibilities**:
- Handle JWT token injection
- Centralize HTTP error handling
- Provide common HTTP utilities

**Dependencies**:
- External: Blazored.LocalStorage for token storage

**Notes**:
- Used by all other Blazor services
- Manages authentication headers

---

## Cross-Cutting Components

### Logging

**Purpose**: Structured logging throughout application.

**Interface**: `IAppLogger<T>` (ApplicationCore)

**Implementation**: Uses Microsoft.Extensions.Logging

**Notes**:
- Abstraction over logging framework
- Allows switching logging providers
- Structured logging with context

### Mapping

**Purpose**: Object-to-object mapping for DTOs and view models.

**Technology**: AutoMapper

**Usage**:
- Map entities to DTOs in API endpoints
- Map entities to view models in services

**Configuration**: AutoMapper profiles in each project

---

## Component Dependencies Summary

### Dependency Flow (Typical Request)

1. **Web UI Request** → Controller/Page
2. **Controller** → View Model Service
3. **View Model Service** → Repository (via Interface)
4. **Repository** → DbContext
5. **DbContext** → Database

### API Request Flow

1. **HTTP Request** → Endpoint
2. **Endpoint** → Repository/Service (via Interface)
3. **Repository** → DbContext
4. **DbContext** → Database
5. **Response** → Map to DTO → Return

### Key Architectural Patterns

- **Repository Pattern**: Abstracts data access
- **Specification Pattern**: Encapsulates query logic
- **Service Layer**: Encapsulates business logic
- **Dependency Injection**: Loose coupling via interfaces
- **Decorator Pattern**: Cross-cutting concerns (caching)
- **Domain-Driven Design**: Aggregates, entities, value objects
- **CQRS Light**: Separate read models (view models) from write models (entities)

---

## Notes on Component Organization

### ApplicationCore Philosophy

- **No Infrastructure Dependencies**: Core domain is infrastructure-agnostic
- **Interface-First**: All external dependencies accessed via interfaces
- **Rich Domain Models**: Business logic in entities, not services
- **Specification Pattern**: Query logic stays testable

### Infrastructure Responsibilities

- **Data Persistence**: EF Core, DbContext, repositories
- **External Services**: Email, identity, file storage
- **Technical Concerns**: Caching, logging implementations

### Web Project Responsibilities

- **UI Logic**: View models, view model services
- **Presentation**: Controllers, pages, views
- **Client-Side Assets**: JavaScript, CSS, images

### PublicApi Project

- **REST Endpoints**: HTTP API for external consumption
- **DTO Contracts**: API-specific data transfer objects
- **Minimal API Pattern**: Lightweight endpoint definitions

### BlazorAdmin Project

- **Admin UI**: Blazor WebAssembly SPA
- **Admin Services**: HTTP clients for API consumption
- **Client-Side Logic**: Blazor components and state

---

## Component Testing Strategy

### Unit Testing

- **Domain Entities**: Test business rules and validation
- **Services**: Test orchestration logic with mocked repositories
- **Specifications**: Test query building logic

### Integration Testing

- **Repositories**: Test against in-memory or test database
- **API Endpoints**: Test HTTP layer with TestServer
- **Services**: Test with real database

### Functional Testing

- **End-to-End Flows**: Complete user scenarios
- **UI Testing**: Selenium or Playwright for Blazor

---

## Version Information

This component catalog reflects the codebase as of .NET 8.0, using the project structure and patterns present in the eShopOnWeb reference application.

---

## Related Documentation

- [System Overview](../architecture/system-overview.md)
- [Technical Architecture](../architecture/technical-architecture.md)
- [API Documentation](./api-documentation.md)
- [Data Dictionary](./data-dictionary.md)
- [Technology Stack](./technology-stack.md)
