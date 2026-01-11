# Documentation & OpenAPI

## What You'll Learn
How to create excellent API documentation using OpenAPI specifications, contract-first development, and developer experience best practices. You'll understand how to make your API discoverable, understandable, and easy to integrate.

## Why This Matters
Documentation is the first (and often only) interaction developers have with your API before integration. Poor docs mean frustrated developers, support tickets, failed integrations, and abandoned adoption. Great docs accelerate onboarding from days to hours. OpenAPI provides a machine-readable contract that generates docs, validates requests, mocks servers, and generates client SDKs. Documentation isn't an afterthought—it's a product.

## Contract-First Development

### The Approach

**Design API spec first** → Review & iterate → Generate code → Implement → Validate against spec

**Benefits**:
- **Design review before coding**: Catch issues early
- **Parallel development**: Frontend/backend teams work simultaneously with mocks
- **Consistency**: Spec is single source of truth
- **Automation**: Generate validators, clients, docs from spec

### OpenAPI 3.x Structure

```yaml
openapi: 3.0.3
info:
  title: Orders API
  version: 1.0.0
  description: RESTful API for order management
  contact:
    name: API Support
    email: api@example.com
    url: https://api.example.com/support
  license:
    name: Apache 2.0
    url: https://www.apache.org/licenses/LICENSE-2.0

servers:
  - url: https://api.example.com/v1
    description: Production
  - url: https://sandbox.api.example.com/v1
    description: Sandbox

paths:
  /orders:
    get:
      summary: List orders
      description: Returns a paginated list of orders with optional filtering
      operationId: listOrders
      tags: [Orders]
      parameters:
        - name: status
          in: query
          description: Filter by order status
          schema:
            type: string
            enum: [pending, processing, shipped, delivered, cancelled]
        - name: cursor
          in: query
          description: Pagination cursor
          schema:
            type: string
        - name: limit
          in: query
          description: Page size (max 100)
          schema:
            type: integer
            minimum: 1
            maximum: 100
            default: 20
      responses:
        '200':
          description: Successful response
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/OrderListResponse'
              examples:
                success:
                  $ref: '#/components/examples/OrderListSuccess'
        '400':
          $ref: '#/components/responses/BadRequest'
        '401':
          $ref: '#/components/responses/Unauthorized'
        '429':
          $ref: '#/components/responses/TooManyRequests'
      security:
        - bearerAuth: []

components:
  schemas:
    Order:
      type: object
      required: [id, customerId, status, total, createdAt]
      properties:
        id:
          type: string
          format: uuid
          example: 550e8400-e29b-41d4-a716-446655440000
        customerId:
          type: string
          example: cust-12345
        status:
          type: string
          enum: [pending, processing, shipped, delivered, cancelled]
        total:
          type: number
          format: decimal
          minimum: 0
          example: 99.99
        currency:
          type: string
          pattern: '^[A-Z]{3}$'
          example: USD
        items:
          type: array
          minItems: 1
          maxItems: 100
          items:
            $ref: '#/components/schemas/OrderItem'
        createdAt:
          type: string
          format: date-time
          example: '2026-01-11T10:30:00Z'

    OrderListResponse:
      type: object
      required: [data, pagination]
      properties:
        data:
          type: array
          items:
            $ref: '#/components/schemas/Order'
        pagination:
          $ref: '#/components/schemas/PaginationMetadata'

    PaginationMetadata:
      type: object
      properties:
        nextCursor:
          type: string
          nullable: true
        hasNext:
          type: boolean
        limit:
          type: integer

  securitySchemes:
    bearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT
      description: JWT token obtained from /auth/token endpoint

  responses:
    BadRequest:
      description: Invalid request
      content:
        application/problem+json:
          schema:
            $ref: '#/components/schemas/ProblemDetails'
    
  examples:
    OrderListSuccess:
      value:
        data:
          - id: ord-123
            customerId: cust-456
            status: shipped
            total: 99.99
            currency: USD
            createdAt: '2026-01-11T10:00:00Z'
        pagination:
          nextCursor: eyJpZCI6Im9yZC0xMjMifQ==
          hasNext: true
          limit: 20
```

### Component Reuse

**Schemas**: Define once, reference everywhere
```yaml
components:
  schemas:
    ErrorResponse:
      # ... definition

paths:
  /orders:
    get:
      responses:
        '400':
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
```

**Parameters**: Common query params
```yaml
components:
  parameters:
    CursorParam:
      name: cursor
      in: query
      schema:
        type: string
    LimitParam:
      name: limit
      in: query
      schema:
        type: integer
        minimum: 1
        maximum: 100
        default: 20

paths:
  /orders:
    get:
      parameters:
        - $ref: '#/components/parameters/CursorParam'
        - $ref: '#/components/parameters/LimitParam'
```

## Developer Experience (DX)

### Essential Elements

1. **Quick Start**: Get first API call working in 5 minutes
2. **Examples**: Real, runnable examples for every endpoint
3. **Error Guide**: What each error means and how to fix
4. **Use Cases**: Common scenarios with complete flows
5. **SDKs**: Client libraries in popular languages

