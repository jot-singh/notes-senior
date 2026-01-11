# Client SDKs & Integration Patterns

## What You'll Learn
How to design client libraries that make your API easy to use, handle authentication automatically, implement retry logic, and provide excellent developer experience. You'll understand code generation vs hand-crafted approaches, error handling, and testing patterns.

## Why This Matters
Raw HTTP clients are tedious: manual JSON parsing, error handling, retries, authentication refreshes. Good SDKs abstract complexity, enforce best practices, and reduce integration time from days to hours. A well-designed SDK is a competitive advantage—it lowers friction for adoption and reduces support burden. As a senior engineer, you'll often design SDKs or evaluate third-party libraries, so understanding principles matters.

## SDK Design Principles

### 1. Idiomatic Code

**Match Language Conventions**:

**Python** (snake_case, context managers):
```python
# Good: Pythonic
with APIClient(api_key=os.getenv('API_KEY')) as client:
    users = client.users.list(page_size=50)
    for user in users:  # Generator, handles pagination
        print(user.email)

# Bad: Not Pythonic
client = APIClient(apiKey=process.env['API_KEY'])  # Wrong naming
users = client.getUsers({'pageSize': 50})  # Not method-style
```

**JavaScript** (camelCase, Promises):
```javascript
// Good: JS conventions
const client = new APIClient({ apiKey: process.env.API_KEY });
const users = await client.users.list({ pageSize: 50 });
users.forEach(user => console.log(user.email));

// Bad: Not idiomatic
const client = new APIClient(process.env.API_KEY);  // No config object
client.get_users({ page_size: 50 }, function(err, users) { });  // Callbacks in 2024
```

**Java** (camelCase, builders):
```java
// Good: Java conventions
APIClient client = APIClient.builder()
    .apiKey(System.getenv("API_KEY"))
    .build();
    
List<User> users = client.users()
    .list()
    .pageSize(50)
    .execute();

// Bad: Not Java-like
APIClient client = new APIClient(System.getenv("API_KEY"));
List<User> users = client.list_users(Map.of("page_size", 50));  // Python style
```

### 2. Strongly Typed

**Leverage Type Systems**:

**TypeScript**:
```typescript
interface User {
  id: string;
  email: string;
  name: string;
  created_at: Date;
}

interface ListUsersOptions {
  pageSize?: number;
  cursor?: string;
  status?: 'active' | 'inactive' | 'suspended';  // Union type
}

class UsersResource {
  async list(options?: ListUsersOptions): Promise<Page<User>> {
    // Implementation
  }
  
  async get(id: string): Promise<User> {
    // Auto-complete for return type
  }
}

// Usage with type safety:
const users: Page<User> = await client.users.list({ 
  pageSize: 50,
  status: 'active'  // IDE suggests valid values
});

// Compile error:
const invalid = await client.users.list({ status: 'unknown' });  // TS error
```

**Java with Generics**:
```java
public class Page<T> {
    private List<T> data;
    private String nextCursor;
    private boolean hasMore;
    
    public Iterator<T> iterator() { 
        return new PageIterator<>(this); 
    }
}

// Usage:
Page<User> users = client.users().list().pageSize(50).execute();
for (User user : users) {  // Type-safe iteration
    System.out.println(user.getEmail());
}
```

### 3. Minimal Dependencies

**Problem**: Heavy dependency trees

**Bad**:
```json
// package.json
{
  "dependencies": {
    "axios": "^1.0.0",
    "lodash": "^4.17.21",
    "moment": "^2.29.4",  // 280KB just for dates!
    "bluebird": "^3.7.2",
    "winston": "^3.8.0"   // Logging framework
  }
}
```

**Good**:
```json
// package.json - lean SDK
{
  "dependencies": {
    // Zero dependencies! Use native fetch, built-in Date
  },
  "devDependencies": {
    // Build tools only
  }
}
```

**Trade-offs**:
- **Minimal**: Fast installs, no conflicts, but more code to maintain
- **Batteries-included**: Convenient, but bloated and version conflicts

**Recommendation**: Peer dependencies for optional features

```json
{
  "peerDependencies": {
    "winston": "^3.0.0"  // Optional: user provides if they want logging
  },
  "peerDependenciesMeta": {
    "winston": { "optional": true }
  }
}
```

### 4. Versioned & Compatibility

**SDK Version ≠ API Version**:

```
SDK v3.2.1 → API v2
SDK v3.3.0 → API v2 (new features)
SDK v4.0.0 → API v3 (breaking changes)
```

