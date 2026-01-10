# Spring Framework

## Concept Overview

Spring Framework is a comprehensive Java platform providing infrastructure support for enterprise applications. Spring Core offers dependency injection (DI) and inversion of control (IoC), while Spring Boot simplifies configuration through convention over configuration. Spring has become the de facto standard for Java enterprise development.

## Core Mechanics & Implementation

### Dependency Injection

```java
// Constructor injection (recommended)
@Service
public class OrderService {
    private final OrderRepository repository;
    private final PaymentService paymentService;
    
    @Autowired  // Optional in Spring 4.3+ with single constructor
    public OrderService(OrderRepository repository, PaymentService paymentService) {
        this.repository = repository;
        this.paymentService = paymentService;
    }
    
    public Order createOrder(OrderRequest request) {
        Order order = new Order(request);
        paymentService.process(order);
        return repository.save(order);
    }
}

// Field injection (avoid - harder to test)
@Service
public class UserService {
    @Autowired
    private UserRepository repository;  // Not recommended
}
```

### Bean Configuration

```java
// Component scanning
@Configuration
@ComponentScan("com.example")
public class AppConfig {
    
    @Bean
    public DataSource dataSource() {
        HikariDataSource ds = new HikariDataSource();
        ds.setJdbcUrl("jdbc:postgresql://localhost/mydb");
        ds.setUsername("user");
        ds.setPassword("password");
        return ds;
    }
    
    @Bean
    @Profile("production")
    public CacheManager cacheManager() {
        return new RedisCacheManager();
    }
}

// Stereotype annotations
@Component     // Generic component
@Service       // Service layer
@Repository    // Data access layer (adds exception translation)
@Controller    // Web layer
@RestController // @Controller + @ResponseBody
```

### Bean Scopes

```java
@Component
@Scope("singleton")  // Default - single instance per container
public class SingletonBean { }

@Component
@Scope("prototype")  // New instance each time requested
public class PrototypeBean { }

@Component
@Scope("request")    // One instance per HTTP request
public class RequestBean { }

@Component
@Scope("session")    // One instance per HTTP session
public class SessionBean { }
```

### Spring Boot Application

```java
@SpringBootApplication  // @Configuration + @EnableAutoConfiguration + @ComponentScan
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}

// REST Controller
@RestController
@RequestMapping("/api/users")
public class UserController {
    private final UserService userService;
    
    public UserController(UserService userService) {
        this.userService = userService;
    }
    
    @GetMapping("/{id}")
    public ResponseEntity<User> getUser(@PathVariable Long id) {
        return userService.findById(id)
            .map(ResponseEntity::ok)
            .orElse(ResponseEntity.notFound().build());
    }
    
    @PostMapping
    public ResponseEntity<User> createUser(@Valid @RequestBody CreateUserRequest request) {
        User user = userService.create(request);
        URI location = ServletUriComponentsBuilder.fromCurrentRequest()
            .path("/{id}").buildAndExpand(user.getId()).toUri();
        return ResponseEntity.created(location).body(user);
    }
    
    @ExceptionHandler(UserNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(UserNotFoundException ex) {
        return ResponseEntity.status(HttpStatus.NOT_FOUND)
            .body(new ErrorResponse(ex.getMessage()));
    }
}
```

### Configuration Properties

```yaml
# application.yml
app:
  name: MyApplication
  api:
    base-url: https://api.example.com
    timeout: 30s
    retry:
      max-attempts: 3
      delay: 1000
```

```java
@ConfigurationProperties(prefix = "app.api")
@Validated
public class ApiProperties {
    @NotBlank
    private String baseUrl;
    private Duration timeout = Duration.ofSeconds(30);
    private RetryProperties retry = new RetryProperties();
    
    // Getters and setters
    
    public static class RetryProperties {
        private int maxAttempts = 3;
        private long delay = 1000;
        // Getters and setters
    }
}
```

### Aspect-Oriented Programming (AOP)