### Example-Driven Documentation

**Every endpoint needs**:
- Request example (curl, language SDKs)
- Success response example
- Error response examples
- Explanation of parameters

```markdown
### Create Order

Creates a new order for a customer.

**Request**
```bash
curl -X POST https://api.example.com/v1/orders \
  -H 'Authorization: Bearer eyJhbGc...' \
  -H 'Content-Type: application/json' \
  -H 'Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000' \
  -d '{
    "customerId": "cust-123",
    "items": [
      {"productId": "prod-456", "quantity": 2}
    ]
  }'
```

**Success Response (201 Created)**
```json
{
  "id": "ord-789",
  "customerId": "cust-123",
  "status": "pending",
  "total": 49.98,
  "currency": "USD",
  "createdAt": "2026-01-11T10:30:00Z"
}
```

**Error Response (422 Unprocessable Entity)**
```json
{
  "type": "https://api.example.com/errors/validation-error",
  "title": "Validation Failed",
  "status": 422,
  "errors": [
    {
      "field": "items[0].quantity",
      "code": "OUT_OF_STOCK",
      "message": "Product prod-456 has only 1 unit in stock"
    }
  ]
}
```
```

### Interactive Documentation

**Swagger UI**: Try API in browser
- OAuth flows built-in
- Request/response inspection
- No code required

**ReDoc**: Beautiful, three-panel layout
- Better for complex APIs
- Search and navigation
- Print-friendly

### Quickstart Template

```markdown
# Quickstart

Get your first API call running in 5 minutes.

## 1. Get API Key

Sign up at [dashboard.example.com](https://dashboard.example.com) and generate an API key.

## 2. Make Your First Request

```bash
curl https://api.example.com/v1/orders \
  -H 'Authorization: Bearer YOUR_API_KEY'
```

## 3. Create an Order

```bash
curl -X POST https://api.example.com/v1/orders \
  -H 'Authorization: Bearer YOUR_API_KEY' \
  -H 'Content-Type: application/json' \
  -d '{
    "customerId": "cust-123",
    "items": [{"productId": "prod-456", "quantity": 1}]
  }'
```

## Next Steps

- [Authentication Guide](./auth.md)
- [Pagination](./pagination.md)
- [Error Handling](./errors.md)
- [SDKs](./sdks.md)
```

## Governance & Quality

### Style Guide

Document and enforce:
- Resource naming (plural, lowercase, hyphens)
- Field naming (camelCase)
- Date formats (ISO 8601 UTC)
- Pagination pattern (cursor vs offset)
- Error format (RFC 7807)
- Versioning strategy

### Linting with Spectral

```yaml
# .spectral.yaml
extends: [[spectral:oas, recommended]]

rules:
  operation-operationId:
    description: Every operation must have operationId
    severity: error
    
  operation-tag-defined:
    description: Operation tags must be defined in global tags
    severity: warn
    
  oas3-valid-example:
    description: Examples must be valid against schema
    severity: error
    
  pagination-pattern:
    description: List endpoints must support pagination
    given: $.paths[*][get]
    then:
      - field: parameters
        function: schema
        functionOptions:
          schema:
            type: array
            contains:
              anyOf:
                - properties:
                    name:
                      const: cursor
                - properties:
                    name:
                      const: page
```

**Run in CI**:
```bash
spectral lint openapi.yaml --fail-severity warn
```

## Change Management

### Changelog Per Version

```markdown
# Changelog

## v2.1.0 (2026-01-15)

### Added
- New endpoint: `GET /orders/{id}/shipments`
- Optional `trackingNumber` field in shipment response

### Changed
- Increased max page size from 50 to 100

### Deprecated
- `GET /orders/{id}/tracking` - use `/orders/{id}/shipments` instead
  - Sunset date: 2026-07-15

### Fixed
- Fixed pagination cursor stability with concurrent inserts

## v2.0.0 (2025-12-01) - BREAKING

### Breaking Changes
- Renamed `customer_id` to `customerId` in all responses
- Changed `amount` from number to string (decimal precision)
- Removed deprecated `GET /order` endpoint
```

### Deprecation Annotations

```yaml
paths:
  /orders/{id}/tracking:
    get:
      deprecated: true
      description: |
        ⚠️ **DEPRECATED**: This endpoint is deprecated and will be removed on 2026-07-15.
        Please migrate to `GET /orders/{id}/shipments`.
        
        [Migration Guide](https://docs.example.com/migration-v2)
      x-sunset-date: '2026-07-15'
```

## Discoverability

### API Catalog/Portal

Centralized hub:
- Browse all APIs
- Search by capability
- Getting started guides
- Code examples
- SDKs and tools
- Support resources
- Status page

### Internal Developer Portal (Backstage, etc.)

- API ownership and SLAs
- Dependency graphs
- Metrics and health
- Deployment status
- Cost attribution