**Compatibility Matrix** (in docs):
```markdown
| SDK Version | API Version | Min Language Version |
|-------------|-------------|----------------------|
| 4.x.x       | v3          | Node 18+             |
| 3.x.x       | v2          | Node 14+             |
| 2.x.x       | v1          | Node 12+ (EOL)       |
```

## Generation vs Hand-Crafted

### Code Generation (OpenAPI → SDK)

**Tools**:
- OpenAPI Generator
- Swagger Codegen
- openapi-typescript
- openapi-python-client

**Example**:
```bash
# Generate TypeScript SDK from OpenAPI spec
openapi-generator generate \
  -i openapi.yaml \
  -g typescript-fetch \
  -o ./generated-sdk

# Result: Auto-generated but awkward
import { DefaultApi, User } from './generated-sdk';

const api = new DefaultApi({ basePath: 'https://api.example.com' });
const response = await api.usersGet();  // Generic method names
```

**Pros**:
- Fast: spec → code in seconds
- Consistent: never out of sync
- Complete: covers all endpoints

**Cons**:
- Generic names (`usersGet` vs `users.list`)
- Awkward types (overly nested)
- No convenience methods
- Poor error messages

### Hand-Crafted

```typescript
// Carefully designed API
class APIClient {
  users: UsersResource;
  orders: OrdersResource;
  
  constructor(config: ClientConfig) {
    this.users = new UsersResource(this);
    this.orders = new OrdersResource(this);
  }
}

class UsersResource {
  async list(options?: ListOptions): Promise<Page<User>> {
    // Custom logic: handle pagination, retries, errors
  }
  
  async get(id: string): Promise<User> {
    // Convenience: single user fetch
  }
  
  async create(data: CreateUserRequest): Promise<User> {
    // Typed request/response
  }
}
```

**Pros**:
- Polished API design
- Idiomatic code
- Helper methods (e.g., `iterator()`)
- Better error messages

**Cons**:
- Manual maintenance
- Can drift from API spec
- More development time

### Hybrid Approach (Recommended)

```
1. Generate base from OpenAPI
2. Wrap with hand-crafted layer
3. Extend with convenience methods
```

**Example**:
```typescript
// 1. Generated (don't edit):
// generated/api.ts
export class GeneratedUsersApi { /* auto-generated */ }

// 2. Hand-crafted wrapper:
// src/resources/users.ts
import { GeneratedUsersApi } from '../generated/api';

export class UsersResource {
  private generated: GeneratedUsersApi;
  
  constructor(client: APIClient) {
    this.generated = new GeneratedUsersApi(client.config);
  }
  
  // Convenience method (not in OpenAPI)
  async *listAll(options?: ListOptions): AsyncGenerator<User> {
    let cursor = options?.cursor;
    while (true) {
      const page = await this.generated.listUsers({ ...options, cursor });
      for (const user of page.data) {
        yield user;
      }
      if (!page.has_more) break;
      cursor = page.next_cursor;
    }
  }
  
  // Wrapper adds better error handling
  async get(id: string): Promise<User> {
    try {
      return await this.generated.getUser({ id });
    } catch (error) {
      if (error.status === 404) {
        throw new UserNotFoundError(id);
      }
      throw error;
    }
  }
}
```

## Authentication Handling

### API Key

```typescript
class APIClient {
  constructor(config: { apiKey: string; baseURL?: string }) {
    this.apiKey = config.apiKey;
    this.baseURL = config.baseURL || 'https://api.example.com';
  }
  
  private async request(method: string, path: string, data?: any) {
    const response = await fetch(this.baseURL + path, {
      method,
      headers: {
        'Authorization': `Bearer ${this.apiKey}`,
        'Content-Type': 'application/json'
      },
      body: data ? JSON.stringify(data) : undefined
    });
    
    if (!response.ok) {
      throw await this.parseError(response);
    }
    
    return response.json();
  }
}
```

### OAuth with Token Refresh

```typescript
class APIClient {
  private accessToken: string;
  private refreshToken: string;
  private tokenExpiry: Date;
  
  constructor(config: { clientId: string; clientSecret: string }) {
    this.clientId = config.clientId;
    this.clientSecret = config.clientSecret;
  }
  
  private async ensureValidToken(): Promise<void> {
    if (this.accessToken && new Date() < this.tokenExpiry) {
      return;  // Token still valid
    }
    
    // Refresh token
    const response = await fetch('https://auth.example.com/oauth/token', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        grant_type: 'refresh_token',
        refresh_token: this.refreshToken,
        client_id: this.clientId,
        client_secret: this.clientSecret
      })
    });
    
    const data = await response.json();
    this.accessToken = data.access_token;
    this.refreshToken = data.refresh_token;
    this.tokenExpiry = new Date(Date.now() + data.expires_in * 1000);
  }
  
  private async request(method: string, path: string, data?: any) {
    await this.ensureValidToken();  // Auto-refresh
    
    const response = await fetch(this.baseURL + path, {
      method,
      headers: {
        'Authorization': `Bearer ${this.accessToken}`,
        'Content-Type': 'application/json'
      },
      body: data ? JSON.stringify(data) : undefined
    });
    
    // If 401, token might have been revoked—refresh and retry
    if (response.status === 401) {
      await this.ensureValidToken();
      return this.request(method, path, data);  // Retry once
    }
    
    return response.json();
  }
}
```

