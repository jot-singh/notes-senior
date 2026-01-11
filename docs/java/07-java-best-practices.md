# Java Best Practices

## Concept Overview

Java best practices encompass coding conventions, design principles, and engineering practices that produce maintainable, readable, and robust code. Following established practices ensures code quality, reduces bugs, and facilitates team collaboration.

## Core Practices & Implementation

### Clean Code Principles

```java
// Use meaningful names
// Bad
int d; // elapsed time in days
List<int[]> list1;

// Good
int elapsedTimeInDays;
List<int[]> flaggedCells;

// Keep methods small and focused
// Bad: method does too many things
public void processOrder(Order order) {
    // Validate order (20 lines)
    // Calculate totals (15 lines)
    // Apply discounts (10 lines)
    // Update inventory (15 lines)
    // Send notification (10 lines)
}

// Good: extract into focused methods
public void processOrder(Order order) {
    validateOrder(order);
    OrderTotals totals = calculateTotals(order);
    totals = applyDiscounts(totals, order.getCustomer());
    updateInventory(order);
    sendConfirmation(order);
}

// Avoid magic numbers
// Bad
if (status == 3) { ... }

// Good
private static final int STATUS_COMPLETED = 3;
if (status == STATUS_COMPLETED) { ... }

// Better: use enums
public enum OrderStatus { PENDING, PROCESSING, COMPLETED, CANCELLED }
if (status == OrderStatus.COMPLETED) { ... }
```

### Null Safety

```java
// Return empty collections instead of null
public List<User> findUsers(String query) {
    List<User> results = repository.search(query);
    return results != null ? results : Collections.emptyList();
}

// Use Optional for potentially absent values
public Optional<User> findById(String id) {
    return Optional.ofNullable(repository.findById(id));
}

// Optional usage
Optional<User> userOpt = findById("123");

// Good patterns
String name = userOpt.map(User::getName).orElse("Unknown");
userOpt.ifPresent(user -> sendEmail(user));
User user = userOpt.orElseThrow(() -> new UserNotFoundException(id));

// Bad patterns - avoid
if (userOpt.isPresent()) {
    User user = userOpt.get();  // Anti-pattern
}

// Validate inputs early
public void processPayment(PaymentRequest request) {
    Objects.requireNonNull(request, "Request cannot be null");
    Objects.requireNonNull(request.getAmount(), "Amount cannot be null");
    
    if (request.getAmount().compareTo(BigDecimal.ZERO) <= 0) {
        throw new IllegalArgumentException("Amount must be positive");
    }
    // Process...
}
```

### Immutability

```java
// Make classes immutable when possible
public final class Money {
    private final BigDecimal amount;
    private final Currency currency;
    
    public Money(BigDecimal amount, Currency currency) {
        this.amount = amount;
        this.currency = currency;
    }
    
    public Money add(Money other) {
        requireSameCurrency(other);
        return new Money(this.amount.add(other.amount), this.currency);
    }
    
    public BigDecimal getAmount() { return amount; }
    public Currency getCurrency() { return currency; }
}

// Use unmodifiable collections
public List<String> getItems() {
    return Collections.unmodifiableList(items);
}

// Java 9+ factory methods create immutable collections
List<String> list = List.of("a", "b", "c");
Set<String> set = Set.of("x", "y", "z");
Map<String, Integer> map = Map.of("one", 1, "two", 2);
```

### Exception Handling

```java
// Use specific exceptions
public User findById(String id) throws UserNotFoundException {
    return repository.findById(id)
        .orElseThrow(() -> new UserNotFoundException(id));
}

// Include context in exceptions
public void processOrder(String orderId) {
    try {
        Order order = orderRepository.findById(orderId);
        paymentService.process(order);
    } catch (PaymentException e) {
        throw new OrderProcessingException(
            "Failed to process order: " + orderId, e);  // Include cause
    }
}

// Use try-with-resources
public String readFile(Path path) throws IOException {
    try (BufferedReader reader = Files.newBufferedReader(path)) {
        return reader.lines().collect(Collectors.joining("\n"));
    }
}

// Don't catch generic Exception
// Bad
try {
    processData();
} catch (Exception e) {  // Catches everything including bugs
    log.error("Error", e);
}

// Good
try {
    processData();
} catch (DataAccessException e) {
    log.error("Database error", e);
    throw new ServiceException("Failed to process data", e);
}
```

### Defensive Programming

