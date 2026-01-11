# Spring Security

## Concept Overview

Spring Security provides comprehensive authentication and authorization for Java applications. It protects against common vulnerabilities (CSRF, session fixation, clickjacking), integrates with various authentication providers, and offers flexible authorization mechanisms from URL-based to method-level security.

## Core Mechanics & Implementation

### Security Configuration

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
            .csrf(csrf -> csrf.csrfTokenRepository(
                CookieCsrfTokenRepository.withHttpOnlyFalse()))
            .sessionManagement(session -> session
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/public/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .requestMatchers("/api/users/**").hasAnyRole("USER", "ADMIN")
                .anyRequest().authenticated())
            .oauth2ResourceServer(oauth2 -> oauth2.jwt(Customizer.withDefaults()))
            .build();
    }
    
    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

### Authentication

```java
// Custom UserDetailsService
@Service
public class CustomUserDetailsService implements UserDetailsService {
    private final UserRepository userRepository;
    
    @Override
    public UserDetails loadUserByUsername(String username) {
        User user = userRepository.findByUsername(username)
            .orElseThrow(() -> new UsernameNotFoundException(username));
        
        return org.springframework.security.core.userdetails.User
            .withUsername(user.getUsername())
            .password(user.getPassword())
            .authorities(user.getRoles().stream()
                .map(role -> new SimpleGrantedAuthority("ROLE_" + role))
                .collect(Collectors.toList()))
            .accountLocked(!user.isActive())
            .build();
    }
}

// Authentication manager
@Bean
public AuthenticationManager authenticationManager(
        AuthenticationConfiguration config) throws Exception {
    return config.getAuthenticationManager();
}
```

### JWT Authentication

```java
@Component
public class JwtTokenProvider {
    @Value("${jwt.secret}")
    private String secretKey;
    
    @Value("${jwt.expiration}")
    private long validityInMs;
    
    public String createToken(String username, List<String> roles) {
        Claims claims = Jwts.claims().setSubject(username);
        claims.put("roles", roles);
        
        Date now = new Date();
        Date validity = new Date(now.getTime() + validityInMs);
        
        return Jwts.builder()
            .setClaims(claims)
            .setIssuedAt(now)
            .setExpiration(validity)
            .signWith(SignatureAlgorithm.HS256, secretKey)
            .compact();
    }
    
    public boolean validateToken(String token) {
        try {
            Jwts.parser().setSigningKey(secretKey).parseClaimsJws(token);
            return true;
        } catch (JwtException | IllegalArgumentException e) {
            return false;
        }
    }
    
    public String getUsername(String token) {
        return Jwts.parser().setSigningKey(secretKey)
            .parseClaimsJws(token).getBody().getSubject();
    }
}

// JWT Filter
@Component
public class JwtAuthenticationFilter extends OncePerRequestFilter {
    private final JwtTokenProvider tokenProvider;
    private final UserDetailsService userDetailsService;
    
    @Override
    protected void doFilterInternal(HttpServletRequest request,
            HttpServletResponse response, FilterChain chain) 
            throws ServletException, IOException {
        
        String token = extractToken(request);
        
        if (token != null && tokenProvider.validateToken(token)) {
            String username = tokenProvider.getUsername(token);
            UserDetails userDetails = userDetailsService.loadUserByUsername(username);
            
            UsernamePasswordAuthenticationToken auth = 
                new UsernamePasswordAuthenticationToken(
                    userDetails, null, userDetails.getAuthorities());
            auth.setDetails(new WebAuthenticationDetailsSource()
                .buildDetails(request));
            
            SecurityContextHolder.getContext().setAuthentication(auth);
        }
        
        chain.doFilter(request, response);
    }
    
    private String extractToken(HttpServletRequest request) {
        String header = request.getHeader("Authorization");
        if (header != null && header.startsWith("Bearer ")) {
            return header.substring(7);
        }
        return null;
    }
}
```

### Method-Level Security

```java
@Configuration
@EnableMethodSecurity
public class MethodSecurityConfig {
}

@Service
public class OrderService {
    
    @PreAuthorize("hasRole('ADMIN')")
    public void deleteOrder(Long id) { }
    
    @PreAuthorize("#username == authentication.principal.username")
    public Order getOrder(String username, Long orderId) { }
    
    @PreAuthorize("hasPermission(#order, 'write')")
    public void updateOrder(Order order) { }
    
    @PostAuthorize("returnObject.owner == authentication.principal.username")
    public Order getOrderSecure(Long id) { }
    
    @PostFilter("filterObject.owner == authentication.principal.username")
    public List<Order> getAllOrders() { }
}
```