## Retry & Resilience

### Exponential Backoff with Jitter

```typescript
class APIClient {
  private async requestWithRetry(
    method: string, 
    path: string, 
    data?: any,
    options?: { maxRetries?: number; idempotencyKey?: string }
  ): Promise<any> {
    const maxRetries = options?.maxRetries || 3;
    let attempt = 0;
    
    while (attempt <= maxRetries) {
      try {
        const headers: Record<string, string> = {
          'Authorization': `Bearer ${this.apiKey}`,
          'Content-Type': 'application/json'
        };
        
        if (options?.idempotencyKey) {
          headers['Idempotency-Key'] = options.idempotencyKey;
        }
        
        const response = await fetch(this.baseURL + path, {
          method,
          headers,
          body: data ? JSON.stringify(data) : undefined
        });
        
        // Success
        if (response.ok) {
          return await response.json();
        }
        
        // Don't retry client errors (except 429)
        if (response.status >= 400 && response.status < 500 && response.status !== 429) {
          throw await this.parseError(response);
        }
        
        // Server error or rate limit—retry
        if (attempt < maxRetries) {
          const delay = this.calculateBackoff(attempt);
          await this.sleep(delay);
          attempt++;
          continue;
        }
        
        // Max retries reached
        throw await this.parseError(response);
        
      } catch (error) {
        if (attempt >= maxRetries || !this.isRetryable(error)) {
          throw error;
        }
        
        const delay = this.calculateBackoff(attempt);
        await this.sleep(delay);
        attempt++;
      }
    }
  }
  
  private calculateBackoff(attempt: number): number {
    // Exponential: 1s, 2s, 4s, 8s
    const baseDelay = 1000 * Math.pow(2, attempt);
    // Jitter: ±25%
    const jitter = baseDelay * 0.25 * (Math.random() - 0.5);
    return baseDelay + jitter;
  }
  
  private isRetryable(error: any): boolean {
    // Network errors, timeouts, 5xx
    return error.name === 'NetworkError' || 
           error.name === 'TimeoutError' ||
           (error.status >= 500 && error.status < 600);
  }
  
  private sleep(ms: number): Promise<void> {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
}
```

## Pagination Helpers

### Iterator Pattern

**Python**:
```python
class UsersResource:
    def list(self, page_size=50):
        \"\"\"Returns generator that handles pagination automatically.\"\"\"
        cursor = None
        while True:
            response = self._client.request('GET', '/users', params={
                'page_size': page_size,
                'cursor': cursor
            })
            
            for user in response['data']:
                yield User(user)
            
            if not response['pagination']['has_more']:
                break
            cursor = response['pagination']['next_cursor']

# Usage:
for user in client.users.list(page_size=100):
    print(user.email)  # Fetches pages automatically
```

**JavaScript Async Iterator**:
```typescript
class UsersResource {
  async *list(options?: ListOptions): AsyncGenerator<User> {
    let cursor = options?.cursor;
    
    while (true) {
      const page = await this.client.request('GET', '/users', {
        ...options,
        cursor
      });
      
      for (const user of page.data) {
        yield user;
      }
      
      if (!page.pagination.has_more) break;
      cursor = page.pagination.next_cursor;
    }
  }
}

// Usage:
for await (const user of client.users.list({ pageSize: 100 })) {
  console.log(user.email);  // Automatic pagination
}
```

## Error Handling

### Parse Problem Details (RFC 7807)

```typescript
class APIError extends Error {
  status: number;
  type: string;
  title: string;
  detail: string;
  instance?: string;
  requestId?: string;
  
  constructor(response: Response, problem: ProblemDetails) {
    super(problem.detail || problem.title);
    this.name = 'APIError';
    this.status = response.status;
    this.type = problem.type;
    this.title = problem.title;
    this.detail = problem.detail;
    this.instance = problem.instance;
    this.requestId = response.headers.get('X-Request-ID') || undefined;
  }
}

class UserNotFoundError extends APIError {
  constructor(userId: string, requestId?: string) {
    super(
      new Response(null, { status: 404 }),
      {
        type: 'https://api.example.com/problems/user-not-found',
        title: 'User Not Found',
        detail: `User with ID ${userId} does not exist`,
        status: 404
      }
    );
    this.name = 'UserNotFoundError';
  }
}

// Usage:
try {
  const user = await client.users.get('usr_invalid');
} catch (error) {
  if (error instanceof UserNotFoundError) {
    console.error('User not found:', error.detail);
    console.error('Request ID:', error.requestId);  // For support
  } else if (error instanceof APIError) {
    console.error('API error:', error.status, error.title);
  }
}
```

