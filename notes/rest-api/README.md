# REST API Notes (Senior Developer Level)

Comprehensive notes covering REST API design, implementation, security, and operations from a senior software engineering perspective.

## Contents

### Fundamentals & Design
1. [REST API Fundamentals](01-rest-api-fundamentals.md) - REST constraints, HTTP semantics, status codes, resource modeling
2. [API Design Principles](02-api-design-principles.md) - URL design, request/response patterns, pagination, filtering, sorting
3. [Versioning Strategies](07-versioning-strategies.md) - URI vs header versioning, compatibility rules, deprecation lifecycle

### Security & Authorization
4. [Authentication & Authorization](03-authentication-authorization.md) - OAuth 2.0, JWT, API keys, mTLS, token management
5. [Security Best Practices](04-security-best-practices.md) - Transport security, input validation, threat modeling, incident response
6. [Rate Limiting, Throttling, and Quotas](12-rate-limiting-throttling-quotas.md) - Token bucket, leaky bucket, quota management

### Reliability & Performance
7. [Error Handling & Status Codes](05-error-handling-and-status-codes.md) - RFC 7807 problem details, status code mapping, retry semantics
8. [Idempotency & Concurrency Control](09-idempotency-and-concurrency-control.md) - Idempotency keys, optimistic locking, ETags
9. [Caching, ETags, and Performance](08-caching-etags-and-performance.md) - HTTP caching, validators, conditional requests
10. [Pagination, Filtering, Sorting](06-pagination-filtering-sorting.md) - Cursor vs offset pagination, stable ordering, metadata

### Advanced Patterns
11. [Asynchronous Operations](16-async-operations.md) - Long-running tasks, 202 Accepted pattern, status polling, webhooks
12. [Webhooks & Event Callbacks](15-webhooks-callbacks.md) - Push notifications, signature verification, retry strategies
13. [API Gateway Patterns](14-api-gateway-patterns.md) - Gateway responsibilities, BFF pattern, API composition, circuit breaking

### Operations & DevEx
14. [Documentation & OpenAPI](10-documentation-openapi.md) - Contract-first development, OpenAPI specs, developer experience
15. [Testing, Contracts, and Automation](11-testing-contracts-and-automation.md) - Testing pyramid, contract testing, performance testing
16. [Observability & Monitoring](13-observability-monitoring.md) - Metrics, logs, traces, SLOs, distributed tracing
17. [Deprecation & API Lifecycle](17-deprecation-lifecycle.md) - Lifecycle stages, sunset policies, migration strategies
18. [Client SDKs & Integration](18-client-sdks-integration.md) - SDK design, code generation, retry logic, error handling

## Quick Reference

### Common Status Codes
- `200 OK`, `201 Created`, `204 No Content`
- `400 Bad Request`, `401 Unauthorized`, `403 Forbidden`, `404 Not Found`, `409 Conflict`, `422 Unprocessable Entity`, `429 Too Many Requests`
- `500 Internal Server Error`, `502 Bad Gateway`, `503 Service Unavailable`

### Key Headers
- Authentication: `Authorization: Bearer <token>`
- Rate Limiting: `RateLimit-Limit`, `RateLimit-Remaining`, `RateLimit-Reset`
- Caching: `Cache-Control`, `ETag`, `If-None-Match`, `If-Match`
- Tracing: `X-Request-Id`, `traceparent`, `tracestate`
- Deprecation: `Deprecation`, `Sunset`

### Design Checklist
- [ ] Consistent resource naming (nouns, plural)
- [ ] Proper HTTP method semantics (GET safe, PUT idempotent)
- [ ] RFC 7807 problem details for errors
- [ ] Stable pagination with cursors
- [ ] Versioning strategy defined
- [ ] Authentication and authorization
- [ ] Rate limiting and quotas
- [ ] Request/response validation
- [ ] Comprehensive documentation (OpenAPI)
- [ ] Observability: logs, metrics, traces
- [ ] Security headers and TLS enforcement

## Learning Path
1. Start with fundamentals (01-02)
2. Understand security deeply (03-04)
3. Master error handling and reliability patterns (05, 08-09)
4. Learn advanced operational patterns (13-14, 16)
5. Focus on developer experience (10, 18)
6. Study lifecycle management (07, 17)

---

*Aligned with topics 36-45 in the main topics index*
