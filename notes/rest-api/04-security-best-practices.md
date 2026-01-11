# Security Best Practices

## What You'll Learn
Comprehensive API security covering transport layer, authentication, input validation, data protection, and threat mitigation. You'll understand OWASP API Security Top 10, defense-in-depth strategies, and incident response.

## Why This Matters
APIs are prime attack targets: they expose business logic, access sensitive data, and are often less protected than web UIs. A single vulnerability can leak millions of records, enable financial fraud, or compromise entire systems. API breaches make headlines: Equifax (145M records), Facebook (533M users), T-Mobile (77M customers). As a senior developer, security is your responsibility—not an afterthought. Defense-in-depth and secure-by-default design prevent disasters.

## Transport & Network Security

### HTTPS Everywhere

**Why**: HTTP traffic is plaintext—passwords, tokens, data visible to anyone on network.

**Implementation**:
- Redirect all HTTP → HTTPS (301 or 308)
- Use modern TLS (1.2 minimum, prefer 1.3)
- Disable insecure ciphers (RC4, 3DES, MD5-based)
- Use strong key exchange (ECDHE)

**HSTS (HTTP Strict Transport Security)**:
```http
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
```
- Forces browsers to use HTTPS for 1 year
- Prevents downgrade attacks
- `preload`: Add to browser HSTS preload list

### Security Headers

**X-Content-Type-Options**:
```http
X-Content-Type-Options: nosniff
```
Prevents MIME sniffing attacks; browsers trust declared Content-Type.

**X-Frame-Options**:
```http
X-Frame-Options: DENY
```
Prevents clickjacking; API endpoints shouldn't be framed.

**Content-Security-Policy**:
```http
Content-Security-Policy: default-src 'none'; frame-ancestors 'none'
```
For API: deny all content loading.

**Referrer-Policy**:
```http
Referrer-Policy: no-referrer
```
Don't leak URLs in Referer header (may contain tokens in query params).

### CORS (Cross-Origin Resource Sharing)

**Problem**: Without CORS, any website can call your API from JavaScript.

**Bad (allows everything)**:
```http
Access-Control-Allow-Origin: *
Access-Control-Allow-Credentials: true
// DANGEROUS: any site can make authenticated requests
```

**Good (whitelist origins)**:
```http
Access-Control-Allow-Origin: https://app.example.com
Access-Control-Allow-Methods: GET, POST, PUT, DELETE
Access-Control-Allow-Headers: Authorization, Content-Type
Access-Control-Max-Age: 86400
```

**Never**: `*` with credentials—browsers block this anyway.

## Input Validation & Sanitization

### Validate Everything

**Defense Layers**:
1. **Syntax validation**: JSON parseable, required fields present
2. **Type validation**: String/number/boolean match schema
3. **Format validation**: Email, URL, UUID format
4. **Range validation**: Min/max length, numeric bounds
5. **Business validation**: Valid state transitions, business rules

**Example**:
```java
public class CreateOrderRequest {
    @NotNull
    @Size(min = 1, max = 100)
    private String customerId;
    
    @NotNull
    @DecimalMin("0.01")
    @DecimalMax("100000.00")
    @Digits(integer = 8, fraction = 2)
    private BigDecimal amount;
    
    @NotNull
    @Pattern(regexp = "^[A-Z]{3}$")
    private String currency;  // ISO 4217
    
    @Valid
    @Size(min = 1, max = 100)
    private List<OrderItem> items;
}
```

### Allow-Lists Over Block-Lists

**Bad**: Block known bad patterns
```java
if (input.contains("<script>") || input.contains("DROP TABLE")) {
    throw new ValidationException();
}
// Attacker: <ScRiPt>, dr0p table, etc.
```

**Good**: Allow only known good patterns
```java
if (!input.matches("^[a-zA-Z0-9_-]{1,50}$")) {
    throw new ValidationException();
}
```

### Prevent Resource Exhaustion

**JSON Parsing Limits**:
```java
// Max request size
@Bean
public CommonsMultipartResolver multipartResolver() {
    resolver.setMaxUploadSize(10_000_000);  // 10MB
}

// JSON depth limit
ObjectMapper mapper = new ObjectMapper();
mapper.setStreamReadConstraints(
    StreamReadConstraints.builder()
        .maxNestingDepth(20)  // Prevent deeply nested attacks
        .maxStringLength(10_000_000)
        .build()
);
```

**Rate Limiting**: Covered in module 12

### SQL Injection Prevention

**Vulnerable**:
```java
String query = "SELECT * FROM users WHERE email = '" + userInput + "'";
// Input: ' OR '1'='1'; DROP TABLE users; --
```

**Safe (parameterized queries)**:
```java
String query = "SELECT * FROM users WHERE email = ?";
JdbcTemplate.queryForObject(query, User.class, userInput);
```

**Safe (ORM)**:
```java
@Query("SELECT u FROM User u WHERE u.email = :email")
User findByEmail(@Param("email") String email);
```

### NoSQL Injection

**Vulnerable (MongoDB)**:
```javascript
db.users.find({ email: req.body.email });
// Input: {"$gt": ""} returns all users
```

**Safe**:
```javascript
db.users.find({ email: String(req.body.email) });
// Cast to string
```
