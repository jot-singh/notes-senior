# Security in Distributed Systems

## What You'll Learn

- Authentication and authorization patterns
- Encryption at rest and in transit
- API security best practices
- DDoS protection and rate limiting
- Security auditing and compliance

## Why This Matters

Security breaches cost companies millions. Equifax breach: $1.4B. Target breach: $290M. Beyond financial loss, breaches destroy customer trust and brand reputation. In distributed systems, attack surface expands dramatically - every service, API, and network connection is a potential vulnerability. Understanding security patterns prevents disasters and maintains customer confidence.

## Authentication Patterns

### JWT (JSON Web Token)

Stateless authentication tokens containing claims about the user.

```java
// JWT generation
public class JwtService {
    private final String secretKey;
    private final long expirationMs = 3600000; // 1 hour
    
    public String generateToken(User user) {
        Map<String, Object> claims = new HashMap<>();
        claims.put("userId", user.getId());
        claims.put("email", user.getEmail());
        claims.put("roles", user.getRoles());
        
        return Jwts.builder()
            .setClaims(claims)
            .setSubject(user.getEmail())
            .setIssuedAt(new Date())
            .setExpiration(new Date(System.currentTimeMillis() + expirationMs))
            .signWith(Keys.hmacShaKeyFor(secretKey.getBytes()), SignatureAlgorithm.HS256)
            .compact();
    }
    
    public Claims validateToken(String token) {
        try {
            return Jwts.parserBuilder()
                .setSigningKey(Keys.hmacShaKeyFor(secretKey.getBytes()))
                .build()
                .parseClaimsJws(token)
                .getBody();
        } catch (ExpiredJwtException e) {
            throw new AuthenticationException("Token expired");
        } catch (JwtException e) {
            throw new AuthenticationException("Invalid token");
        }
    }
}

// Security filter
@Component
public class JwtAuthenticationFilter extends OncePerRequestFilter {
    private final JwtService jwtService;
    
    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                   HttpServletResponse response,
                                   FilterChain filterChain) {
        String authHeader = request.getHeader("Authorization");
        
        if (authHeader != null && authHeader.startsWith("Bearer ")) {
            String token = authHeader.substring(7);
            
            try {
                Claims claims = jwtService.validateToken(token);
                
                UserDetails userDetails = new User(
                    claims.getSubject(),
                    "",
                    getAuthorities(claims.get("roles"))
                );
                
                Authentication auth = new UsernamePasswordAuthenticationToken(
                    userDetails, null, userDetails.getAuthorities()
                );
                
                SecurityContextHolder.getContext().setAuthentication(auth);
                
            } catch (AuthenticationException e) {
                response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
                return;
            }
        }
        
        filterChain.doFilter(request, response);
    }
}
```

### OAuth 2.0

Delegated authorization framework for third-party access.

```python
# OAuth 2.0 Authorization Code Flow
from flask import Flask, redirect, request, session
import requests

app = Flask(__name__)

OAUTH_CONFIG = {
    'client_id': 'your_client_id',
    'client_secret': 'your_client_secret',
    'authorization_url': 'https://provider.com/oauth/authorize',
    'token_url': 'https://provider.com/oauth/token',
    'redirect_uri': 'https://yourapp.com/callback'
}

@app.route('/login')
def login():
    """Step 1: Redirect to OAuth provider"""
    params = {
        'client_id': OAUTH_CONFIG['client_id'],
        'redirect_uri': OAUTH_CONFIG['redirect_uri'],
        'response_type': 'code',
        'scope': 'read write',
        'state': generate_random_state()
    }
    
    session['oauth_state'] = params['state']
    
    auth_url = f"{OAUTH_CONFIG['authorization_url']}?" + urlencode(params)
    return redirect(auth_url)

@app.route('/callback')
def callback():
    """Step 2: Handle OAuth callback"""
    code = request.args.get('code')
    state = request.args.get('state')
    
    # Verify state to prevent CSRF
    if state != session.get('oauth_state'):
        return "Invalid state", 400
    
    # Step 3: Exchange code for access token
    token_response = requests.post(
        OAUTH_CONFIG['token_url'],
        data={
            'grant_type': 'authorization_code',
            'code': code,
            'redirect_uri': OAUTH_CONFIG['redirect_uri'],
            'client_id': OAUTH_CONFIG['client_id'],
            'client_secret': OAUTH_CONFIG['client_secret']
        }
    )
    
    if token_response.status_code == 200:
        tokens = token_response.json()
        session['access_token'] = tokens['access_token']
        session['refresh_token'] = tokens.get('refresh_token')
        
        return redirect('/dashboard')
    else:
        return "Authentication failed", 401

@app.route('/api/protected')
def protected_resource():
    """Access protected resource"""
    access_token = session.get('access_token')
    
    if not access_token:
        return redirect('/login')
    
    # Call API with access token
    response = requests.get(
        'https://api.provider.com/user',
        headers={'Authorization': f'Bearer {access_token}'}
    )
    
    return response.json()
```

