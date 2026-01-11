# Versioning Strategies

## What You'll Learn
How to evolve APIs without breaking existing clients. You'll understand different versioning approaches, compatibility rules, deprecation processes, and how to manage multiple API versions in production.

## Why This Matters
APIs are contracts. Breaking them breaks every client integration, costing customers time and money. Yet APIs must evolve—new features, performance improvements, security updates. Versioning strategy determines how fast you can innovate while maintaining trust. Poor versioning leads to sprawl (100+ versions), breaking changes disguised as minor updates, or stagnation. Senior developers balance evolution with stability.

## Versioning Approaches

### 1. URI Versioning

**Format**: Version in URL path
```
https://api.example.com/v1/orders
https://api.example.com/v2/orders
```

**Advantages**:
- **Explicit**: Version immediately visible
- **Simple**: Easy to route, cache, document
- **Browser-friendly**: Testable in browser, works with all tools
- **Gateway routing**: Route /v1/* to old service, /v2/* to new

**Disadvantages**:
- **Coarse-grained**: Entire API versioned together
- **URL changes**: Breaks caching if URLs change
- **Resource duplication**: Same resource at multiple URLs

**When to Use**:
- Public APIs
- Breaking changes affect many endpoints
- Simple infrastructure
- Need caching at CDN

**Implementation**:
```
/v1/orders → OrderServiceV1
/v2/orders → OrderServiceV2 (new schema, breaking changes)
/v2/customers → CustomerService (v2 but compatible with v1)
```

### 2. Header Versioning

**Format**: Version in Accept header (media type)
```
GET /orders
Accept: application/vnd.example.v2+json
```

or custom header:
```
GET /orders
API-Version: 2
```

**Advantages**:
- **Fine-grained**: Version per resource type
- **Stable URLs**: Same URL, different representations
- **HTTP-semantic**: Leverages content negotiation
- **Multiple versions**: Same endpoint can serve v1, v2, v3

**Disadvantages**:
- **Less visible**: Not in URL, harder to test
- **Browser unfriendly**: Can't easily test in browser
- **Caching complexity**: Must use Vary header correctly

**When to Use**:
- Internal APIs with sophisticated clients
- Gradual migration (clients upgrade piecemeal)
- Hypermedia APIs (HATEOAS)
- Per-resource evolution

**Implementation**:
```java
@GetMapping(value = "/orders", produces = "application/vnd.example.v1+json")
public List<OrderV1> getOrdersV1() { ... }

@GetMapping(value = "/orders", produces = "application/vnd.example.v2+json")
public List<OrderV2> getOrdersV2() { ... }
```

### 3. Query Parameter Versioning

**Format**: `?version=2` or `?api-version=2`

**Advantages**: Simple, visible, easy to test
**Disadvantages**: Mixes versioning with filtering, pollutes query space
**Recommendation**: Avoid; use URI or header

### 4. No Versioning (Evolution Only)

**Approach**: Never break compatibility; only additive changes.

**Rules**:
- New fields always optional
- Never rename or remove fields (deprecate and add new)
- Semantic changes require new endpoints

**When to Use**:
- Internal APIs with tight control
- Very stable domains
- Strong contract testing

**Pitfall**: Eventually need breaking changes; delayed cost explodes.

## Compatibility Rules

### Non-Breaking (Backward Compatible) Changes

Clients continue working without modification:

✅ **Add optional request field**
```json
// V1
{"email": "user@example.com"}

// V2 (compatible)
{"email": "user@example.com", "phone": "+1234567890"}  // phone optional
```

✅ **Add response field**
```json
// V1
{"id": "123", "status": "pending"}

// V2 (compatible)
{"id": "123", "status": "pending", "trackingNumber": "TRK-789"}  // clients ignore
```

✅ **Add new endpoint**
```
POST /v1/orders       (existing)
POST /v1/bulk-orders  (new)
```

✅ **Add new optional query parameter**
```
GET /orders?status=pending           (v1)
GET /orders?status=pending&urgent=true  (v2, urgent is optional)
```

✅ **Widen validation** (accept more)
```
// V1: email required, 1-50 chars
// V2: email required, 1-100 chars  (accepts old + new)
```

✅ **Add new enum value** (if clients use open enums)
```json
// V1 status: "pending", "completed"
// V2 status: "pending", "processing", "completed"  // if clients allow unknown
```

### Breaking (Incompatible) Changes

Require client updates:

❌ **Remove or rename field**
```json
// V1
{"customerId": "123"}

// V2 (BREAKING)
{"customer_id": "123"}  // renamed
```

❌ **Change field type**
```json
// V1
{"amount": 100}  // number

// V2 (BREAKING)
{"amount": "100.00"}  // string
```

❌ **Make optional field required**
```json
// V1: phone optional
// V2: phone required (BREAKING for clients not sending it)
```

❌ **Narrow validation** (reject previously valid input)
```
// V1: email 1-100 chars
// V2: email 1-50 chars (BREAKING)
```

❌ **Change status codes**
```
// V1: 404 when not found
// V2: 204 when not found (BREAKING - clients check 404)
```

❌ **Change endpoint semantics**
```
// V1: POST /orders is synchronous
// V2: POST /orders is async, returns 202 (BREAKING)
```

❌ **Remove endpoint**

## Semantic Versioning for APIs

**Format**: MAJOR.MINOR.PATCH

- **MAJOR**: Breaking changes (v1 → v2)
- **MINOR**: New features, backward compatible (v1.1 → v1.2)
- **PATCH**: Bug fixes, no API changes (v1.1.0 → v1.1.1)

**Communication**:
```
v2.3.1
^ Major: Breaking changes, requires client update
  ^ Minor: New optional features
    ^ Patch: Bug fixes, transparent
```
