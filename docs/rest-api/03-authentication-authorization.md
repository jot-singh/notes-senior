# Authentication & Authorization

## What You'll Learn
How to secure your API using modern authentication and authorization patterns. You'll understand different auth mechanisms, when to use each, how to manage tokens securely, and how to implement fine-grained access control.

## Why This Matters
Security breaches often stem from auth failures: leaked credentials, privilege escalation, token theft, or authorization bypasses. As a senior developer, you need to design auth systems that are both secure and usable, choosing appropriate mechanisms for different client types and threat models. Understanding OAuth 2.0, JWT, and authorization models is essential for microservices and distributed systems.

## Authentication Mechanisms Deep Dive

### OAuth 2.0: The Industry Standard

**What It Is**: An authorization framework (RFC 6749) that enables third-party applications to obtain limited access to user accounts without exposing passwords.

**When to Use**:
- User-facing applications (web, mobile, SPA)
- Third-party integrations
- Delegated access scenarios

**Grant Types**:

1. **Authorization Code Flow with PKCE** (Proof Key for Code Exchange)
   - **For**: Browser apps, mobile apps, SPAs
   - **Why**: Prevents authorization code interception attacks
   - **Flow**: User redirects to auth server → user logs in → auth code returned → exchange code for token with secret
   - **PKCE Addition**: Code verifier (random string) + code challenge prevent token theft if auth code is intercepted
   
   ```
   Client generates: code_verifier (random 43-128 chars)
   Calculates: code_challenge = SHA256(code_verifier)
   
   1. GET /authorize?response_type=code&code_challenge=...
   2. Auth server returns code
   3. POST /token with code + code_verifier
   4. Server verifies SHA256(verifier) == challenge
   ```

2. **Client Credentials Flow**
   - **For**: Service-to-service (machine-to-machine)
   - **Why**: No user context; service authenticates as itself
   - **Flow**: `POST /token` with client_id + client_secret
   - **Use Case**: Batch job accessing API, microservice calling another service

3. **Refresh Token Flow**
   - **For**: Long-lived access without re-authentication
   - **Why**: Short-lived access tokens (15 min), long-lived refresh tokens (days/months)
   - **Security**: Refresh token rotation on each use; detect token reuse (potential theft)

### JWT (JSON Web Tokens)

**What They Are**: Self-contained tokens with claims (user ID, permissions, expiry) signed by issuer.

**Structure**: `header.payload.signature`
```json
Header: {"alg": "RS256", "typ": "JWT"}
Payload: {
  "sub": "user-12345",
  "iat": 1673448000,
  "exp": 1673451600,
  "iss": "https://auth.example.com",
  "aud": "api.example.com",
  "scope": "read:orders write:orders"
}
Signature: RSASSA-PKCS1-v1_5(base64(header) + "." + base64(payload), privateKey)
```

**Advantages**:
- **Stateless**: No database lookup to validate; verify signature + expiry
- **Distributed**: Multiple services validate same token without coordination
- **Self-contained**: Claims embedded (user ID, roles, tenant)

**Trade-offs**:
- **No Revocation**: Once issued, valid until expiry (unless you maintain blacklist)
- **Size**: Larger than opaque tokens (session IDs)
- **Exposure**: Don't put sensitive data in JWT (it's base64, not encrypted)

**Best Practices**:
- Short expiry (15-60 minutes)
- Use RS256 (asymmetric) for multi-service validation
- Validate `iss`, `aud`, `exp`, `nbf`
- Rotate signing keys; distribute via JWKS endpoint

### Opaque Tokens vs JWT

| Aspect | Opaque Token | JWT |
|--------|--------------|-----|
| Structure | Random string (session ID) | Signed JSON claims |
| Validation | Introspection API call | Signature verification (local) |
| Revocation | Immediate (delete from DB) | Requires blacklist or short TTL |
| Size | Small (~32 chars) | Large (~500-2000 bytes) |
| Use Case | High-security, need revocation | Distributed systems, stateless |

