# Error Handling & Status Codes

## What You'll Learn
How to design error responses that are consistent, actionable, and debuggable. You'll learn status code semantics, RFC 7807 Problem Details, and how to handle errors in distributed systems.

## Why This Matters
Poor error handling causes integration nightmares: clients can't distinguish temporary from permanent errors, missing context blocks debugging, inconsistent formats require special handling per endpoint. Good error design makes your API self-documenting and reduces support burden. When things fail (and they will), clear errors are the difference between a 5-minute fix and a 5-hour debugging session.

## Principles of Good Error Handling

### 1. Deterministic & Consistent
**Same error, same shape**: Every error response follows the same structure. Clients parse once, handle everywhere.

**Bad**:
```json
// Endpoint A:
{"error": "Invalid email"}

// Endpoint B:
{"message": "Email validation failed", "code": 400}

// Endpoint C:
{"errors": [{"field": "email", "issue": "format"}]}
```

**Good**: Use RFC 7807 everywhere.

### 2. Machine-Readable
**Include error codes**: Stable identifiers clients can switch on.

```json
{
  "type": "https://api.example.com/errors/validation-error",
  "code": "INVALID_EMAIL_FORMAT",
  "status": 422
}
```

Clients can handle: `if (error.code === 'INVALID_EMAIL_FORMAT') { showEmailHelp(); }`

### 3. Safe Detail
**Don't leak internals**: Avoid stack traces, SQL queries, internal service names in production.

**Bad**:
```json
{
  "error": "SQL error: column 'usr_emai' does not exist",
  "trace": "at OrderService.java:145..."
}
```

**Good**:
```json
{
  "type": "https://api.example.com/errors/internal-error",
  "status": 500,
  "detail": "An unexpected error occurred. Contact support with request ID.",
  "instance": "/orders",
  "request_id": "req-abc123"
}
```

### 4. Correlation
**Include request/trace IDs**: Essential for distributed debugging.

## RFC 7807 Problem Details

### Standard Fields

**`type`** (string, URI):
- Identifies error type
- Should be stable, dereferenceable documentation URL
- Example: `https://api.example.com/errors/insufficient-balance`

**`title`** (string):
- Short, human-readable summary
- Should not change for same error type
- Example: `"Insufficient Balance"`

**`status`** (integer):
- HTTP status code for convenience
- Must match response status

**`detail`** (string):
- Human-readable explanation specific to this occurrence
- Can include instance-specific information
- Example: `"Your account balance of $45.00 is insufficient for a $100.00 charge."`

**`instance`** (string, URI):
- Identifies specific occurrence
- Often the request path or unique error ID
- Example: `"/orders/ord-12345"`

### Extension Fields

Add domain-specific fields:

```json
{
  "type": "https://api.example.com/errors/validation-error",
  "title": "Validation Failed",
  "status": 422,
  "detail": "One or more fields failed validation",
  "instance": "/orders",
  "errors": [
    {
      "field": "email",
      "code": "INVALID_FORMAT",
      "message": "Must be a valid email address",
      "value": "not-an-email"
    },
    {
      "field": "amount",
      "code": "OUT_OF_RANGE",
      "message": "Must be between 0.01 and 10000.00",
      "value": -5
    }
  ],
  "request_id": "req-abc123",
  "trace_id": "trace-xyz789"
}
```

### Full Example Response

```http
HTTP/1.1 422 Unprocessable Entity
Content-Type: application/problem+json
X-Request-ID: req-abc123

{
  "type": "https://api.example.com/errors/validation-error",
  "title": "Invalid request",
  "status": 422,
  "detail": "Field 'email' must be a valid address",
  "instance": "/orders/123",
  "errors": [
    {"field": "email", "message": "Invalid format"}
  ],
  "request_id": "req-abc123"
}
```

## Status Code Mapping & Decision Making

### Understanding Status Families

**2xx Success**: Request understood and processed successfully.
**4xx Client Error**: Problem with request; client should fix before retrying.
**5xx Server Error**: Server failed; client might retry (maybe with backoff).

### Detailed Status Code Guide

#### 200 OK
**Use**: Successful GET, PUT, PATCH, DELETE (when returning body).
```http
GET /orders/123
→ 200 OK
{"id": "123", "status": "shipped"}
```

#### 201 Created
**Use**: Successful POST that creates a resource.
**Include**: `Location` header with new resource URL.
```http
POST /orders
→ 201 Created
Location: /orders/123
{"id": "123", "status": "pending"}
```

#### 204 No Content
**Use**: Successful request with no response body.
**Common**: DELETE operations, PATCH when not returning updated resource.
```http
DELETE /orders/123
→ 204 No Content
```

#### 400 Bad Request
**Use**: Malformed request (invalid JSON, missing required fields, type errors).
**Not for**: Business logic validation (use 422).
```json
// Request: {"amount": "not-a-number"}
{
  "type": "https://api.example.com/errors/bad-request",
  "title": "Bad Request",
  "status": 400,
  "detail": "Field 'amount' must be a number"
}
```