```java
// Defensive copies
public class Order {
    private final List<Item> items;
    
    public Order(List<Item> items) {
        this.items = new ArrayList<>(items);  // Defensive copy in
    }
    
    public List<Item> getItems() {
        return Collections.unmodifiableList(items);  // Defensive copy out
    }
}

// Validate state
public class Account {
    private BigDecimal balance;
    
    public void withdraw(BigDecimal amount) {
        if (amount.compareTo(BigDecimal.ZERO) <= 0) {
            throw new IllegalArgumentException("Amount must be positive");
        }
        if (amount.compareTo(balance) > 0) {
            throw new InsufficientFundsException(balance, amount);
        }
        balance = balance.subtract(amount);
    }
}

// Fail fast
public class EmailService {
    private final SmtpClient client;
    
    public EmailService(SmtpClient client) {
        this.client = Objects.requireNonNull(client);  // Fail at construction
    }
}
```

### Logging Best Practices

```java
public class OrderService {
    private static final Logger log = LoggerFactory.getLogger(OrderService.class);
    
    public Order createOrder(OrderRequest request) {
        log.info("Creating order for customer: {}", request.getCustomerId());
        
        try {
            Order order = processOrder(request);
            log.info("Order created successfully: {}", order.getId());
            return order;
        } catch (Exception e) {
            log.error("Failed to create order for customer: {}", 
                request.getCustomerId(), e);
            throw e;
        }
    }
    
    // Use appropriate log levels
    // ERROR: failures requiring attention
    // WARN: potential issues, recoverable errors
    // INFO: significant events (start/stop, config)
    // DEBUG: detailed flow information
    // TRACE: very detailed debugging
    
    // Avoid string concatenation in logs
    // Bad (string always built)
    log.debug("Processing user: " + userId + " with data: " + data);
    
    // Good (only built if DEBUG enabled)
    log.debug("Processing user: {} with data: {}", userId, data);
}
```

### Testing Practices

```java
// Unit test structure: Arrange-Act-Assert
@Test
void shouldCalculateOrderTotalWithDiscount() {
    // Arrange
    Order order = new Order();
    order.addItem(new Item("Product A", BigDecimal.valueOf(100)));
    order.addItem(new Item("Product B", BigDecimal.valueOf(50)));
    Customer premiumCustomer = new Customer(CustomerType.PREMIUM);
    
    // Act
    BigDecimal total = orderService.calculateTotal(order, premiumCustomer);
    
    // Assert
    assertThat(total).isEqualTo(BigDecimal.valueOf(135));  // 10% discount
}

// Test edge cases
@ParameterizedTest
@ValueSource(strings = {"", " ", "   "})
void shouldRejectBlankUsername(String username) {
    assertThrows(IllegalArgumentException.class, 
        () -> userService.createUser(username, "email@test.com"));
}

// Use meaningful test names
@Test
void findById_WhenUserExists_ReturnsUser() { }

@Test
void findById_WhenUserDoesNotExist_ThrowsNotFoundException() { }
```

## Interview Questions

**Q1: What are SOLID principles?**

Single Responsibility: class has one reason to change. Open/Closed: open for extension, closed for modification. Liskov Substitution: subtypes replaceable for base types. Interface Segregation: small, specific interfaces. Dependency Inversion: depend on abstractions, not concretions.

**Q2: How to handle null effectively?**

Return Optional for potentially absent values, empty collections instead of null, use Objects.requireNonNull() for validation. Use @NonNull/@Nullable annotations. Fail fast on null inputs. Avoid returning null from methods.

**Q3: Why prefer immutability?**

Thread-safe without synchronization, simpler reasoning, safe as map keys/set elements, no defensive copies needed, enables caching. Make fields final, no setters, defensive copies of mutable inputs, return copies or unmodifiable views.

**Q4: What makes good exception handling?**

Use specific exceptions with context, include cause exception, use try-with-resources, don't catch generic Exception, don't swallow exceptions, document thrown exceptions. Checked for recoverable conditions, unchecked for programming errors.

**Q5: How to write testable code?**

Use dependency injection, program to interfaces, keep methods small and focused, avoid static methods with side effects, separate business logic from infrastructure. Constructor injection for mandatory dependencies.

**Q6: What are defensive programming practices?**

Validate inputs early (fail fast), make defensive copies, use immutable objects, check preconditions/postconditions/invariants, don't expose internal state, use final where possible, handle edge cases explicitly.

**Q7: Explain clean code naming conventions.**

Classes: nouns (Customer, OrderProcessor). Methods: verbs (calculateTotal, sendEmail). Variables: reveal intent (elapsedTimeInDays not d). Booleans: is/has/can prefix. Constants: UPPER_SNAKE_CASE. Avoid abbreviations and single letters except loops.

**Q8: How to structure error handling in services?**

Translate low-level exceptions to domain exceptions, include context (what failed, relevant IDs), log at boundaries, use checked exceptions for recoverable business errors, unchecked for programming errors. Global exception handler for REST APIs.
