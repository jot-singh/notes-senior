# Deprecation & API Lifecycle Management

## What You'll Learn
How to manage API evolution over time, gracefully deprecate old endpoints, and guide users through migrations without breaking their applications. You'll understand lifecycle stages, communication strategies, and governance processes.

## Why This Matters
APIs have long lifecycles—years, not months. As requirements evolve, you'll need to introduce new features, fix design mistakes, and sunset unmaintained functionality. Poorly managed deprecations erode trust and cause customer churn. Well-executed deprecations maintain relationships, give users time to adapt, and enable your team to innovate without accumulating technical debt. This is as much about communication and empathy as it is about technical implementation.

## API Lifecycle Stages

```
┌─────────┐    ┌──────┐    ┌────┐    ┌───────────┐    ┌──────┐
│  Alpha  │ →  │ Beta │ →  │ GA │ →  │ Deprecated│ →  │ EOL  │
└─────────┘    └──────┘    └────┘    └───────────┘    └──────┘
Experimental   Preview     Stable     Discouraged      Sunset
```

### Alpha
- **Purpose**: Early experimentation; gather feedback
- **Characteristics**:
  - No SLA or uptime guarantees
  - Breaking changes allowed without notice
  - May be removed at any time
  - Limited availability (invite-only or feature flag)
- **Documentation**: Clear "ALPHA" labels; warn of instability
- **Usage**: Internal tools, select partners

### Beta
- **Purpose**: Stabilizing for GA; broader testing
- **Characteristics**:
  - Basic SLA (e.g., 95% uptime)
  - Breaking changes require advance notice (e.g., 30 days)
  - Feature-complete but may have bugs
  - Open to all developers
- **Documentation**: "BETA" label; feedback channels prominent
- **Usage**: Production-ready for non-critical paths

### GA (General Availability)
- **Purpose**: Production-grade; long-term support
- **Characteristics**:
  - Full SLA (e.g., 99.9% uptime)
  - Breaking changes require new major version
  - Backward-compatible enhancements only
  - Comprehensive monitoring and support
- **Documentation**: Full reference docs, examples, SLAs published
- **Usage**: All production workloads

### Deprecated
- **Purpose**: Sunset announced; migration window open
- **Characteristics**:
  - Still functional; SLA maintained
  - No new features; security fixes only
  - Sunset date announced (6-12 months out)
  - Migration guides available
- **Documentation**: Prominent deprecation notices; replacement guidance
- **Usage**: Legacy applications migrating off

### EOL (End of Life)
- **Purpose**: No longer supported; service terminated
- **Characteristics**:
  - Returns `410 Gone` or redirects to new version
  - No SLA; no support
  - Data retention per policy (e.g., 90 days)
- **Documentation**: EOL notices; archive documentation
- **Usage**: None

## Deprecation Headers

### RFC 8594: Deprecation Header

```http
HTTP/1.1 200 OK
Deprecation: true
Sunset: Sat, 31 Dec 2026 23:59:59 GMT
Link: </v2/users>; rel="successor-version"
Warning: 299 - "This endpoint will be removed on 2026-12-31. Migrate to /v2/users"
```

**Headers**:
- `Deprecation: true` (or `@timestamp`): Marks endpoint as deprecated
- `Sunset: <HTTP-date>`: EOL date (RFC 8594)
- `Link: <url>; rel="successor-version"`: Points to replacement
- `Warning: 299 - "<message>"`: Human-readable notice

### Implementation

```java
@GetMapping("/v1/users")
@Deprecated
public ResponseEntity<List<User>> getUsersV1() {
    HttpHeaders headers = new HttpHeaders();
    headers.add("Deprecation", "true");
    headers.add("Sunset", "Sat, 31 Dec 2026 23:59:59 GMT");
    headers.add("Link", "</v2/users>; rel=\"successor-version\"");
    headers.add("Warning", "299 - \"Migrate to /v2/users by 2026-12-31\"");
    
    List<User> users = userService.findAll();
    return new ResponseEntity<>(users, headers, HttpStatus.OK);
}
```

## Communication Strategy

### Timeline (12-Month Sunset)

```
T-12 months: Announce deprecation
    │
    ├─ Blog post, changelog entry
    ├─ Email to all API consumers
    └─ In-app notification banner
    
T-9 months: Add deprecation headers
    │
    ├─ All deprecated endpoints return headers
    └─ Dashboard shows affected endpoints
    
T-6 months: Migration guides & webinars
    │
    ├─ Side-by-side API comparison
    ├─ Code examples for migration
    └─ Live Q&A sessions
    
T-3 months: Final warnings & direct outreach
    │
    ├─ Email campaigns to users still on old version
    ├─ Phone calls to high-volume clients
    └─ Dashboard: red alert banner
    
T-1 month: Countdown notifications
    │
    └─ Daily reminders to affected users
    
T=0: EOL
    │
    ├─ Return 410 Gone
    └─ Or redirect to new version
```

### Communication Channels