### API Keys

Simple authentication for service-to-service communication.

```javascript
// API key generation and validation
const crypto = require('crypto');

class ApiKeyManager {
    constructor(db) {
        this.db = db;
    }
    
    generateApiKey() {
        // Generate cryptographically secure random key
        const apiKey = crypto.randomBytes(32).toString('hex');
        const hashedKey = this.hashApiKey(apiKey);
        
        return { apiKey, hashedKey };
    }
    
    hashApiKey(apiKey) {
        return crypto
            .createHash('sha256')
            .update(apiKey)
            .digest('hex');
    }
    
    async createApiKey(userId, name, permissions) {
        const { apiKey, hashedKey } = this.generateApiKey();
        
        await this.db.insert('api_keys', {
            user_id: userId,
            name: name,
            key_hash: hashedKey,
            permissions: JSON.stringify(permissions),
            created_at: new Date(),
            last_used: null
        });
        
        // Return unhashed key only once
        return apiKey;
    }
    
    async validateApiKey(apiKey) {
        const hashedKey = this.hashApiKey(apiKey);
        
        const keyRecord = await this.db.findOne('api_keys', {
            key_hash: hashedKey,
            revoked: false
        });
        
        if (!keyRecord) {
            return null;
        }
        
        // Update last used timestamp
        await this.db.update('api_keys', 
            { id: keyRecord.id },
            { last_used: new Date() }
        );
        
        return {
            userId: keyRecord.user_id,
            permissions: JSON.parse(keyRecord.permissions)
        };
    }
}

// Express middleware
const apiKeyAuth = async (req, res, next) => {
    const apiKey = req.headers['x-api-key'];
    
    if (!apiKey) {
        return res.status(401).json({ error: 'API key required' });
    }
    
    const keyInfo = await apiKeyManager.validateApiKey(apiKey);
    
    if (!keyInfo) {
        return res.status(401).json({ error: 'Invalid API key' });
    }
    
    req.user = keyInfo;
    next();
};
```

## Authorization Patterns

### Role-Based Access Control (RBAC)

```java
// RBAC implementation
public class RBACAuthorization {
    private final Map<String, Set<String>> rolePermissions = Map.of(
        "admin", Set.of("read", "write", "delete", "manage_users"),
        "editor", Set.of("read", "write"),
        "viewer", Set.of("read")
    );
    
    public boolean hasPermission(User user, String requiredPermission) {
        return user.getRoles().stream()
            .flatMap(role -> rolePermissions.getOrDefault(role, Set.of()).stream())
            .anyMatch(permission -> permission.equals(requiredPermission));
    }
}

// Spring Security annotation
@PreAuthorize("hasPermission(#documentId, 'document', 'write')")
@PutMapping("/documents/{documentId}")
public ResponseEntity<Document> updateDocument(
    @PathVariable Long documentId,
    @RequestBody DocumentUpdate update) {
    // Only users with 'write' permission can access
    return ResponseEntity.ok(documentService.update(documentId, update));
}
```

### Attribute-Based Access Control (ABAC)

More flexible than RBAC, considers multiple attributes.

