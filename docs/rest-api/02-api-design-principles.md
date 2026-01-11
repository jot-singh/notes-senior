# API Design Principles

## What You'll Learn
Practical patterns for designing intuitive, consistent, and evolvable REST APIs. You'll learn how to model resources, design URLs, structure requests/responses, and maintain compatibility as your API evolves.

## Why This Matters
Good API design is about developer experience (DX). Well-designed APIs are self-documenting, predictable, and easy to use correctly. Poor design leads to support burden, integration bugs, and client frustration. As a senior developer, you're responsible for APIs that thousands of developers might consume—design choices have compound effects.

## URL & Resource Naming

### Core Principles
**Think Resources, Not Actions**: REST is resource-oriented. URLs identify "things" (nouns), HTTP methods specify actions (verbs).

### Pattern: Plural Collections
- Collections: `/orders`, `/customers`, `/products`
- Individual resource: `/orders/{orderId}`, `/customers/{customerId}`

**Why Plural**: Consistency—whether fetching one or many, the base path is the same. `/orders` (list) vs `/orders/123` (specific order).

### Pattern: Hierarchical Relationships
For containment or strong ownership:
- `/customers/{customerId}/orders` - orders belonging to a customer
- `/orders/{orderId}/items` - line items within an order

**When to Use**: Clear parent-child relationship where child doesn't make sense without parent.
**When to Avoid**: Loosely coupled resources; prefer top-level + filtering: `/orders?customerId=123`.

### Avoiding Verbs in URLs
**Good**: `POST /orders` (create), `DELETE /orders/123` (cancel/delete)
**Acceptable**: `POST /orders/123/cancel` (non-CRUD operation)
**Bad**: `POST /createOrder`, `GET /getOrder/123`

**Why**: HTTP methods already express intent. Verb-based URLs duplicate this and break REST conventions.

### Casing & Formatting
- **Recommended**: lowercase, hyphen-separated: `/customer-orders`, `/order-items`
- **Avoid**: camelCase (`/customerOrders`), snake_case (`/customer_orders`)
- **Why**: URL standards recommend lowercase; hyphens are more readable and URL-safe

### Real-World Example
```
Good Design:
GET    /api/v1/customers/c-12345/orders          # List orders for customer
POST   /api/v1/orders                             # Create new order
GET    /api/v1/orders/ord-67890                   # Get specific order
PATCH  /api/v1/orders/ord-67890                   # Update order
POST   /api/v1/orders/ord-67890/cancel            # Cancel order (custom action)
GET    /api/v1/orders/ord-67890/shipments         # Get shipments for order
```

## Request & Response Shapes
- JSON by default; avoid envelope unless needed; include metadata minimally.
- Consistent field naming (camelCase) and types; avoid polymorphic surprises.
- Timestamps: ISO 8601 UTC; money: decimal strings with currency codes.
- Localization: separate localized fields; avoid mixing in core entities.

## Pagination, Filtering, Sorting
- Offset vs cursor; prefer cursor for large datasets and stable pagination.
- Sorting: `sort=createdAt:desc`; Filtering: `status=pending&customerId=...`.
- Provide `Link` headers for next/prev; include total counts only if cheap or cached.

## Partial Responses & Expansions
- Field selection: `fields=summary,total,items(id,sku)`.
- Expansions: `expand=customer,shipments` with bounds to avoid N+1 explosions.

## Consistency & Compatibility
- Additive changes are non-breaking: new fields must be optional and ignored by old clients.
- Renames/removals are breaking: require version bump and deprecation plan.
- Contract-first via OpenAPI; lint for consistency.

## Action Semantics
- Use POST for non-idempotent operations (create, execute).
- Use PUT for full replacement; PATCH for partial updates.
- Use 202 Accepted for async operations with status endpoint.

## Headers & Metadata
- Correlation IDs: `X-Request-Id`; trace headers (W3C Trace Context).
- Rate limit headers: `RateLimit-Limit`, `RateLimit-Remaining`, `RateLimit-Reset`.

## Checklists
- Uniform error format (RFC 7807).
- Stable pagination across filters/sorts.
- Clear semantics for idempotency.
- Document limits: max page size, payload size.
