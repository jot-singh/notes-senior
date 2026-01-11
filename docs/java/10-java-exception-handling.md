# Java Exception Handling

## Concept Overview

Java's exception handling mechanism enables robust error management through try-catch-finally blocks, exception propagation, and a well-defined exception hierarchy. Proper exception handling separates error handling from normal flow, provides meaningful error information, and ensures resources are properly cleaned up.

## Core Mechanics & Implementation

### Exception Hierarchy

```
Throwable
├── Error (unrecoverable - don't catch)
│   ├── OutOfMemoryError
│   ├── StackOverflowError
│   └── VirtualMachineError
└── Exception
    ├── RuntimeException (unchecked)
    │   ├── NullPointerException
    │   ├── IllegalArgumentException
    │   ├── IllegalStateException
    │   ├── IndexOutOfBoundsException
    │   └── ClassCastException
    └── Checked Exceptions
        ├── IOException
        ├── SQLException
        └── InterruptedException
```

### Checked vs Unchecked Exceptions

```java
// Checked - must be handled or declared
public void readFile(String path) throws IOException {
    BufferedReader reader = new BufferedReader(new FileReader(path));
    // ...
}

// Unchecked (RuntimeException) - no declaration required
public void process(String data) {
    if (data == null) {
        throw new IllegalArgumentException("Data cannot be null");
    }
}

// When to use which:
// Checked: recoverable conditions, caller can reasonably handle
// Unchecked: programming errors, precondition violations, unrecoverable
```

### Try-Catch-Finally

```java
// Basic structure
try {
    riskyOperation();
} catch (SpecificException e) {
    handleSpecific(e);
} catch (GeneralException e) {
    handleGeneral(e);
} finally {
    cleanup();  // Always executes
}

// Multi-catch (Java 7+)
try {
    process();
} catch (IOException | SQLException e) {
    log.error("I/O or database error", e);
}

// Try-with-resources (Java 7+)
try (BufferedReader reader = new BufferedReader(new FileReader(path));
     BufferedWriter writer = new BufferedWriter(new FileWriter(output))) {
    String line;
    while ((line = reader.readLine()) != null) {
        writer.write(line);
    }
}  // Resources automatically closed in reverse order
```

### Custom Exceptions

```java
// Domain-specific checked exception
public class OrderProcessingException extends Exception {
    private final String orderId;
    private final ErrorCode errorCode;
    
    public OrderProcessingException(String message, String orderId, ErrorCode code) {
        super(message);
        this.orderId = orderId;
        this.errorCode = code;
    }
    
    public OrderProcessingException(String message, String orderId, Throwable cause) {
        super(message, cause);
        this.orderId = orderId;
        this.errorCode = ErrorCode.UNKNOWN;
    }
    
    public String getOrderId() { return orderId; }
    public ErrorCode getErrorCode() { return errorCode; }
}

// Unchecked exception for invalid input
public class InvalidRequestException extends RuntimeException {
    private final List<ValidationError> errors;
    
    public InvalidRequestException(List<ValidationError> errors) {
        super("Validation failed: " + errors);
        this.errors = List.copyOf(errors);
    }
    
    public List<ValidationError> getErrors() { return errors; }
}
```

### Exception Handling Patterns

```java
// Catch and rethrow with context
public Order processOrder(OrderRequest request) {
    try {
        return orderRepository.save(createOrder(request));
    } catch (DataAccessException e) {
        throw new OrderProcessingException(
            "Failed to save order for customer: " + request.getCustomerId(),
            request.getId(),
            e  // Preserve cause
        );
    }
}

// Convert checked to unchecked
public void loadConfig() {
    try {
        properties.load(new FileInputStream("config.properties"));
    } catch (IOException e) {
        throw new UncheckedIOException("Config file not found", e);
    }
}

// Exception translation at boundaries
@ExceptionHandler(DataAccessException.class)
public ResponseEntity<ErrorResponse> handleDatabaseError(DataAccessException e) {
    log.error("Database error", e);
    return ResponseEntity
        .status(HttpStatus.INTERNAL_SERVER_ERROR)
        .body(new ErrorResponse("A database error occurred"));
}
```

### Resource Management

```java
// AutoCloseable implementation
public class DatabaseConnection implements AutoCloseable {
    private Connection connection;
    
    public DatabaseConnection(String url) throws SQLException {
        this.connection = DriverManager.getConnection(url);
    }
    
    @Override
    public void close() throws SQLException {
        if (connection != null && !connection.isClosed()) {
            connection.close();
        }
    }
}

// Proper resource handling
public List<User> queryUsers(String sql) throws SQLException {
    try (DatabaseConnection db = new DatabaseConnection(url);
         PreparedStatement stmt = db.prepareStatement(sql);
         ResultSet rs = stmt.executeQuery()) {
        
        List<User> users = new ArrayList<>();
        while (rs.next()) {
            users.add(mapUser(rs));
        }
        return users;
    }
}
```

## Best Practices

```java
// DO: Include context in exceptions
throw new IllegalArgumentException("User ID must be positive, got: " + userId);

// DO: Preserve exception chain
catch (SQLException e) {
    throw new DataAccessException("Failed to query users", e);
}

// DO: Use specific exceptions
catch (FileNotFoundException e) {
    // Handle missing file specifically
} catch (IOException e) {
    // Handle other I/O errors
}

// DON'T: Catch generic Exception
catch (Exception e) {  // Catches bugs too!
    // ...
}

// DON'T: Swallow exceptions
catch (Exception e) {
    // Silent failure - bad!
}

// DON'T: Use exceptions for flow control
try {
    int value = Integer.parseInt(str);
} catch (NumberFormatException e) {
    value = 0;  // Use validation instead
}
```

## Interview Questions

**Q1: What is the difference between checked and unchecked exceptions?**

Checked exceptions must be declared or caught (IOException, SQLException). Compiler enforces handling. Use for recoverable conditions caller should handle. Unchecked (RuntimeException) don't require declaration. Use for programming errors and precondition violations.

**Q2: When should you create custom exceptions?**

When domain needs specific error representation, when you need to carry additional context (error codes, affected entities), when translating low-level exceptions to domain language, when you want type-safe exception handling at different layers.

**Q3: How does try-with-resources work?**

Resources implementing AutoCloseable are automatically closed when try block exits (normal or exceptional). Closed in reverse order of declaration. Suppressed exceptions preserved if close() throws while handling another exception. Access via getSuppressed().

**Q4: What is exception chaining?**

Passing cause exception to new exception's constructor. Preserves original stack trace and error context. Essential for debugging. Use `new Exception(message, cause)` or `initCause()`. Access via `getCause()`.

**Q5: Should you catch Error?**

Generally no. Errors indicate serious JVM problems (OutOfMemoryError, StackOverflowError). Application usually cannot recover. May catch in specific cases: cleanup before termination, logging, or framework code that must continue.

**Q6: What happens if exception thrown in finally block?**

Finally exception suppresses try/catch exception (lost). Use try-with-resources to preserve both (suppressed exception). Avoid throwing from finally. If must, handle internally or wrap original.

**Q7: Explain @ExceptionHandler in Spring.**

Method-level annotation handling specific exceptions in controller. Return ResponseEntity for custom response. Use @ControllerAdvice for global exception handling across controllers. Can specify multiple exception types.

**Q8: What are best practices for exception messages?**

Include what went wrong, relevant context (IDs, values), what was expected vs actual. Don't include sensitive data. Be specific enough for debugging. Use structured logging with exception parameter.