```python
# ABAC policy engine
class ABACPolicy:
    def __init__(self):
        self.policies = []
    
    def add_policy(self, name, condition_fn):
        self.policies.append({'name': name, 'condition': condition_fn})
    
    def evaluate(self, subject, resource, action, context):
        """Evaluate all policies"""
        for policy in self.policies:
            if not policy['condition'](subject, resource, action, context):
                return False, policy['name']
        return True, None

# Define policies
abac = ABACPolicy()

# Policy 1: Users can only edit their own documents
abac.add_policy(
    "owner_only_edit",
    lambda subject, resource, action, ctx:
        action != "edit" or resource.owner_id == subject.user_id
)

# Policy 2: No edits during maintenance window
abac.add_policy(
    "no_maintenance_edits",
    lambda subject, resource, action, ctx:
        action != "edit" or not ctx.is_maintenance_window()
)

# Policy 3: Sensitive documents require MFA
abac.add_policy(
    "sensitive_requires_mfa",
    lambda subject, resource, action, ctx:
        not resource.is_sensitive or subject.has_mfa
)

# Usage
def check_access(user, document, action):
    context = {
        'is_maintenance_window': is_maintenance_window,
        'ip_address': request.remote_addr,
        'time': datetime.now()
    }
    
    allowed, denied_by = abac.evaluate(user, document, action, context)
    
    if not allowed:
        log_access_denied(user, document, action, denied_by)
        raise Forbidden(f"Access denied by policy: {denied_by}")
    
    log_access_granted(user, document, action)
    return True
```

## Encryption

### Encryption in Transit (TLS/SSL)

```javascript
// Node.js HTTPS server with TLS
const https = require('https');
const fs = require('fs');

const options = {
    key: fs.readFileSync('private-key.pem'),
    cert: fs.readFileSync('certificate.pem'),
    ca: fs.readFileSync('ca-cert.pem'),
    
    // Security settings
    ciphers: 'ECDHE-RSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384',
    honorCipherOrder: true,
    minVersion: 'TLSv1.2',
    maxVersion: 'TLSv1.3',
    
    // Require client certificates for mutual TLS
    requestCert: true,
    rejectUnauthorized: true
};

const server = https.createServer(options, (req, res) => {
    // Verify client certificate
    const cert = req.socket.getPeerCertificate();
    
    if (req.client.authorized) {
        res.writeHead(200);
        res.end('Authorized');
    } else {
        res.writeHead(401);
        res.end('Unauthorized');
    }
});

server.listen(443);
```

### Encryption at Rest

```java
// AES-256 encryption for sensitive data
public class EncryptionService {
    private final SecretKey secretKey;
    private static final String ALGORITHM = "AES/GCM/NoPadding";
    
    public EncryptionService(String base64Key) {
        byte[] decodedKey = Base64.getDecoder().decode(base64Key);
        this.secretKey = new SecretKeySpec(decodedKey, "AES");
    }
    
    public String encrypt(String plaintext) throws Exception {
        Cipher cipher = Cipher.getInstance(ALGORITHM);
        
        // Generate random IV
        byte[] iv = new byte[12];
        SecureRandom random = new SecureRandom();
        random.nextBytes(iv);
        GCMParameterSpec parameterSpec = new GCMParameterSpec(128, iv);
        
        cipher.init(Cipher.ENCRYPT_MODE, secretKey, parameterSpec);
        
        byte[] ciphertext = cipher.doFinal(plaintext.getBytes(StandardCharsets.UTF_8));
        
        // Prepend IV to ciphertext
        byte[] combined = new byte[iv.length + ciphertext.length];
        System.arraycopy(iv, 0, combined, 0, iv.length);
        System.arraycopy(ciphertext, 0, combined, iv.length, ciphertext.length);
        
        return Base64.getEncoder().encodeToString(combined);
    }
    
    public String decrypt(String encryptedData) throws Exception {
        byte[] combined = Base64.getDecoder().decode(encryptedData);
        
        // Extract IV
        byte[] iv = new byte[12];
        System.arraycopy(combined, 0, iv, 0, iv.length);
        
        // Extract ciphertext
        byte[] ciphertext = new byte[combined.length - iv.length];
        System.arraycopy(combined, iv.length, ciphertext, 0, ciphertext.length);
        
        Cipher cipher = Cipher.getInstance(ALGORITHM);
        GCMParameterSpec parameterSpec = new GCMParameterSpec(128, iv);
        cipher.init(Cipher.DECRYPT_MODE, secretKey, parameterSpec);
        
        byte[] plaintext = cipher.doFinal(ciphertext);
        return new String(plaintext, StandardCharsets.UTF_8);
    }
}

// Database field encryption
@Entity
public class User {
    @Id
    private Long id;
    
    private String email;
    
    @Convert(converter = EncryptedStringConverter.class)
    private String socialSecurityNumber;  // Encrypted at rest
}

@Converter
public class EncryptedStringConverter implements AttributeConverter<String, String> {
    private final EncryptionService encryptionService;
    
    @Override
    public String convertToDatabaseColumn(String attribute) {
        try {
            return encryptionService.encrypt(attribute);
        } catch (Exception e) {
            throw new RuntimeException("Encryption failed", e);
        }
    }
    
    @Override
    public String convertToEntityAttribute(String dbData) {
        try {
            return encryptionService.decrypt(dbData);
        } catch (Exception e) {
            throw new RuntimeException("Decryption failed", e);
        }
    }
}
```