## Testing & Mocking

### Mock Client for Unit Tests

```typescript
class MockAPIClient implements APIClient {
  users: MockUsersResource;
  
  constructor(fixtures?: Record<string, any>) {
    this.users = new MockUsersResource(fixtures?.users || []);
  }
}

class MockUsersResource implements UsersResource {
  private data: User[];
  
  constructor(data: User[]) {
    this.data = data;
  }
  
  async list(options?: ListOptions): Promise<Page<User>> {
    return {
      data: this.data.slice(0, options?.pageSize || 50),
      pagination: {
        next_cursor: null,
        has_more: false
      }
    };
  }
  
  async get(id: string): Promise<User> {
    const user = this.data.find(u => u.id === id);
    if (!user) throw new UserNotFoundError(id);
    return user;
  }
}

// Test usage:
const mockClient = new MockAPIClient({
  users: [
    { id: 'usr_1', email: 'test@example.com', name: 'Test User' }
  ]
});

const user = await mockClient.users.get('usr_1');
expect(user.email).toBe('test@example.com');
```

### VCR-Style Fixtures (Record/Replay)

```typescript
// Record HTTP interactions
import { setupRecorder } from 'nock-record';

test('users.list()', async () => {
  const { completeRecording, assertScopesFinished } = await setupRecorder({
    mode: 'record',  // or 'replay'
    outputObjects: true
  });
  
  const client = new APIClient({ apiKey: process.env.TEST_API_KEY });
  const users = await client.users.list({ pageSize: 10 });
  
  expect(users.data).toHaveLength(10);
  
  completeRecording();
  assertScopesFinished();
});
```

## Distribution & Release

### Package Managers

**npm** (JavaScript):
```bash
# Publish
npm publish

# Install
npm install @example/api-sdk
```

**PyPI** (Python):
```bash
# Build
python -m build

# Publish
python -m twine upload dist/*

# Install
pip install example-api-sdk
```

**Maven Central** (Java):
```xml
<dependency>
    <groupId>com.example</groupId>
    <artifactId>api-sdk</artifactId>
    <version>3.2.1</version>
</dependency>
```

### Semantic Versioning

```
MAJOR.MINOR.PATCH

3.2.1
│ │ │
│ │ └─ Bug fixes (backward-compatible)
│ └─── New features (backward-compatible)
└───── Breaking changes

Examples:
- 3.2.1 → 3.2.2: Fixed retry bug
- 3.2.2 → 3.3.0: Added webhooks support
- 3.3.0 → 4.0.0: Removed deprecated methods
```

### Automated Releases

```yaml
# .github/workflows/release.yml
name: Release
on:
  push:
    tags:
      - 'v*'

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: 18
          registry-url: 'https://registry.npmjs.org'
      
      - run: npm ci
      - run: npm test
      - run: npm run build
      - run: npm publish
        env:
          NODE_AUTH_TOKEN: ${{secrets.NPM_TOKEN}}
      
      - name: Generate Changelog
        run: |
          npx conventional-changelog-cli -p angular -i CHANGELOG.md -s
          
      - name: Create GitHub Release
        uses: actions/create-release@v1
        with:
          tag_name: ${{ github.ref }}
          release_name: Release ${{ github.ref }}
          body_path: CHANGELOG.md
```

## Best Practices

1. **Idiomatic**: Match language conventions
2. **Typed**: Leverage type systems for safety
3. **Lean**: Minimal dependencies
4. **Retries**: Exponential backoff + jitter
5. **Pagination**: Transparent iterators
6. **Errors**: Typed exceptions, include request IDs
7. **Auth**: Auto-refresh tokens
8. **Testing**: Provide mocks and fixtures
9. **Docs**: Quickstart, API reference, troubleshooting
10. **Versioning**: Semver + clear changelog

## Anti-Patterns

❌ **Stale SDK**: Lags API updates by months
❌ **Generic Errors**: `throw new Error('Request failed')` with no context
❌ **Manual Pagination**: User handles cursors themselves
❌ **No Retries**: Fails on transient errors
❌ **Breaking Changes in Minor**: Violates semver
❌ **Heavy Dependencies**: 100MB SDK for simple API