**1. In-Product Notifications**:
```json
GET /v1/orders
→ 200 OK
X-API-Notice: "This endpoint is deprecated. Migrate to /v2/orders by 2026-12-31"

{
  "_metadata": {
    "deprecation": {
      "deprecated": true,
      "sunset_date": "2026-12-31T23:59:59Z",
      "replacement": "/v2/orders",
      "migration_guide": "https://docs.example.com/migrate-to-v2"
    }
  },
  "orders": [...]
}
```

**2. Developer Dashboard**:
```
┌─────────────────────────────────────────────────────┐
│ ⚠️  DEPRECATION WARNING                              │
│                                                       │
│ Your application uses deprecated endpoints:          │
│                                                       │
│ • GET /v1/users (47,823 calls/day)                  │
│ • POST /v1/orders (12,401 calls/day)                │
│                                                       │
│ Sunset Date: December 31, 2026                       │
│                                                       │
│ [View Migration Guide]  [Contact Support]            │
└─────────────────────────────────────────────────────┘
```

**3. Email Campaigns**:
- T-12: "We're evolving our API"
- T-6: "How to migrate to v2"
- T-3: "Action required: API sunset in 90 days"
- T-1: "Final reminder: 30 days until EOL"

**4. API Response Warnings**:
```javascript
// Client SDK logs warning
console.warn('[API Deprecation] GET /v1/users is deprecated. ' +
             'Migrate to /v2/users by 2026-12-31. ' +
             'See: https://docs.example.com/migrate-to-v2');
```

## Migration Support

### Side-by-Side Comparison

**v1 (Deprecated)**:
```http
GET /v1/users?page=2&limit=50
Authorization: Bearer token

→ 200 OK
{
  "users": [...],
  "page": 2,
  "limit": 50,
  "total": 1523
}
```

**v2 (Recommended)**:
```http
GET /v2/users?cursor=eyJpZCI6MTAwfQ&limit=50
Authorization: Bearer token

→ 200 OK
{
  "data": [...],
  "pagination": {
    "next_cursor": "eyJpZCI6MTUwfQ",
    "has_more": true
  }
}
```

**Migration Guide**:
```markdown
## Key Changes

1. **Pagination**: Offset → Cursor-based
   - Replace `page` param with `cursor`
   - Use `pagination.next_cursor` for next page

2. **Response Structure**: `users` → `data`
   - Top-level array wrapped in `data` field

3. **Authentication**: No change

## Code Example (JavaScript)

### Before (v1):
```javascript
const response = await fetch('/v1/users?page=2&limit=50');
const { users, page, total } = await response.json();
```

### After (v2):
```javascript
const response = await fetch('/v2/users?limit=50');
const { data: users, pagination } = await response.json();

// To get next page:
if (pagination.has_more) {
  const nextPage = await fetch(`/v2/users?cursor=${pagination.next_cursor}&limit=50`);
}
```
```

### Dual-Running Period

**Pattern**: Both versions active simultaneously

```
┌─────────────┬────────────────┬──────────┐
│   Period    │  v1 Status     │ v2 Status│
├─────────────┼────────────────┼──────────┤
│ T-12 to T-6 │ Deprecated     │ GA       │
│ T-6 to T-3  │ Deprecated     │ GA       │
│ T-3 to T    │ Final warnings │ GA       │
│ T onwards   │ EOL (410 Gone) │ GA       │
└─────────────┴────────────────┴──────────┘
```

**Benefits**:
- No forced migrations
- Gradual rollout
- Rollback if issues arise

### Automated Migration Tools

**Example: CLI migration helper**:
```bash
$ api-migrate check
Scanning codebase for deprecated API usage...

Found 3 deprecated endpoints:
  ✗ GET /v1/users (src/api/users.ts:42)
  ✗ POST /v1/orders (src/api/orders.ts:15)
  ✗ GET /v1/reports (src/api/reports.ts:78)

$ api-migrate fix --dry-run
Proposed changes:
  src/api/users.ts:42
    - fetch('/v1/users?page=2')
    + fetch('/v2/users?cursor=' + cursor)

$ api-migrate fix --apply
✓ Migrated 3 endpoints
⚠️  Manual review required for pagination logic
```

## Analytics & Tracking

### Monitor Deprecated Endpoint Usage

```sql
-- Daily usage of deprecated endpoints
SELECT 
    endpoint,
    DATE(timestamp) AS date,
    COUNT(*) AS requests,
    COUNT(DISTINCT client_id) AS unique_clients
FROM api_logs
WHERE endpoint LIKE '/v1/%'
    AND timestamp > NOW() - INTERVAL '30 days'
GROUP BY endpoint, DATE(timestamp)
ORDER BY date DESC, requests DESC;
```

**Metrics Dashboard**:
```
┌─────────────────────────────────────────────┐
│ Deprecated Endpoint Usage (Last 30 Days)    │
├─────────────────────────────────────────────┤
│ GET /v1/users        │ 1.2M req │ ↓ 15%     │
│ POST /v1/orders      │ 450K req │ ↓ 32%     │
│ GET /v1/reports      │  85K req │ ↓ 8%      │
├─────────────────────────────────────────────┤
│ Active Clients Still on v1: 127             │
│ Days Until Sunset: 89                        │
└─────────────────────────────────────────────┘
```