#### 401 Unauthorized
**Use**: Authentication required or failed.
**Include**: `WWW-Authenticate` header.
**Name Confusion**: Actually means "unauthenticated" (not logged in).
```http
GET /orders
Authorization: Bearer invalid-token
→ 401 Unauthorized
WWW-Authenticate: Bearer error="invalid_token"
```

#### 403 Forbidden
**Use**: Authenticated but not authorized (lacks permission).
**Meaning**: "I know who you are, but you can't do this."
```json
// User authenticated but lacks permission
{
  "type": "https://api.example.com/errors/forbidden",
  "title": "Forbidden",
  "status": 403,
  "detail": "Requires 'admin' role"
}
```

#### 404 Not Found
**Use**: Resource doesn't exist.
**Avoid**: Returning 200 with `{"error": "not found"}` envelope.
```http
GET /orders/nonexistent
→ 404 Not Found
```

**Security Consideration**: Don't reveal if resource exists when user lacks permission. Return 404 instead of 403 to prevent enumeration.

#### 409 Conflict
**Use**: Request conflicts with current state.
**Examples**: Duplicate creation, state transition violation.
```json
// Trying to cancel an already-shipped order
{
  "type": "https://api.example.com/errors/state-conflict",
  "title": "Invalid State Transition",
  "status": 409,
  "detail": "Cannot cancel order in 'shipped' status",
  "current_status": "shipped"
}
```

#### 412 Precondition Failed
**Use**: Conditional request failed (If-Match, If-None-Match).
**Common**: Optimistic locking with ETags.
```http
PATCH /orders/123
If-Match: "etag-v1"

→ 412 Precondition Failed
ETag: "etag-v2"

// Order was modified by another request
```

#### 422 Unprocessable Entity
**Use**: Syntactically correct but semantically invalid.
**Examples**: Business rule violations, validation errors.
```json
// Valid JSON, but business rules violated
{
  "type": "https://api.example.com/errors/validation-error",
  "title": "Validation Error",
  "status": 422,
  "detail": "Order amount exceeds customer credit limit",
  "errors": [
    {
      "field": "amount",
      "code": "EXCEEDS_LIMIT",
      "message": "$500.00 exceeds credit limit of $250.00"
    }
  ]
}
```

#### 429 Too Many Requests
**Use**: Rate limit exceeded.
**Include**: `Retry-After` header.
```http
GET /orders
→ 429 Too Many Requests
Retry-After: 60
RateLimit-Limit: 100
RateLimit-Remaining: 0
RateLimit-Reset: 1673451600

{
  "type": "https://api.example.com/errors/rate-limit-exceeded",
  "title": "Rate Limit Exceeded",
  "status": 429,
  "detail": "API rate limit of 100 requests per minute exceeded",
  "retry_after": 60
}
```

#### 500 Internal Server Error
**Use**: Unexpected server error.
**Hide details**: Don't expose stack traces.
```json
{
  "type": "https://api.example.com/errors/internal-error",
  "title": "Internal Server Error",
  "status": 500,
  "detail": "An unexpected error occurred",
  "request_id": "req-abc123"
}
```

#### 502 Bad Gateway
**Use**: Upstream service returned invalid response.
**Context**: API gateway or proxy received bad response from backend.

#### 503 Service Unavailable
**Use**: Service temporarily unavailable (maintenance, overload).
**Include**: `Retry-After` header.
```http
→ 503 Service Unavailable
Retry-After: 300

{
  "type": "https://api.example.com/errors/service-unavailable",
  "title": "Service Unavailable",
  "status": 503,
  "detail": "Service undergoing maintenance",
  "retry_after": 300
}
```

#### 504 Gateway Timeout
**Use**: Upstream service didn't respond in time.
**Context**: Gateway waited but backend never responded.

### Decision Tree: Which Status Code?

```
Request arrives
├─ Is JSON parseable? NO → 400 Bad Request
├─ Is authentication provided? NO → 401 Unauthorized
├─ Is authentication valid? NO → 401 Unauthorized
├─ Does user have permission? NO → 403 Forbidden (or 404 for security)
├─ Does resource exist? NO → 404 Not Found
├─ Is request syntax valid? NO → 400 Bad Request
├─ Did ETag/version check fail? YES → 412 Precondition Failed
├─ Is rate limit exceeded? YES → 429 Too Many Requests
├─ Are business rules satisfied? NO → 422 Unprocessable Entity
├─ Is there a state conflict? YES → 409 Conflict
├─ Did server error occur? YES → 500 Internal Server Error
└─ Success → 200/201/204
```

## Retries & Idempotency
- Mark retryable vs non-retryable errors explicitly.
- Use `idempotency-key` header for repeatable POST operations.

## Localization
- Provide a stable machine code; messages optionally localized via `Accept-Language` or client-side.

## Logging
- Structure logs with error codes, request IDs, user/tenant context; mask sensitive fields.
