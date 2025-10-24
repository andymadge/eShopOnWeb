# Data Dictionary

This document provides comprehensive documentation of all data entities, database tables, and their relationships in the eShopOnWeb application.

---

## Table of Contents

- [Database Overview](#database-overview)
- [Catalog Entities](#catalog-entities)
  - [CatalogItem (Catalog Table)](#catalogitem-catalog-table)
  - [CatalogBrand](#catalogbrand)
  - [CatalogType](#catalogtype)
- [Basket Aggregate](#basket-aggregate)
  - [Basket](#basket)
  - [BasketItem](#basketitem)
- [Order Aggregate](#order-aggregate)
  - [Order](#order)
  - [OrderItem](#orderitem)
  - [Address (Owned Entity)](#address-owned-entity)
- [Identity Entities](#identity-entities)
- [Entity Relationships](#entity-relationships)
- [Database Indexes](#database-indexes)
- [Database Sequences](#database-sequences)

---

## Database Overview

### Database Technology

- **Provider**: SQL Server (default), with in-memory option for testing
- **ORM**: Entity Framework Core 8.0
- **Migration Strategy**: Code-first with EF Core Migrations
- **Connection String Config**: `appsettings.json` (ConnectionStrings:CatalogConnection)

### Schema Information

- **Default Schema**: `dbo`
- **Naming Convention**: PascalCase for tables and columns
- **Primary Key Pattern**: Integer `Id` column with HiLo generation for some entities
- **Audit Fields**: Not implemented (no CreatedAt, UpdatedAt fields)

### Database Context

- **Context Class**: `CatalogContext`
- **Location**: `/src/Infrastructure/Data/CatalogContext.cs`
- **Configuration**: Fluent API in separate configuration classes (`/src/Infrastructure/Data/Config/`)

---

## Catalog Entities

### CatalogItem (Catalog Table)

**Purpose**: Represents a product in the catalog with pricing, description, classification, and image.

**Table Name**: `Catalog`

**Entity Class**: `/src/ApplicationCore/Entities/CatalogItem.cs`

**Configuration**: `/src/Infrastructure/Data/Config/CatalogItemConfiguration.cs`

#### Attributes

| Column | Type | Nullable | Default | Description | Constraints |
|--------|------|----------|---------|-------------|-------------|
| Id | int | No | Auto | Unique catalog item identifier | Primary Key, HiLo sequence |
| Name | nvarchar(50) | No | - | Product name | Required, Max 50 chars |
| Description | nvarchar(max) | No | - | Product description | Required |
| Price | decimal(18,2) | No | - | Product price | Required, Must be > 0 |
| PictureUri | nvarchar(max) | Yes | null | Relative path to product image | Optional |
| CatalogTypeId | int | No | - | Foreign key to CatalogType | Required |
| CatalogBrandId | int | No | - | Foreign key to CatalogBrand | Required |

#### Relationships

**Many-to-One with CatalogBrand**:
- Foreign Key: `CatalogBrandId`
- Navigation Property: `CatalogBrand`
- Cascade: Restrict (cannot delete brand with items)

**Many-to-One with CatalogType**:
- Foreign Key: `CatalogTypeId`
- Navigation Property: `CatalogType`
- Cascade: Restrict (cannot delete type with items)

#### Indexes

- **Primary Key Index**: Clustered index on `Id`
- **Foreign Key Indexes**: Non-clustered on `CatalogBrandId` and `CatalogTypeId`
- **Potential Custom Indexes**: Consider adding index on `Name` for search operations

#### Business Rules

- Name must be unique (enforced in application layer, not database)
- Price must be positive (enforced in domain entity)
- Cannot be null or empty: Name, Description
- Picture URI uses format: `images\products\{filename}?{ticks}`

#### Sample Data

```sql
Id  | Name                      | Description                           | Price  | PictureUri              | TypeId | BrandId
----|---------------------------|---------------------------------------|--------|-------------------------|--------|--------
1   | .NET Bot Black Hoodie     | Premium quality hoodie with design    | 19.50  | images\products\1.png   | 2      | 1
2   | .NET Black & White Mug    | Classic ceramic mug                   | 8.50   | images\products\2.png   | 1      | 1
3   | .NET Foundation T-Shirt   | Lightweight cotton t-shirt            | 12.00  | images\products\3.png   | 2      | 1
```

#### Notes

- Uses HiLo identity generation for high-performance ID allocation
- Table name differs from entity name (Catalog vs CatalogItem)
- Price stored as decimal for precision in monetary calculations
- PictureUri is relative path (composed to absolute URL at runtime)

---

### CatalogBrand

**Purpose**: Lookup table for product brands/manufacturers.

**Table Name**: `CatalogBrand`

**Entity Class**: `/src/ApplicationCore/Entities/CatalogBrand.cs`

**Configuration**: `/src/Infrastructure/Data/Config/CatalogBrandConfiguration.cs`

#### Attributes

| Column | Type | Nullable | Default | Description | Constraints |
|--------|------|----------|---------|-------------|-------------|
| Id | int | No | Identity | Unique brand identifier | Primary Key, Identity(1,1) |
| Brand | nvarchar(100) | No | - | Brand name | Required |

#### Relationships

**One-to-Many with CatalogItem**:
- Referenced by: `CatalogItem.CatalogBrandId`
- Cascade: Restrict

#### Indexes

- **Primary Key Index**: Clustered index on `Id`

#### Business Rules

- Brand name should be unique (not enforced by database constraint)
- Immutable after creation (no update methods in entity)

#### Sample Data

```sql
Id  | Brand
----|------------------
1   | .NET
2   | Azure
3   | Visual Studio
4   | Other
```

#### Notes

- Simple lookup/reference table
- Small dataset (typically < 100 brands)
- No soft delete (must not have associated catalog items to delete)

---

### CatalogType

**Purpose**: Lookup table for product types/categories.

**Table Name**: `CatalogType`

**Entity Class**: `/src/ApplicationCore/Entities/CatalogType.cs`

**Configuration**: `/src/Infrastructure/Data/Config/CatalogTypeConfiguration.cs`

#### Attributes

| Column | Type | Nullable | Default | Description | Constraints |
|--------|------|----------|---------|-------------|-------------|
| Id | int | No | Identity | Unique type identifier | Primary Key, Identity(1,1) |
| Type | nvarchar(100) | No | - | Type/category name | Required |

#### Relationships

**One-to-Many with CatalogItem**:
- Referenced by: `CatalogItem.CatalogTypeId`
- Cascade: Restrict

#### Indexes

- **Primary Key Index**: Clustered index on `Id`

#### Business Rules

- Type name should be unique (not enforced by database constraint)
- Immutable after creation (no update methods in entity)

#### Sample Data

```sql
Id  | Type
----|--------------------
1   | Mug
2   | T-Shirt
3   | Sheet
4   | USB Memory Stick
```

#### Notes

- Simple lookup/reference table
- Small dataset (typically < 50 types)
- No soft delete (must not have associated catalog items to delete)

---

## Basket Aggregate

### Basket

**Purpose**: Shopping cart containing items a customer intends to purchase.

**Table Name**: `Baskets`

**Entity Class**: `/src/ApplicationCore/Entities/BasketAggregate/Basket.cs`

**Configuration**: `/src/Infrastructure/Data/Config/BasketConfiguration.cs`

#### Attributes

| Column | Type | Nullable | Default | Description | Constraints |
|--------|------|----------|---------|-------------|-------------|
| Id | int | No | Identity | Unique basket identifier | Primary Key, Identity(1,1) |
| BuyerId | nvarchar(256) | No | - | Username or anonymous ID | Required, Max 256 chars |

#### Relationships

**One-to-Many with BasketItem**:
- Child Collection: `Items` (List<BasketItem>)
- Cascade: Cascade delete (deleting basket deletes all items)

#### Indexes

- **Primary Key Index**: Clustered index on `Id`
- **Recommended Index**: Non-clustered index on `BuyerId` for fast lookups

#### Business Rules

- BuyerId can be username (authenticated) or GUID (anonymous)
- Empty baskets (no items) are valid
- Basket ownership can be transferred (anonymous to authenticated user)
- Items collection is private (add via AddItem method only)

#### Computed Properties

- `TotalItems`: Sum of quantities across all basket items (not stored in DB)

#### Sample Data

```sql
Id  | BuyerId
----|----------------------------------
1   | demouser@microsoft.com
2   | 550e8400-e29b-41d4-a716-446655440000
3   | admin@microsoft.com
```

#### Notes

- Baskets persist until explicitly deleted or transferred
- Anonymous baskets created on first add-to-cart action
- Anonymous basket merged into authenticated basket on login

---

### BasketItem

**Purpose**: Line item within a basket representing a catalog item with quantity and captured price.

**Table Name**: `BasketItems`

**Entity Class**: `/src/ApplicationCore/Entities/BasketAggregate/BasketItem.cs`

**Configuration**: `/src/Infrastructure/Data/Config/BasketItemConfiguration.cs`

#### Attributes

| Column | Type | Nullable | Default | Description | Constraints |
|--------|------|----------|---------|-------------|-------------|
| Id | int | No | Identity | Unique basket item identifier | Primary Key, Identity(1,1) |
| UnitPrice | decimal(18,2) | No | - | Price per unit at time of add | Required |
| Quantity | int | No | - | Number of units | Required, Must be >= 0 |
| CatalogItemId | int | No | - | Reference to catalog item | Required |
| BasketId | int | No | - | Foreign key to Basket | Required |

#### Relationships

**Many-to-One with Basket**:
- Foreign Key: `BasketId`
- Navigation Property: (none, owned by Basket aggregate)
- Cascade: Cascade delete

**Reference to CatalogItem**:
- Foreign Key: `CatalogItemId`
- Note: Not a true navigation property (reference only)
- Catalog item can be deleted without affecting basket items

#### Indexes

- **Primary Key Index**: Clustered index on `Id`
- **Foreign Key Index**: Non-clustered on `BasketId`

#### Business Rules

- Quantity must be >= 0 (enforced by guard clauses)
- Quantity of 0 indicates item marked for removal (cleaned up by RemoveEmptyItems)
- UnitPrice is snapshot at time of adding (doesn't update if catalog price changes)
- One basket item per catalog item per basket (quantities consolidated)

#### Sample Data

```sql
Id  | UnitPrice | Quantity | CatalogItemId | BasketId
----|-----------|----------|---------------|----------
1   | 19.50     | 2        | 1             | 1
2   | 8.50      | 1        | 2             | 1
3   | 12.00     | 3        | 3             | 2
```

#### Notes

- Part of Basket aggregate (no direct repository access)
- Price stored as snapshot (not recalculated from catalog)
- Empty items (quantity = 0) removed before order creation

---

## Order Aggregate

### Order

**Purpose**: Completed purchase order with buyer information, shipping address, and order items.

**Table Name**: `Orders`

**Entity Class**: `/src/ApplicationCore/Entities/OrderAggregate/Order.cs`

**Configuration**: `/src/Infrastructure/Data/Config/OrderConfiguration.cs`

#### Attributes

| Column | Type | Nullable | Default | Description | Constraints |
|--------|------|----------|---------|-------------|-------------|
| Id | int | No | Identity | Unique order identifier | Primary Key, Identity(1,1) |
| BuyerId | nvarchar(256) | No | - | Username of buyer | Required, Max 256 chars |
| OrderDate | datetimeoffset | No | GETDATE() | When order was created | Required |
| ShipToAddress_Street | nvarchar(180) | No | - | Shipping street address | Required, Max 180 chars |
| ShipToAddress_City | nvarchar(100) | No | - | Shipping city | Required, Max 100 chars |
| ShipToAddress_State | nvarchar(60) | Yes | - | Shipping state/province | Max 60 chars |
| ShipToAddress_Country | nvarchar(90) | No | - | Shipping country | Required, Max 90 chars |
| ShipToAddress_ZipCode | nvarchar(18) | No | - | Shipping postal code | Required, Max 18 chars |

#### Relationships

**One-to-Many with OrderItem**:
- Child Collection: `OrderItems` (List<OrderItem>)
- Cascade: Cascade delete

**Owns Address**:
- Owned Entity: `ShipToAddress` (Address value object)
- Storage: Columns prefixed with `ShipToAddress_`

#### Indexes

- **Primary Key Index**: Clustered index on `Id`
- **Recommended Indexes**:
  - Non-clustered on `BuyerId` for user order history
  - Non-clustered on `OrderDate` for recent orders queries

#### Business Rules

- Immutable after creation (no update methods)
- Must have at least one order item
- BuyerId must be authenticated user (no anonymous orders)
- Address is required (owned entity)
- Order date set automatically on creation

#### Computed Properties

- `Total()`: Sum of (UnitPrice × Units) for all order items (not stored)

#### Sample Data

```sql
Id | BuyerId                | OrderDate            | ShipToAddress_Street | ShipToAddress_City | ShipToAddress_State | ShipToAddress_Country | ShipToAddress_ZipCode
---|------------------------|----------------------|----------------------|--------------------|---------------------|-----------------------|----------------------
1  | demouser@microsoft.com | 2024-10-20T14:30:00Z | 123 Main St          | Seattle            | WA                  | USA                   | 98101
2  | admin@microsoft.com    | 2024-10-22T09:15:00Z | 456 Oak Ave          | Redmond            | WA                  | USA                   | 98052
```

#### Notes

- Orders are immutable (audit trail)
- No order status field (could be added for order tracking)
- Address stored as owned entity (not separate table)
- Uses DateTimeOffset for timezone-aware dates

---

### OrderItem

**Purpose**: Line item within an order representing a purchased catalog item with snapshot data.

**Table Name**: `OrderItems`

**Entity Class**: `/src/ApplicationCore/Entities/OrderAggregate/OrderItem.cs`

**Configuration**: `/src/Infrastructure/Data/Config/OrderItemConfiguration.cs`

#### Attributes

| Column | Type | Nullable | Default | Description | Constraints |
|--------|------|----------|---------|-------------|-------------|
| Id | int | No | Identity | Unique order item identifier | Primary Key, Identity(1,1) |
| UnitPrice | decimal(18,2) | No | - | Price per unit at time of order | Required |
| Units | int | No | - | Quantity ordered | Required, Must be > 0 |
| OrderId | int | No | - | Foreign key to Order | Required |
| ItemOrdered_CatalogItemId | int | No | - | Original catalog item ID | Required |
| ItemOrdered_ProductName | nvarchar(50) | No | - | Product name at time of order | Required, Max 50 chars |
| ItemOrdered_PictureUri | nvarchar(max) | Yes | - | Product image at time of order | Optional |

#### Relationships

**Many-to-One with Order**:
- Foreign Key: `OrderId`
- Navigation Property: (none, owned by Order aggregate)
- Cascade: Cascade delete

**Owns CatalogItemOrdered**:
- Owned Entity: `ItemOrdered` (CatalogItemOrdered value object)
- Storage: Columns prefixed with `ItemOrdered_`

#### Indexes

- **Primary Key Index**: Clustered index on `Id`
- **Foreign Key Index**: Non-clustered on `OrderId`

#### Business Rules

- Immutable after creation
- Units must be > 0
- Stores snapshot of catalog item (name, picture) at time of order
- Original CatalogItemId preserved for reference but may no longer exist

#### Computed Properties

- Line Total: `UnitPrice × Units` (not stored, calculated on demand)

#### Sample Data

```sql
Id | UnitPrice | Units | OrderId | ItemOrdered_CatalogItemId | ItemOrdered_ProductName     | ItemOrdered_PictureUri
---|-----------|-------|---------|---------------------------|-----------------------------|-----------------------
1  | 19.50     | 2     | 1       | 1                         | .NET Bot Black Hoodie       | images\products\1.png
2  | 8.50      | 1     | 1       | 2                         | .NET Black & White Mug      | images\products\2.png
3  | 12.00     | 3     | 2       | 3                         | .NET Foundation T-Shirt     | images\products\3.png
```

#### Notes

- Part of Order aggregate
- Snapshot pattern preserves order history even if catalog item changes
- No foreign key to CatalogItem table (allows catalog item deletion)
- ItemOrdered stored as owned entity (not separate table)

---

### Address (Owned Entity)

**Purpose**: Value object representing a shipping or billing address.

**Table Name**: N/A (stored in Order table)

**Entity Class**: `/src/ApplicationCore/Entities/OrderAggregate/Address.cs`

**Configuration**: Owned by Order in `/src/Infrastructure/Data/Config/OrderConfiguration.cs`

#### Attributes

| Property | Type | Nullable | Description | Constraints |
|----------|------|----------|-------------|-------------|
| Street | string | No | Street address | Required, Max 180 chars in DB |
| City | string | No | City name | Required, Max 100 chars in DB |
| State | string | No | State/Province | Max 60 chars in DB |
| Country | string | No | Country name | Required, Max 90 chars in DB |
| ZipCode | string | No | Postal/ZIP code | Required, Max 18 chars in DB |

#### Storage

Stored as columns in the `Orders` table:
- `ShipToAddress_Street`
- `ShipToAddress_City`
- `ShipToAddress_State`
- `ShipToAddress_Country`
- `ShipToAddress_ZipCode`

#### Business Rules

- Immutable (no setters)
- All fields except State are required
- Value object (no identity, compared by value)

#### Notes

- Classic DDD value object pattern
- Owned by Order entity (not separate table)
- Could be shared with Buyer entity in future (billing address)

---

## Identity Entities

The application uses **ASP.NET Core Identity** for user management. Identity tables are in a separate context but share the same database.

### AspNetUsers

**Purpose**: User accounts for authentication.

**Key Columns**:
- `Id` (nvarchar(450), PK)
- `UserName` (nvarchar(256))
- `NormalizedUserName` (nvarchar(256), unique)
- `Email` (nvarchar(256))
- `NormalizedEmail` (nvarchar(256))
- `PasswordHash` (nvarchar(max))
- `SecurityStamp` (nvarchar(max))
- Additional ASP.NET Identity columns

### AspNetRoles

**Purpose**: Role definitions (e.g., Administrator).

**Key Columns**:
- `Id` (nvarchar(450), PK)
- `Name` (nvarchar(256))
- `NormalizedName` (nvarchar(256), unique)

### AspNetUserRoles

**Purpose**: Many-to-many relationship between users and roles.

**Key Columns**:
- `UserId` (nvarchar(450), FK to AspNetUsers, PK)
- `RoleId` (nvarchar(450), FK to AspNetRoles, PK)

#### Default Roles

- **Administrators**: Full access to catalog management APIs

#### Sample Users

**Administrator**:
- Username: `admin@microsoft.com`
- Password: `Pass@word1` (hashed)
- Role: Administrators

**Demo User**:
- Username: `demouser@microsoft.com`
- Password: `Pass@word1` (hashed)
- Role: (none)

---

## Entity Relationships

### Entity Relationship Diagram

```
CatalogBrand (1) ----< (N) CatalogItem (N) >---- (1) CatalogType
                              |
                              | (referenced by, no FK)
                              |
Basket (1) ----< (N) BasketItem >---- (ref) CatalogItem
  |
  | (BuyerId reference)
  |
AspNetUsers

Order (1) ----< (N) OrderItem
  |                     |
  |                     | (owns)
  |                     |
  | (owns)              CatalogItemOrdered (value object)
  |
Address (value object)
```

### Relationship Details

#### CatalogItem Relationships

- **CatalogItem N:1 CatalogBrand**: Many items per brand
  - FK: `CatalogBrandId`
  - Delete: Restrict

- **CatalogItem N:1 CatalogType**: Many items per type
  - FK: `CatalogTypeId`
  - Delete: Restrict

#### Basket Relationships

- **Basket 1:N BasketItem**: One basket has many items
  - FK: `BasketId` in BasketItem
  - Delete: Cascade

- **BasketItem references CatalogItem**: Reference only, no FK constraint
  - Field: `CatalogItemId`
  - Allows basket items to remain if catalog item deleted

#### Order Relationships

- **Order 1:N OrderItem**: One order has many items
  - FK: `OrderId` in OrderItem
  - Delete: Cascade

- **Order owns Address**: Owned entity
  - Stored as columns in Order table

- **OrderItem owns CatalogItemOrdered**: Owned entity
  - Stored as columns in OrderItem table

---

## Database Indexes

### Existing Indexes

#### Primary Keys (Clustered)

All tables have clustered index on `Id` column:
- `PK_CatalogBrand`
- `PK_CatalogType`
- `PK_Catalog`
- `PK_Baskets`
- `PK_BasketItems`
- `PK_Orders`
- `PK_OrderItems`

#### Foreign Key Indexes (Non-Clustered)

Automatically created for foreign keys:
- `IX_Catalog_CatalogBrandId`
- `IX_Catalog_CatalogTypeId`
- `IX_BasketItems_BasketId`
- `IX_OrderItems_OrderId`

### Recommended Additional Indexes

#### For Performance Optimization

```sql
-- Basket lookup by buyer
CREATE NONCLUSTERED INDEX IX_Baskets_BuyerId
ON Baskets (BuyerId);

-- Order history by buyer
CREATE NONCLUSTERED INDEX IX_Orders_BuyerId
ON Orders (BuyerId);

-- Recent orders
CREATE NONCLUSTERED INDEX IX_Orders_OrderDate
ON Orders (OrderDate DESC);

-- Catalog item search by name
CREATE NONCLUSTERED INDEX IX_Catalog_Name
ON Catalog (Name);
```

---

## Database Sequences

### HiLo Sequences

The `CatalogItem` table uses **HiLo** (High-Low) pattern for ID generation:

**Sequence Name**: `catalog_hilo`

**Purpose**: Generate unique IDs in batches for better performance

**Configuration**:
- Increment: 10 (default)
- Min Value: 1
- Max Value: 9223372036854775807

**How it works**:
1. Application requests a "high" value from database
2. Application generates "low" values locally (0-9)
3. Next request gets next "high" value

**Benefits**:
- Reduces database round trips
- Improves insert performance
- Still maintains unique IDs

**Example**:
```
High=1 → App generates IDs: 10, 11, 12, 13, 14, 15, 16, 17, 18, 19
High=2 → App generates IDs: 20, 21, 22, 23, 24, 25, 26, 27, 28, 29
```

---

## Data Access Patterns

### Repository Pattern

All entities accessed through `IRepository<T>` interface:

```csharp
var item = await _repository.GetByIdAsync(id);
var items = await _repository.ListAsync(specification);
await _repository.AddAsync(newItem);
await _repository.UpdateAsync(item);
await _repository.DeleteAsync(item);
```

### Specification Pattern

Complex queries encapsulated in specification classes:

- `CatalogFilterSpecification` - Filter by brand/type
- `CatalogFilterPaginatedSpecification` - Filter + pagination
- `BasketWithItemsSpecification` - Basket with eager-loaded items
- `CatalogItemsSpecification` - Multiple items by ID array

### Entity Framework Features Used

- **Fluent API**: Entity configuration
- **Shadow Properties**: EF-managed properties
- **Owned Entities**: Value objects (Address, CatalogItemOrdered)
- **Navigation Properties**: Relationship traversal
- **Lazy Loading**: Disabled (explicit Include required)
- **Change Tracking**: Enabled for updates

---

## Migration History

Migrations located in: `/src/Infrastructure/Data/Migrations/`

### Initial Migration

Creates all tables and relationships:
- Catalog tables (CatalogItem, CatalogBrand, CatalogType)
- Basket tables (Basket, BasketItem)
- Order tables (Order, OrderItem)
- Identity tables (AspNetUsers, AspNetRoles, etc.)

### Applying Migrations

**From Package Manager Console**:
```powershell
Update-Database
```

**From Command Line**:
```bash
dotnet ef database update --project src/Infrastructure --startup-project src/Web
```

---

## Data Seeding

### Seed Data Location

`/src/Infrastructure/Data/CatalogContextSeed.cs`

### Seeded Data

**CatalogBrands**:
- .NET
- Azure
- Visual Studio
- Other

**CatalogTypes**:
- Mug
- T-Shirt
- Sheet
- USB Memory Stick

**CatalogItems**:
- 12 sample products with images
- Various combinations of brands and types
- Prices ranging from $8.50 to $19.50

**Identity**:
- Administrator user: `admin@microsoft.com`
- Demo user: `demouser@microsoft.com`
- Administrator role

### Running Seed

Seed runs automatically on application startup if database is empty.

---

## Database Constraints Summary

### Primary Key Constraints

All entities have integer primary keys with appropriate generation strategy (Identity or HiLo).

### Foreign Key Constraints

- `FK_Catalog_CatalogBrand` - Restrict delete
- `FK_Catalog_CatalogType` - Restrict delete
- `FK_BasketItems_Basket` - Cascade delete
- `FK_OrderItems_Order` - Cascade delete

### Check Constraints

None currently defined (validation in application layer).

**Recommended additions**:
```sql
ALTER TABLE Catalog
ADD CONSTRAINT CK_Catalog_Price CHECK (Price > 0);

ALTER TABLE BasketItems
ADD CONSTRAINT CK_BasketItems_Quantity CHECK (Quantity >= 0);

ALTER TABLE OrderItems
ADD CONSTRAINT CK_OrderItems_Units CHECK (Units > 0);
```

### Unique Constraints

None on business data (only on Identity tables).

**Recommended additions**:
```sql
ALTER TABLE Catalog
ADD CONSTRAINT UQ_Catalog_Name UNIQUE (Name);
```

---

## Backup and Recovery

### Backup Strategy

**Development**:
- No backup required (seeded data)

**Production**:
- Full backup: Daily
- Differential backup: Hourly
- Transaction log backup: Every 15 minutes
- Point-in-time recovery enabled

### Recovery Time Objective (RTO)

Target: < 1 hour for production

### Recovery Point Objective (RPO)

Target: < 15 minutes for production

---

## Performance Considerations

### Query Performance

- Use specifications for complex queries
- Enable query result caching where appropriate
- Monitor query execution plans
- Add indexes for frequently queried columns

### Write Performance

- HiLo pattern reduces ID generation overhead
- Batch operations where possible
- Use `AddRangeAsync` for bulk inserts

### Database Size Estimates

**Small Shop**:
- Catalog Items: < 1000 rows
- Orders: < 10,000 rows
- Total Size: < 100 MB

**Medium Shop**:
- Catalog Items: < 50,000 rows
- Orders: < 100,000 rows
- Total Size: < 1 GB

**Large Shop**:
- Catalog Items: < 500,000 rows
- Orders: < 1,000,000 rows
- Total Size: < 10 GB

---

## Related Documentation

- [Component Catalog](./component-catalog.md)
- [API Documentation](./api-documentation.md)
- [Technology Stack](./technology-stack.md)
- [Technical Architecture](../architecture/technical-architecture.md)