### Proactive Outreach

```python
# Identify high-volume clients still on v1
high_volume_v1_clients = """
SELECT 
    client_id,
    client_name,
    email,
    COUNT(*) AS v1_requests,
    MAX(timestamp) AS last_v1_request
FROM api_logs
WHERE endpoint LIKE '/v1/%'
    AND timestamp > NOW() - INTERVAL '7 days'
GROUP BY client_id, client_name, email
HAVING COUNT(*) > 10000  -- High volume threshold
ORDER BY v1_requests DESC;
"""

# Send personalized outreach
for client in fetch_clients(high_volume_v1_clients):
    send_email(
        to=client.email,
        subject=f"Action Required: Migrate {client.name} to API v2",
        body=f"""
        Hi {client.name},
        
        Your application makes {client.v1_requests:,} requests/week to our 
        deprecated v1 API. To avoid service disruption, please migrate to v2 
        by {SUNSET_DATE}.
        
        We're here to help:
        - Migration guide: {MIGRATION_GUIDE_URL}
        - Schedule consultation: {CALENDLY_URL}
        
        Questions? Reply to this email.
        """
    )
```

## EOL Implementation

### Option 1: 410 Gone

```java
@GetMapping("/v1/users")
public ResponseEntity<Problem> usersV1EOL() {
    Problem problem = Problem.builder()
        .withType(URI.create("https://docs.example.com/problems/endpoint-eol"))
        .withTitle("Endpoint Reached End of Life")
        .withStatus(Status.GONE)
        .withDetail("This endpoint was sunset on 2026-12-31. Use /v2/users instead.")
        .with("sunset_date", "2026-12-31T23:59:59Z")
        .with("replacement", "/v2/users")
        .with("migration_guide", "https://docs.example.com/migrate-to-v2")
        .build();
    
    return ResponseEntity.status(HttpStatus.GONE).body(problem);
}
```

**Response**:
```http
HTTP/1.1 410 Gone
Content-Type: application/problem+json

{
  "type": "https://docs.example.com/problems/endpoint-eol",
  "title": "Endpoint Reached End of Life",
  "status": 410,
  "detail": "This endpoint was sunset on 2026-12-31. Use /v2/users instead.",
  "sunset_date": "2026-12-31T23:59:59Z",
  "replacement": "/v2/users",
  "migration_guide": "https://docs.example.com/migrate-to-v2"
}
```

### Option 2: Redirect (if semantically equivalent)

```java
@GetMapping("/v1/users")
public ResponseEntity<Void> redirectToV2() {
    return ResponseEntity
        .status(HttpStatus.MOVED_PERMANENTLY)  // 301
        .header("Location", "/v2/users")
        .build();
}
```

**Use Case**: Only if v2 is drop-in replacement (same request/response structure)

### Option 3: Graceful Degradation

```java
@GetMapping("/v1/users")
public ResponseEntity<?> usersV1Degraded() {
    // Return cached/stale data with warning
    List<User> cachedUsers = cache.get("users_v1_snapshot");
    
    HttpHeaders headers = new HttpHeaders();
    headers.add("Warning", "299 - \"This endpoint is EOL. Returning cached data from 2026-12-31.\"");
    headers.add("X-Data-Freshness", "stale");
    
    return new ResponseEntity<>(cachedUsers, headers, HttpStatus.OK);
}
```

## Governance & Policy

### API Review Board

**Composition**:
- Engineering leads
- Product managers
- Developer relations
- Customer success

**Responsibilities**:
- Approve deprecations and new versions
- Assess customer impact
- Set sunset timelines
- Review migration plans

**Process**:
1. **Proposal**: Engineer submits deprecation RFC
2. **Impact Analysis**: Review usage metrics
3. **Communication Plan**: Draft timeline and messaging
4. **Approval**: Board votes (requires consensus)
5. **Execution**: Announce and implement

### Versioning Policy

**Example Policy**:
```
- Maximum 2 concurrent major versions supported
- Minimum 12 months between major version releases
- Minor versions: backward-compatible only
- Deprecation notice: minimum 6 months before EOL
- Security fixes: all supported versions
- Bug fixes: latest version only
```

## Best Practices

1. **Long Notice Periods**: 6-12 months minimum
2. **Clear Communication**: Multiple channels, repeated messages
3. **Migration Guides**: Side-by-side examples, code snippets
4. **Dual-Running**: Support old and new simultaneously
5. **Monitor Usage**: Track adoption, identify laggards
6. **Proactive Outreach**: Contact high-volume users directly
7. **Graceful EOL**: 410 Gone with helpful error messages
8. **Document Everything**: Archive old docs for reference

## Anti-Patterns

❌ **Surprise EOL**: No notice or short notice (< 3 months)
❌ **Silent Breaking Changes**: Behavior changes without version bump
❌ **No Migration Path**: Deprecated without replacement
❌ **Ignoring Feedback**: Not adjusting timeline based on user needs
❌ **Abrupt Shutdown**: No grace period or fallback
❌ **Missing Analytics**: Don't know who's affected