**When to Use Opaque**: Admin dashboards, financial transactions, anything requiring instant revocation.
**When to Use JWT**: Microservices, public APIs, mobile apps, stateless validation.

### API Keys

**What They Are**: Long-lived credentials (often UUID or random string) identifying an application.

**When to Use**:
- Server-to-server integrations
- Public APIs with rate limiting
- Developer testing/sandbox access

**Never For**: User-level authentication without additional context (user sessions, OAuth)

**Best Practices**:
- Scope keys (read-only, write, admin)
- Rotate regularly (every 90 days)
- Allow multiple keys per account (seamless rotation)
- Hash keys in database (treat like passwords)
- Pair with IP whitelisting for server-to-server

**Example**:
```
GET /api/v1/orders
Authorization: Bearer ak_live_a1b2c3d4e5f6...
```

### mTLS (Mutual TLS)

**What It Is**: Both client and server present certificates; cryptographic identity verification.

**When to Use**:
- High-security service-to-service (banking, healthcare)
- Zero-trust network architectures
- Replacing API keys for machine authentication

**How It Works**:
1. Server presents certificate (standard TLS)
2. Client presents certificate (mutual)
3. Both verify against trusted CA
4. Connection succeeds only if both valid

**Benefits**: Strongest auth; impossible to phish; no credentials to leak.
**Challenges**: Certificate lifecycle management; client configuration complexity.

**Modern Pattern**: Bind access tokens to TLS channel (DPoP, mTLS-bound tokens) preventing token theft even if intercepted.

## Token Management & Security

### Token Lifecycle Strategy

**Access Tokens**: Short-lived (15-60 minutes)
- Used for API requests
- Cached by client
- Contains user identity and permissions
- Not revocable (expires naturally)

**Refresh Tokens**: Long-lived (days to months)
- Used to obtain new access tokens
- Stored securely (encrypted at rest)
- Can be revoked
- Should rotate on each use (detect theft)

**Example Flow**:
```
1. User logs in → receives access token (exp: 15min) + refresh token
2. Client uses access token for API calls
3. Access token expires
4. Client calls /token with refresh token → new access + new refresh token
5. Old refresh token invalidated (rotation)
```

### Token Claims & Validation

**Standard Claims**:
```json
{
  "iss": "https://auth.example.com",    // Issuer (who created token)
  "sub": "user-12345",                   // Subject (who token is about)
  "aud": "api.example.com",              // Audience (who should accept)
  "exp": 1673451600,                     // Expiry (Unix timestamp)
  "nbf": 1673448000,                     // Not Before
  "iat": 1673448000,                     // Issued At
  "jti": "tok-abc123"                    // JWT ID (unique identifier)
}
```

**Custom Claims**:
```json
{
  "scope": "read:orders write:orders",
  "tenant_id": "org-789",
  "roles": ["admin", "billing"],
  "permissions": ["order:create", "order:cancel"]
}
```

**Validation Checklist**:
1. Verify signature (check against public key)
2. Check expiry (`exp > now`)
3. Check not-before (`nbf <= now`)
4. Validate issuer (`iss` matches expected)
5. Validate audience (`aud` contains your API)
6. Check scope/permissions for endpoint
7. Handle clock skew (±5 minutes tolerance)

### Key Rotation & JWKS

**Why Rotate**: Limit blast radius if key compromised; compliance requirements.

**JWKS (JSON Web Key Set)**: Public endpoint serving signing keys
```json
{
  "keys": [
    {
      "kid": "key-2024-01",
      "kty": "RSA",
      "alg": "RS256",
      "use": "sig",
      "n": "...",  // Public key modulus
      "e": "AQAB"
    }
  ]
}
```

**Rotation Strategy**:
1. Generate new key; add to JWKS with new `kid`
2. Start signing new tokens with new key
3. Keep old key in JWKS for validation (grace period)
4. After all tokens with old key expire, remove from JWKS

**Client Implementation**:
- Cache JWKS with TTL (1 hour)
- Refresh if `kid` in token not found in cached keys
- Validate signature against matching key