## DDoS Protection

```python
# Rate limiting with token bucket
from time import time
from threading import Lock

class TokenBucket:
    def __init__(self, capacity, refill_rate):
        self.capacity = capacity
        self.refill_rate = refill_rate  # tokens per second
        self.tokens = capacity
        self.last_refill = time()
        self.lock = Lock()
    
    def consume(self, tokens=1):
        """Try to consume tokens, return True if successful"""
        with self.lock:
            self._refill()
            
            if self.tokens >= tokens:
                self.tokens -= tokens
                return True
            return False
    
    def _refill(self):
        now = time()
        elapsed = now - self.last_refill
        tokens_to_add = elapsed * self.refill_rate
        
        self.tokens = min(self.capacity, self.tokens + tokens_to_add)
        self.last_refill = now

# Per-IP rate limiting
class RateLimiter:
    def __init__(self):
        self.buckets = {}  # ip_address -> TokenBucket
    
    def is_allowed(self, ip_address, tokens=1):
        """Check if request from IP is allowed"""
        if ip_address not in self.buckets:
            # 100 requests per minute per IP
            self.buckets[ip_address] = TokenBucket(
                capacity=100,
                refill_rate=100/60  # ~1.67 per second
            )
        
        return self.buckets[ip_address].consume(tokens)

# Flask middleware
rate_limiter = RateLimiter()

@app.before_request
def rate_limit_check():
    ip = request.remote_addr
    
    if not rate_limiter.is_allowed(ip):
        return jsonify({'error': 'Rate limit exceeded'}), 429
```

## Security Best Practices

**Principle of Least Privilege**: Grant minimum permissions necessary. Revoke access when no longer needed.

**Defense in Depth**: Multiple layers of security. If one fails, others protect.

**Input Validation**: Never trust user input. Validate, sanitize, and escape all data.

**Secure Defaults**: Systems should be secure by default, not require configuration.

**Audit Logging**: Log all security-relevant events for forensics and compliance.

## Anti-Patterns

❌ **Storing Passwords in Plain Text**: Always hash with bcrypt, scrypt, or Argon2

❌ **Security Through Obscurity**: Hiding implementation details isn't security

❌ **Weak Encryption**: DES, MD5, SHA1 are broken. Use AES-256, SHA-256+

❌ **Hardcoded Secrets**: Credentials in code get leaked. Use secret management

❌ **No Rate Limiting**: APIs without rate limits are DDoS targets

## Real-World Examples

**GitHub**: OAuth for third-party apps. API keys for automation. 2FA for accounts. SOC 2 compliant.

**Stripe**: PCI DSS compliant. TLS 1.2+. API keys with granular permissions. Extensive audit logging.

**AWS**: IAM for fine-grained access control. KMS for key management. CloudTrail for auditing.

**Auth0**: Specialized authentication service. Handles OAuth, SAML, social logins. SOC 2, ISO 27001 certified.

## Related Topics

- [Rate Limiting](07-rate-limiting.md)
- [API Gateway](08-api-gateway.md)
- [Distributed Systems Fundamentals](11-distributed-systems-fundamentals.md)
- [Monitoring & Observability](13-monitoring-observability.md)