### OAuth2 / OpenID Connect

```yaml
# application.yml
spring:
  security:
    oauth2:
      client:
        registration:
          google:
            client-id: ${GOOGLE_CLIENT_ID}
            client-secret: ${GOOGLE_CLIENT_SECRET}
            scope: openid, profile, email
        provider:
          google:
            authorization-uri: https://accounts.google.com/o/oauth2/auth
            token-uri: https://oauth2.googleapis.com/token
```

```java
@Configuration
public class OAuth2Config {
    
    @Bean
    public SecurityFilterChain oauth2SecurityChain(HttpSecurity http) throws Exception {
        return http
            .oauth2Login(oauth2 -> oauth2
                .userInfoEndpoint(userInfo -> userInfo
                    .userService(customOAuth2UserService()))
                .successHandler(oauth2SuccessHandler()))
            .build();
    }
    
    @Bean
    public OAuth2UserService<OAuth2UserRequest, OAuth2User> customOAuth2UserService() {
        return new CustomOAuth2UserService();
    }
}
```

### CORS Configuration

```java
@Bean
public CorsConfigurationSource corsConfigurationSource() {
    CorsConfiguration config = new CorsConfiguration();
    config.setAllowedOrigins(List.of("https://example.com"));
    config.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE"));
    config.setAllowedHeaders(List.of("Authorization", "Content-Type"));
    config.setExposedHeaders(List.of("X-Custom-Header"));
    config.setAllowCredentials(true);
    config.setMaxAge(3600L);
    
    UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
    source.registerCorsConfiguration("/api/**", config);
    return source;
}
```

## Best Practices

- Use BCrypt for password hashing (work factor 10-12)
- Always use HTTPS in production
- Implement proper session management
- Enable CSRF protection for browser clients
- Use short-lived JWT tokens with refresh tokens
- Validate and sanitize all inputs
- Log security events (failed logins, access denied)
- Use method-level security for fine-grained control

## Interview Questions

**Q1: Explain the Spring Security filter chain.**

Series of filters processing each request. Key filters: SecurityContextPersistenceFilter (loads context), UsernamePasswordAuthenticationFilter (form login), BasicAuthenticationFilter, ExceptionTranslationFilter, FilterSecurityInterceptor (authorization). Custom filters added at specific positions.

**Q2: How does JWT authentication work?**

User authenticates, server issues signed JWT containing claims (user, roles, expiry). Client sends JWT in Authorization header. Server validates signature, extracts claims, sets SecurityContext. Stateless—no server session. Use short expiry + refresh tokens.

**Q3: What is the difference between authentication and authorization?**

Authentication verifies identity (who are you?). Authorization determines access (what can you do?). Spring Security: AuthenticationManager handles authentication, AccessDecisionManager handles authorization. Authentication must succeed before authorization.

**Q4: How to secure REST APIs?**

Stateless session, JWT or OAuth2 tokens, CORS configuration, disable CSRF (token-based), HTTPS only, rate limiting, input validation, proper error handling (don't leak info), method-level security for fine-grained control.

**Q5: Explain @PreAuthorize and @PostAuthorize.**

@PreAuthorize: checks before method execution, can use SpEL with method parameters. @PostAuthorize: checks after execution, can access returnObject. Enable with @EnableMethodSecurity. Use for fine-grained business logic authorization.

**Q6: How does OAuth2 work with Spring Security?**

OAuth2 client registration configures providers (Google, GitHub). OAuth2Login handles redirect flow. Resource server validates access tokens. Supports authorization code, client credentials, password grants. UserService customizes user creation from OAuth2 data.

**Q7: What is CSRF and how does Spring Security protect against it?**

Cross-Site Request Forgery: attacker tricks user into submitting requests. Spring generates token, includes in forms/headers, validates on server. CookieCsrfTokenRepository for SPAs. Disable for stateless APIs using token authentication.

**Q8: How to implement remember-me functionality?**

Two approaches: simple hash-based (token in cookie, hashed with expiry) or persistent (token stored in database, more secure). Configure with rememberMe() in HttpSecurity. Set token validity, cookie name, user details service.