## Authorization Models

### RBAC (Role-Based Access Control)

**What It Is**: Users assigned roles; roles have permissions.

**Structure**:
```
User "alice" → Role "OrderManager"
Role "OrderManager" → Permissions ["order:read", "order:update", "order:cancel"]
```

**When to Use**: 
- Clear job functions (admin, editor, viewer)
- Permissions map to roles naturally
- Simpler mental model

**Example**:
```json
Roles:
{
  "admin": ["*:*"],
  "order_manager": ["order:*", "customer:read"],
  "customer_support": ["order:read", "customer:read", "customer:update"],
  "viewer": ["order:read", "customer:read"]
}
```

**Authorization Check**:
```java
if (user.hasRole("order_manager") || user.hasRole("admin")) {
    // Allow order cancellation
}
```

**Limitations**: 
- Role explosion (order_manager_east_coast, order_manager_vip_customers)
- Can't express context ("can cancel order if amount < $1000")

### ABAC (Attribute-Based Access Control)

**What It Is**: Decisions based on attributes of user, resource, environment, and action.

**Attributes**:
- **Subject**: user.role, user.department, user.clearance_level
- **Resource**: order.status, order.amount, order.region
- **Action**: read, write, delete
- **Environment**: time_of_day, ip_address, request_origin

**Policy Example** (pseudo-code):
```
Allow if:
  user.role = "order_manager" AND
  order.region = user.region AND
  order.status NOT IN ["shipped", "delivered"] AND
  current_time BETWEEN 9am AND 6pm
```

**When to Use**:
- Complex, context-dependent policies
- Multi-tenant with isolation requirements
- Fine-grained control beyond roles

**Implementation Patterns**:
- Policy engines (OPA - Open Policy Agent, AWS Cedar)
- Decentralized evaluation (policies distributed to services)
- Centralized evaluation (auth service makes decisions)

**Trade-off**: More expressive but more complex to debug and audit.

### ReBAC (Relationship-Based Access Control)

**What It Is**: Permissions derived from relationships in a graph (like social networks).

**Example**:
```
Alice → owns → Document123
Bob → member_of → TeamA
TeamA → can_edit → Document123

Therefore: Bob can edit Document123
```

**When to Use**: 
- Social features (share with friends)
- Organizational hierarchies
- Document/folder permissions

**Tools**: Google Zanzibar, SpiceDB, Ory Keto

### Choosing an Authorization Model

| Use Case | Model | Why |
|----------|-------|-----|
| Internal admin tools | RBAC | Simple, clear roles |
| SaaS with tenants | RBAC + tenant isolation | Predictable, auditable |
| Financial transactions | ABAC | Context matters (amount, time, location) |
| Document sharing | ReBAC | Natural relationship model |
| API with scopes | Scope-based (OAuth) | Coarse-grained, standard |

### Consent & Least Privilege

**Principle of Least Privilege**: Grant minimum permissions necessary.

**Consent Recording**:
- What data accessed
- Purpose (analytics, marketing, core functionality)
- Duration
- Revocation mechanism

**Example**:
```json
{
  "user_id": "user-123",
  "app_id": "app-analytics",
  "scopes": ["read:orders", "read:customers"],
  "purpose": "sales_reporting",
  "granted_at": "2026-01-11T10:00:00Z",
  "expires_at": "2026-04-11T10:00:00Z"
}
```

**Why It Matters**: GDPR, CCPA compliance; user trust; audit trails.

## Best Practices
- TLS everywhere; HSTS; secure cookies for browser flows.
- Avoid long-lived JWTs; use opaque tokens with introspection for rapid revocation.
- Deny by default; explicit grants; audit every decision.
- Avoid embedding PII in tokens; treat tokens as secrets.

## Pitfalls
- Clock skew causing premature expiry; allow ±5 minutes.
- JWT bloat; prefer references for big claims.
- Scope creep; enforce deny-all baseline.

## Example Headers
- `Authorization: Bearer <token>`
- `WWW-Authenticate` challenges for 401 with relevant error codes.
