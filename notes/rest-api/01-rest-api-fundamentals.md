# REST API Fundamentals (Senior-Level)

## What You'll Learn
This module covers the foundational principles of REST (Representational State Transfer) as an architectural style, not just a protocol. You'll understand why REST's constraints exist, how they enable web-scale systems, and how to apply HTTP semantics correctly in production APIs.

## Why This Matters
REST isn't just about CRUD over HTTP. Understanding its constraints helps you design APIs that scale horizontally, evolve without breaking clients, leverage existing HTTP infrastructure (caches, proxies, CDNs), and integrate seamlessly with the web ecosystem. Senior developers need to distinguish between REST principles and pragmatic HTTP API design, making informed trade-offs.

## Overview
REST is an architectural style for distributed hypermedia systems, introduced by Roy Fielding in his 2000 dissertation. Unlike SOAP or RPC-style APIs, REST leverages HTTP as an application protocol, not just a transport. Mature REST APIs embrace constraints that enable scalability, evolvability, and interoperability over HTTP.

## REST Constraints

### Why Constraints Matter
These constraints aren't arbitrary rules—they're design decisions that enable specific system properties. Each constraint trades local optimization for global scalability and evolvability.

### 1. Client-Server
**What**: Separation of concerns between UI/presentation (client) and data storage/business logic (server).
**Why**: Clients and servers evolve independently. Mobile apps, web frontends, and CLI tools can all consume the same API. Server teams can refactor backends without coordinating client releases.
**Example**: Your API serves both a React SPA and a native iOS app without knowing their implementation details.

### 2. Stateless
**What**: Each request from client to server must contain all information needed to understand the request. No session state on server.
**Why**: Enables horizontal scaling (any server can handle any request), simplifies reliability (no session replication), and improves observability (each request is self-contained).
**Trade-off**: Larger requests (token in every call vs session cookie), but modern tokens are small and compress well.
**Example**: JWT contains user identity and permissions; server doesn't query session store on every request.

### 3. Cacheable
**What**: Responses must implicitly or explicitly label themselves as cacheable or non-cacheable.
**Why**: Eliminates some client-server interactions, improving performance and reducing server load. Leverages existing HTTP caching infrastructure (browsers, CDNs, proxies).
**Example**: `Cache-Control: max-age=3600` lets CDNs serve product catalog data for an hour without hitting your origin servers.

### 4. Uniform Interface
**What**: Standardized interface between components; resource identification, manipulation through representations, self-descriptive messages, HATEOAS.
**Why**: Simplifies architecture, decouples clients from servers. Developers familiar with HTTP already understand your API's fundamentals.
**Components**:
- Resource identification (URIs)
- Resource manipulation through representations (JSON/XML payloads)
- Self-descriptive messages (HTTP headers convey metadata)
- Hypermedia as engine of application state (HATEOAS)

### 5. Layered System
**What**: Client cannot tell if connected directly to server or intermediary (proxy, gateway, load balancer).
**Why**: Enables infrastructure evolution, security boundaries (DMZ), caching layers, load distribution—all transparent to clients.
**Example**: Add Cloudflare CDN without changing client code; implement API gateway for auth/rate limiting without backend changes.

### 6. Code-on-Demand (Optional)
**What**: Servers can extend client functionality by transferring executable code (JavaScript, applets).
**Why**: Reduces pre-implemented client features, allows feature deployment without client updates.
**Rarely Used**: Most APIs don't leverage this; more common in hypermedia-driven UIs.

## HTTP Semantics (RFC 7231)
- Safe Methods: GET, HEAD, OPTIONS; no state changes.
- Idempotent Methods: PUT, DELETE, GET, HEAD, OPTIONS; repeated calls have same effect.
- Partial Update: PATCH; non-idempotent by default, define idempotency contract if needed.
- Resource Representations: JSON (default), also XML, HAL, JSON:API; leverage content negotiation via `Accept`.

## Status Code Families
- 2xx Success: 200 OK, 201 Created, 204 No Content, 206 Partial Content.
- 3xx Redirection: 301/302/307/308; prefer 307/308 for semantic correctness.
- 4xx Client Errors: 400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found, 409 Conflict, 412 Precondition Failed, 422 Unprocessable Entity, 429 Too Many Requests.
- 5xx Server Errors: 500 Internal Server Error, 502 Bad Gateway, 503 Service Unavailable, 504 Gateway Timeout.

## Resource Modeling
- Nouns over verbs: `/orders`, `/orders/{id}`, `/customers/{id}/orders`.
- Relationships: Use nested routes for containment, links for associations.
- Identifiers: Stable server-generated IDs; avoid exposing internal DB keys; consider ULIDs for sortable IDs.
- Time: ISO 8601 UTC (`2026-01-11T13:45:30Z`).
- Collections: Support pagination, filtering, sorting consistently.

## Content Negotiation
- Requests: `Accept: application/json`; optional versioning via media types: `application/vnd.acme.order+json;v=2`.
- Responses: `Content-Type: application/json`; include `Vary` where representation changes (e.g., `Vary: Accept`).

## Idempotency & Safety
- GET is safe and idempotent.
- PUT is idempotent: full replacement semantics.
- DELETE is idempotent: subsequent deletes still yield success (often 204) or 404.
- POST: Not idempotent by default; define idempotency keys for create/charge operations.

## Hypermedia Considerations
- HATEOAS: Include navigable links in responses to reduce coupling.
- Self-descriptive messages: Embed enough metadata to process without out-of-band info.

## Standardized Error Model
- Prefer RFC 7807 Problem Details: `application/problem+json` with `type`, `title`, `status`, `detail`, `instance`.

Example:
```
HTTP/1.1 422 Unprocessable Entity
Content-Type: application/problem+json

{
  "type": "https://api.example.com/problems/validation-error",
  "title": "Invalid request",
  "status": 422,
  "detail": "Field 'email' must be a valid address",
  "instance": "/orders/123"
}
```

## Design Checklist
- Consistent resource naming and HTTP semantics.
- Clear caching policy with `Cache-Control` and validators (ETag/Last-Modified).
- Robust authentication and authorization (OAuth2/JWT/mTLS).
- Stable pagination and filtering contracts.
- Versioning strategy and deprecation lifecycle.
- Observability: request IDs, trace IDs, logs, metrics.
- Security baseline: TLS everywhere, headers, input validation.
