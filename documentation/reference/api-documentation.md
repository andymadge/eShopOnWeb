# API Documentation

This document provides comprehensive documentation for the eShopOnWeb Public API endpoints, including request/response formats, authentication requirements, and usage examples.

---

## Table of Contents

- [API Overview](#api-overview)
- [Authentication](#authentication)
- [Catalog Item Endpoints](#catalog-item-endpoints)
- [Catalog Brand Endpoints](#catalog-brand-endpoints)
- [Catalog Type Endpoints](#catalog-type-endpoints)
- [Authentication Endpoints](#authentication-endpoints)
- [Common Response Patterns](#common-response-patterns)
- [Error Handling](#error-handling)

---

## API Overview

### Base URL

**Development**: `http://localhost:5099`
**Production**: `https://your-domain.com`

### API Prefix

All API endpoints are prefixed with `/api/`

### Content Type

All requests and responses use `application/json` unless otherwise specified.

### API Framework

The Public API is built using:
- **ASP.NET Core 8.0** Minimal APIs
- **MinimalApi.Endpoint** library for organization
- **Swashbuckle** for OpenAPI/Swagger documentation

### Swagger UI

Interactive API documentation available at: `/swagger`

---

## Authentication

### Overview

The API uses **JWT Bearer Token** authentication for protected endpoints.

### Authentication Flow

1. Obtain a JWT token by calling the `/api/authenticate` endpoint with valid credentials
2. Include the token in the `Authorization` header for subsequent requests
3. Token format: `Authorization: Bearer <your-jwt-token>`

### Authorization

Some endpoints require specific roles:
- **Administrator**: Full access to catalog management (create, update, delete)
- **Authenticated Users**: Can access their own data (baskets, orders)
- **Anonymous**: Read-only access to catalog

### Token Expiration

JWT tokens have a configurable expiration time (typically 60-120 minutes). Clients should handle 401 Unauthorized responses by re-authenticating.

---

## Catalog Item Endpoints

### List Catalog Items (Paginated)

Retrieve a paginated, filterable list of catalog items.

**Endpoint**: `GET /api/catalog-items`

**Authentication**: None (public endpoint)

**Query Parameters**:

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| pageSize | integer | No | 10 | Number of items per page |
| pageIndex | integer | No | 0 | Zero-based page index |
| catalogBrandId | integer | No | null | Filter by brand ID |
| catalogTypeId | integer | No | null | Filter by type/category ID |

**Response**: `200 OK`

```json
{
  "correlationId": "550e8400-e29b-41d4-a716-446655440000",
  "catalogItems": [
    {
      "id": 1,
      "name": ".NET Bot Black Hoodie",
      "description": "Premium quality hoodie with .NET Bot design",
      "price": 19.50,
      "pictureUri": "http://localhost:5099/images/products/1.png",
      "catalogTypeId": 2,
      "catalogBrandId": 1
    },
    {
      "id": 2,
      "name": ".NET Black & White Mug",
      "description": "Classic ceramic mug with .NET logo",
      "price": 8.50,
      "pictureUri": "http://localhost:5099/images/products/2.png",
      "catalogTypeId": 1,
      "catalogBrandId": 1
    }
  ],
  "pageCount": 5
}
```

**Response Fields**:

| Field | Type | Description |
|-------|------|-------------|
| correlationId | string (GUID) | Unique identifier for request tracking |
| catalogItems | array | Array of catalog item objects |
| pageCount | integer | Total number of pages available |

**CatalogItem Object**:

| Field | Type | Description |
|-------|------|-------------|
| id | integer | Unique catalog item identifier |
| name | string | Product name |
| description | string | Product description |
| price | decimal | Product price (2 decimal places) |
| pictureUri | string | Absolute URL to product image |
| catalogTypeId | integer | Product type/category ID |
| catalogBrandId | integer | Product brand ID |

**Example Request**:

```bash
# Get first page of all items
curl -X GET "http://localhost:5099/api/catalog-items?pageSize=10&pageIndex=0"

# Filter by brand
curl -X GET "http://localhost:5099/api/catalog-items?catalogBrandId=1"

# Filter by type and brand
curl -X GET "http://localhost:5099/api/catalog-items?catalogTypeId=2&catalogBrandId=1&pageSize=20"
```

**Notes**:
- This endpoint includes a 1-second delay (likely for demonstration purposes)
- Combine filters for more specific queries
- pageCount reflects total pages based on filters and pageSize

---

### Get Catalog Item by ID

Retrieve a single catalog item by its unique identifier.

**Endpoint**: `GET /api/catalog-items/{catalogItemId}`

**Authentication**: None (public endpoint)

**Path Parameters**:

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| catalogItemId | integer | Yes | Unique catalog item ID |

**Response**: `200 OK`

```json
{
  "correlationId": "550e8400-e29b-41d4-a716-446655440001",
  "catalogItem": {
    "id": 1,
    "name": ".NET Bot Black Hoodie",
    "description": "Premium quality hoodie with .NET Bot design",
    "price": 19.50,
    "pictureUri": "http://localhost:5099/images/products/1.png",
    "catalogTypeId": 2,
    "catalogBrandId": 1
  }
}
```

**Error Responses**:

- `404 Not Found` - Catalog item does not exist

**Example Request**:

```bash
curl -X GET "http://localhost:5099/api/catalog-items/1"
```

---

### Create Catalog Item

Create a new catalog item. **Requires Administrator role.**

**Endpoint**: `POST /api/catalog-items`

**Authentication**: Required (JWT Bearer Token)

**Authorization**: Administrator role

**Request Headers**:

```
Authorization: Bearer <jwt-token>
Content-Type: application/json
```

**Request Body**:

```json
{
  "name": ".NET Core Mug",
  "description": "Ceramic mug with .NET Core logo",
  "price": 12.99,
  "catalogTypeId": 1,
  "catalogBrandId": 2,
  "pictureUri": "product-image.png"
}
```

**Request Fields**:

| Field | Type | Required | Constraints | Description |
|-------|------|----------|-------------|-------------|
| name | string | Yes | Unique, not empty | Product name |
| description | string | Yes | Not empty | Product description |
| price | decimal | Yes | > 0 | Product price |
| catalogTypeId | integer | Yes | Valid type ID | Product category |
| catalogBrandId | integer | Yes | Valid brand ID | Product brand |
| pictureUri | string | Yes | - | Image filename (placeholder will be used) |

**Response**: `201 Created`

```json
{
  "correlationId": "550e8400-e29b-41d4-a716-446655440002",
  "catalogItem": {
    "id": 13,
    "name": ".NET Core Mug",
    "description": "Ceramic mug with .NET Core logo",
    "price": 12.99,
    "pictureUri": "http://localhost:5099/images/products/eCatalog-item-default.png",
    "catalogTypeId": 1,
    "catalogBrandId": 2
  }
}
```

**Response Headers**:

```
Location: /api/catalog-items/13
```

**Error Responses**:

- `400 Bad Request` - Duplicate item name exists
- `401 Unauthorized` - Missing or invalid JWT token
- `403 Forbidden` - User lacks Administrator role

**Example Request**:

```bash
curl -X POST "http://localhost:5099/api/catalog-items" \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..." \
  -H "Content-Type: application/json" \
  -d '{
    "name": ".NET Core Mug",
    "description": "Ceramic mug with .NET Core logo",
    "price": 12.99,
    "catalogTypeId": 1,
    "catalogBrandId": 2,
    "pictureUri": "product-image.png"
  }'
```

**Security Notes**:
- Picture upload functionality is disabled for security reasons
- All new items receive a default/placeholder image
- For production, upload images to blob storage and update pictureUri separately

---

### Update Catalog Item

Update an existing catalog item. **Requires Administrator role.**

**Endpoint**: `PUT /api/catalog-items`

**Authentication**: Required (JWT Bearer Token)

**Authorization**: Administrator role

**Request Headers**:

```
Authorization: Bearer <jwt-token>
Content-Type: application/json
```

**Request Body**:

```json
{
  "id": 1,
  "name": ".NET Bot Black Hoodie - Updated",
  "description": "Premium quality hoodie with updated .NET Bot design",
  "price": 24.99,
  "catalogTypeId": 2,
  "catalogBrandId": 1
}
```

**Request Fields**:

| Field | Type | Required | Constraints | Description |
|-------|------|----------|-------------|-------------|
| id | integer | Yes | Must exist | Catalog item ID to update |
| name | string | Yes | Not empty | New product name |
| description | string | Yes | Not empty | New product description |
| price | decimal | Yes | > 0 | New product price |
| catalogTypeId | integer | Yes | Valid type ID | New product category |
| catalogBrandId | integer | Yes | Valid brand ID | New product brand |

**Response**: `200 OK`

```json
{
  "correlationId": "550e8400-e29b-41d4-a716-446655440003",
  "catalogItem": {
    "id": 1,
    "name": ".NET Bot Black Hoodie - Updated",
    "description": "Premium quality hoodie with updated .NET Bot design",
    "price": 24.99,
    "pictureUri": "http://localhost:5099/images/products/1.png",
    "catalogTypeId": 2,
    "catalogBrandId": 1
  }
}
```

**Error Responses**:

- `404 Not Found` - Catalog item does not exist
- `400 Bad Request` - Validation failure (negative price, empty fields)
- `401 Unauthorized` - Missing or invalid JWT token
- `403 Forbidden` - User lacks Administrator role

**Example Request**:

```bash
curl -X PUT "http://localhost:5099/api/catalog-items" \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..." \
  -H "Content-Type: application/json" \
  -d '{
    "id": 1,
    "name": ".NET Bot Black Hoodie - Updated",
    "description": "Premium quality hoodie with updated .NET Bot design",
    "price": 24.99,
    "catalogTypeId": 2,
    "catalogBrandId": 1
  }'
```

**Notes**:
- Picture URI is not updated through this endpoint
- Price must be positive (enforced by domain entity)
- Name and description cannot be null or empty

---

### Delete Catalog Item

Delete a catalog item. **Requires Administrator role.**

**Endpoint**: `DELETE /api/catalog-items/{catalogItemId}`

**Authentication**: Required (JWT Bearer Token)

**Authorization**: Administrator role

**Request Headers**:

```
Authorization: Bearer <jwt-token>
```

**Path Parameters**:

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| catalogItemId | integer | Yes | Catalog item ID to delete |

**Response**: `200 OK`

```json
{
  "correlationId": "550e8400-e29b-41d4-a716-446655440004"
}
```

**Error Responses**:

- `404 Not Found` - Catalog item does not exist
- `401 Unauthorized` - Missing or invalid JWT token
- `403 Forbidden` - User lacks Administrator role

**Example Request**:

```bash
curl -X DELETE "http://localhost:5099/api/catalog-items/13" \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
```

**Notes**:
- Deletion is permanent
- Consider implementing soft deletes for production systems
- May fail if item is referenced in active baskets or orders (database constraint)

---

## Catalog Brand Endpoints

### List Catalog Brands

Retrieve all catalog brands.

**Endpoint**: `GET /api/catalog-brands`

**Authentication**: None (public endpoint)

**Query Parameters**: None

**Response**: `200 OK`

```json
{
  "catalogBrands": [
    {
      "id": 1,
      "brand": ".NET"
    },
    {
      "id": 2,
      "brand": "Azure"
    },
    {
      "id": 3,
      "brand": "Visual Studio"
    },
    {
      "id": 4,
      "brand": "Other"
    }
  ]
}
```

**Response Fields**:

| Field | Type | Description |
|-------|------|-------------|
| catalogBrands | array | Array of brand objects |

**Brand Object**:

| Field | Type | Description |
|-------|------|-------------|
| id | integer | Unique brand identifier |
| brand | string | Brand name |

**Example Request**:

```bash
curl -X GET "http://localhost:5099/api/catalog-brands"
```

**Notes**:
- Returns all brands (no pagination)
- Assumes small dataset
- Used for populating filter dropdowns

---

## Catalog Type Endpoints

### List Catalog Types

Retrieve all catalog types (product categories).

**Endpoint**: `GET /api/catalog-types`

**Authentication**: None (public endpoint)

**Query Parameters**: None

**Response**: `200 OK`

```json
{
  "catalogTypes": [
    {
      "id": 1,
      "type": "Mug"
    },
    {
      "id": 2,
      "type": "T-Shirt"
    },
    {
      "id": 3,
      "type": "Sheet"
    },
    {
      "id": 4,
      "type": "USB Memory Stick"
    }
  ]
}
```

**Response Fields**:

| Field | Type | Description |
|-------|------|-------------|
| catalogTypes | array | Array of type objects |

**Type Object**:

| Field | Type | Description |
|-------|------|-------------|
| id | integer | Unique type identifier |
| type | string | Type/category name |

**Example Request**:

```bash
curl -X GET "http://localhost:5099/api/catalog-types"
```

**Notes**:
- Returns all types (no pagination)
- Assumes small dataset
- Used for populating filter dropdowns

---

## Authentication Endpoints

### Authenticate User

Authenticate with username and password to receive a JWT token.

**Endpoint**: `POST /api/authenticate`

**Authentication**: None (this is the authentication endpoint)

**Request Body**:

```json
{
  "username": "admin@microsoft.com",
  "password": "Pass@word1"
}
```

**Request Fields**:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| username | string | Yes | User email address |
| password | string | Yes | User password |

**Response**: `200 OK` (even on authentication failure)

```json
{
  "correlationId": "550e8400-e29b-41d4-a716-446655440005",
  "result": true,
  "isLockedOut": false,
  "isNotAllowed": false,
  "requiresTwoFactor": false,
  "username": "admin@microsoft.com",
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJhZG1pbkBtaWNyb3NvZnQuY29tIiwianRpIjoiMzc5NjY3MzAtYWJiZi00YjVjLWI5MjUtNzY2MzlhOWMwYzYzIiwiaWF0IjoxNzI5NzY4MDAwLCJyb2xlIjoiQWRtaW5pc3RyYXRvciIsIm5iZiI6MTcyOTc2ODAwMCwiZXhwIjoxNzI5NzcxNjAwfQ.signature"
}
```

**Response Fields**:

| Field | Type | Description |
|-------|------|-------------|
| correlationId | string | Request tracking ID |
| result | boolean | True if authentication succeeded |
| isLockedOut | boolean | True if account is locked |
| isNotAllowed | boolean | True if account is not allowed to sign in |
| requiresTwoFactor | boolean | True if two-factor auth required |
| username | string | Username from request |
| token | string | JWT token (only present if result is true) |

**Example Request**:

```bash
curl -X POST "http://localhost:5099/api/authenticate" \
  -H "Content-Type: application/json" \
  -d '{
    "username": "admin@microsoft.com",
    "password": "Pass@word1"
  }'
```

**Example Success Response**:

```json
{
  "correlationId": "550e8400-e29b-41d4-a716-446655440005",
  "result": true,
  "isLockedOut": false,
  "isNotAllowed": false,
  "requiresTwoFactor": false,
  "username": "admin@microsoft.com",
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
}
```

**Example Failure Response**:

```json
{
  "correlationId": "550e8400-e29b-41d4-a716-446655440006",
  "result": false,
  "isLockedOut": false,
  "isNotAllowed": false,
  "requiresTwoFactor": false,
  "username": "admin@microsoft.com",
  "token": null
}
```

**Example Locked Out Response**:

```json
{
  "correlationId": "550e8400-e29b-41d4-a716-446655440007",
  "result": false,
  "isLockedOut": true,
  "isNotAllowed": false,
  "requiresTwoFactor": false,
  "username": "baduser@example.com",
  "token": null
}
```

**Notes**:
- Endpoint always returns 200 OK with result flag (not 401 on failure)
- Account lockout is enabled after multiple failed attempts
- JWT token includes user claims and role information
- Token should be stored securely on client side
- Demo credentials: `admin@microsoft.com` / `Pass@word1`

**Token Usage**:

After receiving the token, include it in subsequent requests:

```bash
curl -X POST "http://localhost:5099/api/catalog-items" \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..." \
  -H "Content-Type: application/json" \
  -d '{ ... }'
```

---

## Common Response Patterns

### Correlation ID

Every response includes a `correlationId` field - a GUID used for request tracking and troubleshooting. Log this ID on the client side for support requests.

### Timestamps

All timestamps use ISO 8601 format with timezone information:
- Format: `YYYY-MM-DDTHH:mm:ss.sssZ`
- Example: `2024-10-24T14:30:00.000Z`

### Decimal Numbers

Prices and monetary values use decimal type with 2 decimal places:
- Format: `decimal(18,2)`
- Example: `19.99`

### Pagination Metadata

Paginated responses include:
- `pageCount` - Total number of pages
- Request includes `pageSize` and `pageIndex` parameters

### Empty Collections

Empty collections return as empty arrays, not null:

```json
{
  "catalogItems": [],
  "pageCount": 0
}
```

---

## Error Handling

### HTTP Status Codes

The API uses standard HTTP status codes:

| Code | Meaning | Usage |
|------|---------|-------|
| 200 | OK | Successful GET, PUT, DELETE |
| 201 | Created | Successful POST with resource creation |
| 400 | Bad Request | Validation error, duplicate data |
| 401 | Unauthorized | Missing or invalid authentication |
| 403 | Forbidden | Valid auth but insufficient permissions |
| 404 | Not Found | Resource doesn't exist |
| 500 | Internal Server Error | Server-side error |

### Error Response Format

Error responses follow Problem Details format (RFC 7807):

```json
{
  "type": "https://tools.ietf.org/html/rfc7231#section-6.5.1",
  "title": "One or more validation errors occurred.",
  "status": 400,
  "traceId": "00-123abc-456def-00",
  "errors": {
    "Name": ["The Name field is required."],
    "Price": ["Price must be greater than 0."]
  }
}
```

### Common Error Scenarios

#### Invalid JWT Token

```http
HTTP/1.1 401 Unauthorized
```

**Solution**: Re-authenticate to obtain new token

#### Insufficient Permissions

```http
HTTP/1.1 403 Forbidden
```

**Solution**: Ensure user has required role (e.g., Administrator)

#### Validation Failure

```http
HTTP/1.1 400 Bad Request
Content-Type: application/json

{
  "title": "Validation failed",
  "status": 400,
  "errors": {
    "Price": ["Price must be greater than zero"]
  }
}
```

**Solution**: Fix validation errors and retry

#### Resource Not Found

```http
HTTP/1.1 404 Not Found
```

**Solution**: Verify resource ID is correct

---

## Rate Limiting

Currently, the API does not implement rate limiting. For production deployments, consider implementing:

- Per-IP rate limiting
- Per-user rate limiting (authenticated requests)
- Different limits for different endpoints
- Rate limit headers in responses

---

## CORS Configuration

The API supports CORS (Cross-Origin Resource Sharing) for browser-based clients. Allowed origins are configured in `appsettings.json`.

**Default Development CORS**:
- Origins: `http://localhost:3000`, `http://localhost:5000`
- Methods: GET, POST, PUT, DELETE
- Headers: Authorization, Content-Type

---

## Versioning

The current API is unversioned (`/api/` prefix). Future versions may use:

- URL versioning: `/api/v2/catalog-items`
- Header versioning: `X-API-Version: 2`
- Media type versioning: `Accept: application/vnd.eshoponweb.v2+json`

---

## OpenAPI / Swagger

### Swagger UI

Interactive API documentation: `http://localhost:5099/swagger`

### OpenAPI Specification

Download OpenAPI spec: `http://localhost:5099/swagger/v1/swagger.json`

### Swagger Annotations

Endpoints use Swashbuckle annotations for documentation:
- Operation summaries and descriptions
- Response type documentation
- Tag-based organization

---

## SDK / Client Libraries

### Recommended HTTP Clients

- **.NET**: `HttpClient` with `System.Net.Http.Json`
- **JavaScript**: `fetch` API or `axios`
- **Python**: `requests` library
- **Java**: `HttpClient` or `OkHttp`

### Example .NET Client

```csharp
public class CatalogApiClient
{
    private readonly HttpClient _httpClient;
    private readonly string _baseUrl;

    public CatalogApiClient(HttpClient httpClient, string baseUrl)
    {
        _httpClient = httpClient;
        _baseUrl = baseUrl;
    }

    public async Task<ListPagedCatalogItemResponse> GetCatalogItemsAsync(
        int pageSize = 10,
        int pageIndex = 0,
        int? brandId = null,
        int? typeId = null)
    {
        var query = $"pageSize={pageSize}&pageIndex={pageIndex}";
        if (brandId.HasValue) query += $"&catalogBrandId={brandId}";
        if (typeId.HasValue) query += $"&catalogTypeId={typeId}";

        var response = await _httpClient.GetAsync($"{_baseUrl}/api/catalog-items?{query}");
        response.EnsureSuccessStatusCode();

        return await response.Content.ReadFromJsonAsync<ListPagedCatalogItemResponse>();
    }

    public async Task<AuthenticateResponse> AuthenticateAsync(string username, string password)
    {
        var request = new { username, password };
        var response = await _httpClient.PostAsJsonAsync($"{_baseUrl}/api/authenticate", request);
        return await response.Content.ReadFromJsonAsync<AuthenticateResponse>();
    }

    public void SetBearerToken(string token)
    {
        _httpClient.DefaultRequestHeaders.Authorization =
            new System.Net.Http.Headers.AuthenticationHeaderValue("Bearer", token);
    }
}
```

### Example JavaScript Client

```javascript
class CatalogApiClient {
  constructor(baseUrl) {
    this.baseUrl = baseUrl;
    this.token = null;
  }

  async authenticate(username, password) {
    const response = await fetch(`${this.baseUrl}/api/authenticate`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ username, password })
    });

    const data = await response.json();
    if (data.result && data.token) {
      this.token = data.token;
    }
    return data;
  }

  async getCatalogItems(pageSize = 10, pageIndex = 0, brandId = null, typeId = null) {
    const params = new URLSearchParams({
      pageSize: pageSize.toString(),
      pageIndex: pageIndex.toString()
    });

    if (brandId) params.append('catalogBrandId', brandId.toString());
    if (typeId) params.append('catalogTypeId', typeId.toString());

    const response = await fetch(`${this.baseUrl}/api/catalog-items?${params}`);
    return await response.json();
  }

  async createCatalogItem(item) {
    const response = await fetch(`${this.baseUrl}/api/catalog-items`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${this.token}`
      },
      body: JSON.stringify(item)
    });

    if (!response.ok) {
      throw new Error(`HTTP ${response.status}: ${response.statusText}`);
    }

    return await response.json();
  }
}

// Usage
const client = new CatalogApiClient('http://localhost:5099');
await client.authenticate('admin@microsoft.com', 'Pass@word1');
const items = await client.getCatalogItems();
```

---

## Testing

### Test Accounts

**Administrator**:
- Username: `admin@microsoft.com`
- Password: `Pass@word1`
- Roles: Administrator

**Regular User**:
- Username: `demouser@microsoft.com`
- Password: `Pass@word1`
- Roles: (none)

### Postman Collection

A Postman collection is recommended for API testing. Example collection structure:

- **Auth** folder
  - Authenticate (POST /api/authenticate)
- **Catalog Items** folder
  - List Items (GET /api/catalog-items)
  - Get Item by ID (GET /api/catalog-items/{id})
  - Create Item (POST /api/catalog-items) [requires auth]
  - Update Item (PUT /api/catalog-items) [requires auth]
  - Delete Item (DELETE /api/catalog-items/{id}) [requires auth]
- **Catalog Brands** folder
  - List Brands (GET /api/catalog-brands)
- **Catalog Types** folder
  - List Types (GET /api/catalog-types)

**Environment Variables**:
- `baseUrl`: `http://localhost:5099`
- `token`: (set after authentication)

---

## Security Considerations

### HTTPS in Production

Always use HTTPS in production to protect:
- Authentication credentials
- JWT tokens
- Sensitive data

### Token Storage

- **Web Apps**: Store tokens in memory or httpOnly cookies (not localStorage)
- **Mobile Apps**: Use secure storage (Keychain, KeyStore)
- **Never**: Expose tokens in URLs or logs

### SQL Injection

The API uses Entity Framework Core with parameterized queries, providing protection against SQL injection.

### CSRF Protection

API uses JWT tokens (not cookies), providing natural CSRF protection for authenticated requests.

### Input Validation

All inputs are validated:
- Required fields
- Data type validation
- Business rule validation (e.g., positive prices)

---

## Related Documentation

- [Component Catalog](./component-catalog.md)
- [Data Dictionary](./data-dictionary.md)
- [Technology Stack](./technology-stack.md)
- [Technical Architecture](../architecture/technical-architecture.md)