```java
@Aspect
@Component
public class LoggingAspect {
    private static final Logger log = LoggerFactory.getLogger(LoggingAspect.class);
    
    @Around("@annotation(Loggable)")
    public Object logMethod(ProceedingJoinPoint joinPoint) throws Throwable {
        String methodName = joinPoint.getSignature().getName();
        log.info("Entering: {}", methodName);
        
        long start = System.currentTimeMillis();
        try {
            Object result = joinPoint.proceed();
            log.info("Exiting: {} ({}ms)", methodName, System.currentTimeMillis() - start);
            return result;
        } catch (Exception e) {
            log.error("Exception in {}: {}", methodName, e.getMessage());
            throw e;
        }
    }
    
    @Before("execution(* com.example.service.*.*(..))")
    public void beforeServiceMethods(JoinPoint joinPoint) {
        log.debug("Calling: {}", joinPoint.getSignature());
    }
}

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface Loggable { }
```

### Transaction Management

```java
@Service
@Transactional(readOnly = true)  // Default for class
public class OrderService {
    
    @Transactional  // Read-write for this method
    public Order createOrder(OrderRequest request) {
        Order order = orderRepository.save(new Order(request));
        inventoryService.reserve(order.getItems());  // Same transaction
        return order;
    }
    
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void auditLog(String action) {
        // Runs in new transaction, commits independently
        auditRepository.save(new AuditLog(action));
    }
    
    @Transactional(rollbackFor = BusinessException.class)
    public void processPayment(Order order) throws BusinessException {
        // Rolls back on BusinessException (checked exception)
    }
}
```

## Best Practices

- **Constructor injection**: Prefer over field injection for immutability and testability
- **Profiles**: Use for environment-specific configuration
- **Externalize configuration**: Use `application.yml` and `@ConfigurationProperties`
- **Thin controllers**: Keep business logic in services
- **Global exception handling**: Use `@ControllerAdvice`
- **Actuator**: Enable for production monitoring endpoints

## Interview Questions

**Q1: Explain Spring IoC container and its benefits.**

IoC container manages object lifecycle and dependencies. Benefits: loose coupling (depend on interfaces), easier testing (inject mocks), centralized configuration, lifecycle management (init/destroy callbacks). ApplicationContext is the main container, BeanFactory is lightweight alternative.

**Q2: What is the difference between @Component and @Bean?**

`@Component` is class-level annotation for auto-detection via component scanning. `@Bean` is method-level annotation in `@Configuration` class, provides explicit bean definition. Use `@Bean` when you need control over third-party class instantiation or complex creation logic.

**Q3: Explain bean scopes.**

Singleton (default): one instance per container. Prototype: new instance each injection. Request: one per HTTP request. Session: one per HTTP session. Application: one per ServletContext. Custom scopes possible via Scope interface.

**Q4: How does Spring AOP work?**

Spring creates proxies (JDK dynamic or CGLIB) around target beans. Aspects define cross-cutting concerns with pointcuts (where) and advice (what). Advice types: Before, After, Around, AfterReturning, AfterThrowing. Limited to method execution (not field access like AspectJ).

**Q5: What is @Transactional and how does it work?**

Marks methods/classes for transaction management. Spring creates proxy intercepting calls, begins transaction before method, commits on success, rolls back on RuntimeException. Propagation controls transaction boundaries (REQUIRED, REQUIRES_NEW, etc.). Only works on public methods via proxy.

**Q6: Explain Spring Boot auto-configuration.**

`@EnableAutoConfiguration` triggers conditional bean creation based on classpath and properties. Auto-configuration classes use `@ConditionalOnClass`, `@ConditionalOnMissingBean`, etc. Starter dependencies include auto-configurations. Override by defining your own beans.

**Q7: How to handle exceptions globally in Spring?**

Use `@ControllerAdvice` with `@ExceptionHandler` methods. Define handlers for specific exceptions returning appropriate responses. `ResponseEntityExceptionHandler` provides common exception handling. Can also use `@ResponseStatus` on custom exceptions.

**Q8: What is constructor injection advantage over field injection?**

Constructor injection: dependencies are explicit, immutable (final fields), required at construction (fail-fast), easy to unit test (pass mocks in constructor). Field injection: hides dependencies, allows partial construction, requires reflection in tests. Constructor injection follows better OO design.
