# Java Fundamentals

## Concept Overview

Java is a statically-typed, object-oriented programming language designed for platform independence through the "write once, run anywhere" (WORA) principle. Released by Sun Microsystems in 1995, Java compiles source code to bytecode that executes on the Java Virtual Machine (JVM), enabling cross-platform compatibility without recompilation.

Java occupies a critical position in enterprise software development, Android mobile applications, distributed systems, and cloud-native architectures. The language combines memory safety through garbage collection, strong typing for compile-time error detection, and a comprehensive standard library that handles common programming tasks. Its ecosystem includes mature frameworks (Spring, Hibernate), build tools (Maven, Gradle), and extensive third-party libraries.

Java solves several fundamental problems: memory management through automatic garbage collection eliminates manual memory handling and reduces memory leaks; platform independence allows deployment across different operating systems without modification; strong typing catches errors at compile time rather than runtime; and built-in concurrency support enables scalable multi-threaded applications. The language's backward compatibility ensures code written decades ago continues functioning with newer JVM versions, making it suitable for long-term enterprise projects.

Use Java when building enterprise applications requiring robustness and maintainability, distributed systems needing proven libraries for networking and concurrency, Android mobile applications, big data processing tools (Hadoop, Spark), microservices architectures with Spring Boot, or any system where team size and code maintainability outweigh execution speed concerns.

## Core Mechanics & Implementation

### Type System and Variables

Java implements a two-tier type system: primitive types stored directly in memory for performance, and reference types stored as pointers to heap objects. The eight primitive types (byte, short, int, long, float, double, char, boolean) have fixed sizes and value semantics, while reference types (classes, interfaces, arrays) have identity and are compared by reference rather than value.

```java
// Primitives - value stored directly
int count = 42;              // 32-bit signed integer
long timestamp = 1234567890L; // 64-bit signed integer, L suffix required
double price = 99.99;         // 64-bit floating point
boolean active = true;        // true or false only

// Reference types - variable holds memory address
String message = "Hello";     // String object on heap
Integer wrapped = 100;        // Wrapper class for int primitive
int[] numbers = {1, 2, 3};   // Array object on heap

// Autoboxing/unboxing between primitives and wrappers
Integer obj = 42;            // Autoboxing: int -> Integer
int value = obj;             // Unboxing: Integer -> int
```

Variable scopes determine visibility and lifetime. Instance variables belong to objects, class variables (static) are shared across all instances, local variables exist within methods, and parameters pass data into methods. Java initializes instance and class variables to default values (0, false, null) but requires explicit initialization of local variables before use.

### Object-Oriented Principles

Java enforces object-oriented design through classes as blueprints for objects. Encapsulation bundles data with methods and restricts direct access through access modifiers (private, protected, public, package-private). Inheritance enables code reuse through class hierarchies using `extends`, with Java supporting single inheritance for classes but multiple inheritance for interfaces through `implements`.

```java
// Encapsulation: hide internal state, expose through methods
public class BankAccount {
    private double balance;              // Private field, hidden from outside
    private final String accountNumber;  // Immutable after construction
    
    public BankAccount(String accountNumber, double initialBalance) {
        this.accountNumber = accountNumber;
        this.balance = initialBalance;
    }
    
    // Controlled access through public methods
    public void deposit(double amount) {
        if (amount > 0) {
            balance += amount;
        }
    }
    
    public boolean withdraw(double amount) {
        if (amount > 0 && amount <= balance) {
            balance -= amount;
            return true;
        }
        return false;
    }
    
    public double getBalance() {
        return balance;  // Read-only access
    }
}

// Inheritance: reuse and extend functionality
public class SavingsAccount extends BankAccount {
    private double interestRate;
    
    public SavingsAccount(String accountNumber, double initialBalance, double interestRate) {
        super(accountNumber, initialBalance);  // Call parent constructor
        this.interestRate = interestRate;
    }
    
    // Add new behavior
    public void applyInterest() {
        double interest = getBalance() * interestRate;
        deposit(interest);
    }
    
    // Override parent behavior (polymorphism)
    @Override
    public boolean withdraw(double amount) {
        // Add minimum balance check
        if (getBalance() - amount < 100) {
            return false;
        }
        return super.withdraw(amount);
    }
}
```

Polymorphism allows treating objects of different types uniformly through common interfaces or base classes. Method overloading (compile-time polymorphism) allows multiple methods with the same name but different parameters, while method overriding (runtime polymorphism) lets subclasses provide specific implementations of inherited methods.

```java
// Interface defines contract
public interface PaymentProcessor {
    boolean processPayment(double amount);
    void refund(String transactionId);
}

// Multiple implementations of same interface
public class CreditCardProcessor implements PaymentProcessor {
    @Override
    public boolean processPayment(double amount) {
        // Credit card specific logic
        return chargeCreditCard(amount);
    }
    
    @Override
    public void refund(String transactionId) {
        // Credit card refund logic
    }
}

public class PayPalProcessor implements PaymentProcessor {
    @Override
    public boolean processPayment(double amount) {
        // PayPal API specific logic
        return payPalAPI.charge(amount);
    }
    
    @Override
    public void refund(String transactionId) {
        // PayPal refund logic
    }
}

// Polymorphic usage - same code works with different implementations
public class CheckoutService {
    public void checkout(PaymentProcessor processor, double amount) {
        // processor could be CreditCard or PayPal, works either way
        if (processor.processPayment(amount)) {
            System.out.println("Payment successful");
        }
    }
}
```

### Abstraction and Interfaces

Abstraction hides implementation complexity behind simpler interfaces. Abstract classes provide partial implementation with some abstract methods requiring subclass implementation. Interfaces define pure contracts without implementation (prior to Java 8) and support multiple inheritance since a class can implement many interfaces.

Java 8 introduced default methods allowing interfaces to provide implementation, enabling API evolution without breaking existing implementations. Static methods in interfaces provide utility functions related to the interface contract.

```java
// Abstract class with partial implementation
public abstract class Shape {
    protected String color;
    
    public Shape(String color) {
        this.color = color;
    }
    
    // Concrete method available to all subclasses
    public String getColor() {
        return color;
    }
    
    // Abstract method - subclasses must implement
    public abstract double calculateArea();
    public abstract double calculatePerimeter();
}

public class Circle extends Shape {
    private double radius;
    
    public Circle(String color, double radius) {
        super(color);
        this.radius = radius;
    }
    
    @Override
    public double calculateArea() {
        return Math.PI * radius * radius;
    }
    
    @Override
    public double calculatePerimeter() {
        return 2 * Math.PI * radius;
    }
}

// Interface with default and static methods (Java 8+)
public interface Drawable {
    // Abstract method - must implement
    void draw();
    
    // Default method - can override but not required
    default void drawWithBorder() {
        drawBorder();
        draw();
        drawBorder();
    }
    
    private void drawBorder() {
        System.out.println("==========");
    }
    
    // Static utility method
    static Drawable createEmpty() {
        return () -> System.out.println("Empty drawing");
    }
}
```

### Access Modifiers and Visibility

Java provides four access levels controlling visibility: private restricts access to the defining class only; package-private (default, no keyword) allows access within the same package; protected grants access to subclasses and package members; public allows access from anywhere. Proper use of access modifiers enforces encapsulation and defines clear public APIs while hiding implementation details.

```java
public class AccessExample {
    private int privateField;           // Only within this class
    int packageField;                   // Same package only (default)
    protected int protectedField;       // Subclasses and same package
    public int publicField;             // Accessible everywhere
    
    private void privateMethod() {
        // Internal implementation, not part of public API
    }
    
    protected void protectedMethod() {
        // Available to subclasses for extension points
    }
    
    public void publicMethod() {
        // Part of class's public contract
        privateMethod();  // Can call private methods internally
    }
}
```

### Static Members and Initialization

Static members belong to the class rather than instances, shared across all objects. Static variables store class-level state, while static methods perform operations not requiring instance data. Static initialization blocks execute when the class first loads, useful for complex static variable initialization.

```java
public class Configuration {
    // Static variable - one copy shared by all instances
    private static final String APP_NAME = "MyApp";
    private static Properties config;
    
    // Static initialization block - runs once when class loads
    static {
        config = new Properties();
        try {
            config.load(new FileInputStream("config.properties"));
        } catch (IOException e) {
            // Handle initialization error
            config.setProperty("default", "value");
        }
    }
    
    // Static method - no access to instance variables
    public static String getProperty(String key) {
        return config.getProperty(key);
    }
    
    // Instance method can access static members
    public void printAppName() {
        System.out.println(APP_NAME);  // Access static field
    }
}

// Usage - no instance needed for static members
String value = Configuration.getProperty("database.url");
```

### Final Keyword

The `final` keyword provides immutability guarantees at different levels. Final variables cannot be reassigned after initialization, creating constants. Final methods cannot be overridden by subclasses, preserving behavior. Final classes cannot be extended, preventing inheritance entirely.

```java
public final class ImmutablePerson {
    private final String name;      // Must initialize in constructor
    private final int age;
    private final List<String> hobbies;
    
    public ImmutablePerson(String name, int age, List<String> hobbies) {
        this.name = name;
        this.age = age;
        // Defensive copy to prevent external modification
        this.hobbies = new ArrayList<>(hobbies);
    }
    
    public String getName() {
        return name;
    }
    
    public int getAge() {
        return age;
    }
    
    public List<String> getHobbies() {
        // Return unmodifiable view to maintain immutability
        return Collections.unmodifiableList(hobbies);
    }
}

// Method parameters can be final
public void processData(final String input) {
    // input = "new value";  // Compilation error - cannot reassign final parameter
    System.out.println(input);
}
```

### Generics

Generics enable type-safe code that works with multiple types without sacrificing compile-time type checking. Generic classes and methods use type parameters (typically T, E, K, V) as placeholders replaced with concrete types at compile time. Type erasure removes generic information at runtime, replacing type parameters with bounds or Object, enabling backward compatibility with pre-generics code.

```java
// Generic class with type parameter
public class Box<T> {
    private T content;
    
    public void set(T content) {
        this.content = content;
    }
    
    public T get() {
        return content;
    }
}

// Usage with different types
Box<String> stringBox = new Box<>();
stringBox.set("Hello");
String value = stringBox.get();  // No cast needed, type-safe

Box<Integer> intBox = new Box<>();
intBox.set(42);
Integer number = intBox.get();

// Generic method
public <T extends Comparable<T>> T findMax(List<T> list) {
    if (list.isEmpty()) {
        return null;
    }
    
    T max = list.get(0);
    for (T item : list) {
        if (item.compareTo(max) > 0) {
            max = item;
        }
    }
    return max;
}

// Bounded type parameters
public class NumberBox<T extends Number> {
    private T number;
    
    public void set(T number) {
        this.number = number;
    }
    
    // Can call Number methods on T
    public double getDoubleValue() {
        return number.doubleValue();
    }
}

// Wildcards for flexible method parameters
public void processBoxes(List<? extends Number> boxes) {
    // Can read as Number, cannot add (except null)
    for (Number num : boxes) {
        System.out.println(num.doubleValue());
    }
}

public void addNumbers(List<? super Integer> list) {
    // Can add Integer, cannot read (except as Object)
    list.add(42);
    list.add(100);
}
```

## Best Practices & Patterns

### SOLID Principles Application

Apply Single Responsibility Principle by ensuring each class has one reason to change. Separate concerns into focused classes rather than creating god objects. Use Open/Closed Principle through composition and interfaces, allowing extension through new implementations without modifying existing code. Implement Liskov Substitution by ensuring subclasses can replace parent classes without breaking functionality. Apply Interface Segregation by creating focused interfaces rather than large, monolithic ones. Use Dependency Inversion by depending on abstractions rather than concrete implementations.

```java
// Single Responsibility - each class has one job
public class UserRepository {
    public User findById(String id) { /* data access only */ }
    public void save(User user) { /* data access only */ }
}

public class UserValidator {
    public boolean isValid(User user) { /* validation only */ }
}

public class UserService {
    private final UserRepository repository;
    private final UserValidator validator;
    
    // Dependency Inversion - depend on abstractions
    public UserService(UserRepository repository, UserValidator validator) {
        this.repository = repository;
        this.validator = validator;
    }
    
    public void registerUser(User user) {
        if (validator.isValid(user)) {
            repository.save(user);
        }
    }
}

// Interface Segregation - small, focused interfaces
public interface Readable {
    String read();
}

public interface Writable {
    void write(String data);
}

// Implement only needed interfaces
public class ReadOnlyFile implements Readable {
    @Override
    public String read() {
        return "file content";
    }
}

public class ReadWriteFile implements Readable, Writable {
    @Override
    public String read() {
        return "file content";
    }
    
    @Override
    public void write(String data) {
        // Write implementation
    }
}
```

### Immutability Patterns

Prefer immutable objects that cannot change after construction. Immutable objects are thread-safe, simplify reasoning about code, and prevent defensive copying. Make fields final, provide no setters, and return defensive copies of mutable objects.

```java
public final class Money {
    private final BigDecimal amount;
    private final Currency currency;
    
    public Money(BigDecimal amount, Currency currency) {
        // Validate and create defensive copy
        if (amount == null || currency == null) {
            throw new IllegalArgumentException("Amount and currency required");
        }
        this.amount = amount;
        this.currency = currency;
    }
    
    // Operations return new instances rather than modifying
    public Money add(Money other) {
        if (!this.currency.equals(other.currency)) {
            throw new IllegalArgumentException("Currency mismatch");
        }
        return new Money(this.amount.add(other.amount), this.currency);
    }
    
    public Money multiply(BigDecimal multiplier) {
        return new Money(this.amount.multiply(multiplier), this.currency);
    }
    
    // Only getters, no setters
    public BigDecimal getAmount() {
        return amount;
    }
    
    public Currency getCurrency() {
        return currency;
    }
}
```

### Builder Pattern for Complex Objects

Use Builder pattern for objects with many parameters, especially optional ones. Builders provide readable construction syntax and enforce constraints before creating objects.

```java
public class HttpRequest {
    private final String url;
    private final String method;
    private final Map<String, String> headers;
    private final String body;
    private final int timeout;
    
    // Private constructor, only Builder can call
    private HttpRequest(Builder builder) {
        this.url = builder.url;
        this.method = builder.method;
        this.headers = new HashMap<>(builder.headers);
        this.body = builder.body;
        this.timeout = builder.timeout;
    }
    
    public static class Builder {
        private final String url;              // Required
        private String method = "GET";          // Optional with default
        private Map<String, String> headers = new HashMap<>();
        private String body;
        private int timeout = 30000;
        
        public Builder(String url) {
            this.url = url;
        }
        
        public Builder method(String method) {
            this.method = method;
            return this;
        }
        
        public Builder header(String key, String value) {
            this.headers.put(key, value);
            return this;
        }
        
        public Builder body(String body) {
            this.body = body;
            return this;
        }
        
        public Builder timeout(int timeout) {
            this.timeout = timeout;
            return this;
        }
        
        public HttpRequest build() {
            // Validate before constructing
            if (url == null || url.isEmpty()) {
                throw new IllegalStateException("URL required");
            }
            if (body != null && method.equals("GET")) {
                throw new IllegalStateException("GET requests cannot have body");
            }
            return new HttpRequest(this);
        }
    }
}

// Usage - readable and flexible
HttpRequest request = new HttpRequest.Builder("https://api.example.com")
    .method("POST")
    .header("Content-Type", "application/json")
    .header("Authorization", "Bearer token123")
    .body("{\"name\":\"John\"}")
    .timeout(5000)
    .build();
```

### Effective equals() and hashCode()

Override `equals()` and `hashCode()` together when objects need value-based equality. Ensure consistency: objects equal by `equals()` must have identical hash codes. Use all significant fields in both methods.

```java
public class Person {
    private final String id;
    private final String name;
    private final int age;
    
    @Override
    public boolean equals(Object obj) {
        if (this == obj) return true;  // Same reference
        if (obj == null || getClass() != obj.getClass()) return false;
        
        Person other = (Person) obj;
        return age == other.age &&
               Objects.equals(id, other.id) &&
               Objects.equals(name, other.name);
    }
    
    @Override
    public int hashCode() {
        return Objects.hash(id, name, age);  // Use same fields as equals
    }
    
    @Override
    public String toString() {
        return String.format("Person{id='%s', name='%s', age=%d}", id, name, age);
    }
}
```

### Resource Management

Use try-with-resources for automatic resource cleanup. Resources implementing AutoCloseable close automatically when exiting the try block, even if exceptions occur. This prevents resource leaks from forgotten close() calls.

```java
// Try-with-resources - automatic cleanup
public String readFile(String path) throws IOException {
    try (BufferedReader reader = new BufferedReader(new FileReader(path))) {
        StringBuilder content = new StringBuilder();
        String line;
        while ((line = reader.readLine()) != null) {
            content.append(line).append("\n");
        }
        return content.toString();
    }  // reader.close() called automatically, even if exception thrown
}

// Multiple resources
public void copyFile(String source, String destination) throws IOException {
    try (InputStream in = new FileInputStream(source);
         OutputStream out = new FileOutputStream(destination)) {
        
        byte[] buffer = new byte[8192];
        int bytesRead;
        while ((bytesRead = in.read(buffer)) != -1) {
            out.write(buffer, 0, bytesRead);
        }
    }  // Both streams closed in reverse order
}

// Custom AutoCloseable resource
public class DatabaseConnection implements AutoCloseable {
    private Connection connection;
    
    public DatabaseConnection(String url) throws SQLException {
        this.connection = DriverManager.getConnection(url);
    }
    
    public void executeQuery(String sql) throws SQLException {
        try (Statement stmt = connection.createStatement();
             ResultSet rs = stmt.executeQuery(sql)) {
            // Process results
        }
    }
    
    @Override
    public void close() throws SQLException {
        if (connection != null && !connection.isClosed()) {
            connection.close();
        }
    }
}
```

## Real-World Applications

### Enterprise Application Architecture

Java dominates enterprise systems through frameworks like Spring Boot for microservices, Jakarta EE for traditional enterprise applications, and Apache Camel for integration. Large organizations use Java for its maintainability, extensive tooling, and mature ecosystem. Financial services rely on Java for transaction processing systems requiring reliability and performance. E-commerce platforms use Java for order processing, inventory management, and payment systems handling millions of transactions.

```java
// Spring Boot microservice example
@RestController
@RequestMapping("/api/orders")
public class OrderController {
    private final OrderService orderService;
    private final OrderValidator validator;
    
    @Autowired
    public OrderController(OrderService orderService, OrderValidator validator) {
        this.orderService = orderService;
        this.validator = validator;
    }
    
    @PostMapping
    public ResponseEntity<OrderResponse> createOrder(@RequestBody OrderRequest request) {
        ValidationResult validation = validator.validate(request);
        if (!validation.isValid()) {
            return ResponseEntity.badRequest()
                .body(new OrderResponse(validation.getErrors()));
        }
        
        Order order = orderService.createOrder(request);
        return ResponseEntity.status(HttpStatus.CREATED)
            .body(new OrderResponse(order));
    }
    
    @GetMapping("/{id}")
    public ResponseEntity<Order> getOrder(@PathVariable String id) {
        return orderService.findById(id)
            .map(ResponseEntity::ok)
            .orElse(ResponseEntity.notFound().build());
    }
}
```

### Android Mobile Development

Android applications use Java (now primarily Kotlin, but Java remains fully supported) for mobile app development. Android SDK provides UI components, lifecycle management, background services, and hardware access. Major applications like banking apps, social media clients, and business tools use Java for Android development.

```java
// Android Activity example
public class MainActivity extends AppCompatActivity {
    private TextView textView;
    private Button button;
    private ApiService apiService;
    
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        
        textView = findViewById(R.id.textView);
        button = findViewById(R.id.button);
        apiService = RetrofitClient.getInstance().create(ApiService.class);
        
        button.setOnClickListener(v -> loadData());
    }
    
    private void loadData() {
        apiService.getData().enqueue(new Callback<DataResponse>() {
            @Override
            public void onResponse(Call<DataResponse> call, Response<DataResponse> response) {
                if (response.isSuccessful() && response.body() != null) {
                    textView.setText(response.body().getMessage());
                }
            }
            
            @Override
            public void onFailure(Call<DataResponse> call, Throwable t) {
                Toast.makeText(MainActivity.this, "Error loading data", Toast.LENGTH_SHORT).show();
            }
        });
    }
    
    @Override
    protected void onDestroy() {
        super.onDestroy();
        // Clean up resources
    }
}
```

### Big Data Processing

Java powers big data frameworks including Apache Hadoop for distributed storage and processing, Apache Spark for in-memory analytics, Apache Kafka for event streaming, and Apache Flink for stream processing. Companies process petabytes of data using Java-based tools for analytics, machine learning pipelines, and real-time processing.

```java
// Apache Spark job example
public class LogAnalyzer {
    public static void main(String[] args) {
        SparkConf conf = new SparkConf()
            .setAppName("Log Analyzer")
            .setMaster("yarn");
        
        JavaSparkContext sc = new JavaSparkContext(conf);
        
        // Read logs from HDFS
        JavaRDD<String> logs = sc.textFile("hdfs://logs/access.log");
        
        // Filter and transform
        JavaPairRDD<String, Integer> errorCounts = logs
            .filter(line -> line.contains("ERROR"))
            .mapToPair(line -> {
                String[] parts = line.split(" ");
                String endpoint = parts[6];  // Extract endpoint from log
                return new Tuple2<>(endpoint, 1);
            })
            .reduceByKey((a, b) -> a + b);
        
        // Get top 10 endpoints with errors
        List<Tuple2<String, Integer>> top10 = errorCounts
            .mapToPair(Tuple2::swap)  // Swap to sort by count
            .sortByKey(false)
            .take(10);
        
        // Save results
        errorCounts.saveAsTextFile("hdfs://output/error-analysis");
        
        sc.close();
    }
}
```

### Cloud-Native Applications

Java integrates with cloud platforms through AWS SDK, Azure SDK, Google Cloud SDK. Spring Cloud provides patterns for distributed systems including service discovery, configuration management, circuit breakers, and API gateways. Containerized Java applications run efficiently in Kubernetes with proper JVM tuning for container environments.

```java
// AWS Lambda handler example
public class S3EventHandler implements RequestHandler<S3Event, String> {
    private final AmazonS3 s3Client = AmazonS3ClientBuilder.defaultClient();
    private final AmazonDynamoDB dynamoDb = AmazonDynamoDBClientBuilder.defaultClient();
    
    @Override
    public String handleRequest(S3Event event, Context context) {
        LambdaLogger logger = context.getLogger();
        
        event.getRecords().forEach(record -> {
            String bucket = record.getS3().getBucket().getName();
            String key = record.getS3().getObject().getKey();
            
            logger.log("Processing file: " + key + " from bucket: " + bucket);
            
            try {
                // Download and process file
                S3Object s3Object = s3Client.getObject(bucket, key);
                processFile(s3Object.getObjectContent());
                
                // Store metadata in DynamoDB
                saveMetadata(key, s3Object.getObjectMetadata());
                
            } catch (Exception e) {
                logger.log("Error processing file: " + e.getMessage());
                throw new RuntimeException(e);
            }
        });
        
        return "Processed " + event.getRecords().size() + " files";
    }
    
    private void processFile(InputStream input) {
        // File processing logic
    }
    
    private void saveMetadata(String key, ObjectMetadata metadata) {
        // Save to DynamoDB
    }
}
```

## Common Pitfalls & Anti-patterns

### Null Pointer Exceptions

Null references cause NullPointerException, one of the most common runtime errors. Always validate null before dereferencing. Use Optional<T> to explicitly represent absence of value. Consider null-object pattern or defensive programming with null checks.

```java
// Bad - no null check
public void processUser(User user) {
    String name = user.getName();  // NPE if user is null
    System.out.println(name.toUpperCase());  // NPE if name is null
}

// Good - defensive null checks
public void processUser(User user) {
    if (user == null) {
        throw new IllegalArgumentException("User cannot be null");
    }
    
    String name = user.getName();
    if (name != null) {
        System.out.println(name.toUpperCase());
    }
}

// Better - use Optional
public Optional<String> getUserName(String userId) {
    User user = repository.findById(userId);
    return Optional.ofNullable(user)
        .map(User::getName);
}

// Usage with Optional
Optional<String> name = getUserName("123");
name.ifPresent(n -> System.out.println(n.toUpperCase()));

String displayName = name.orElse("Unknown");
String required = name.orElseThrow(() -> new UserNotFoundException());
```

### String Concatenation in Loops

String is immutable, so concatenation creates new objects repeatedly in loops, causing poor performance. Use StringBuilder for efficient string building.

```java
// Bad - creates N string objects
public String buildCsv(List<String> values) {
    String result = "";
    for (String value : values) {
        result += value + ",";  // New string created each iteration
    }
    return result;
}

// Good - single StringBuilder, efficient
public String buildCsv(List<String> values) {
    StringBuilder result = new StringBuilder();
    for (String value : values) {
        result.append(value).append(",");
    }
    return result.toString();
}

// Better - use String.join() for simple cases
public String buildCsv(List<String> values) {
    return String.join(",", values);
}
```

### Ignoring Exceptions

Catching exceptions without handling them hides errors and makes debugging difficult. Either handle the exception properly, log it, or let it propagate with appropriate context.

```java
// Bad - swallowing exceptions
try {
    processData();
} catch (Exception e) {
    // Silent failure, no visibility into problems
}

// Bad - generic catch loses context
try {
    processData();
} catch (Exception e) {
    System.out.println("Error occurred");  // Not enough information
}

// Good - handle or propagate with context
try {
    processData();
} catch (IOException e) {
    logger.error("Failed to process data file", e);
    throw new DataProcessingException("Unable to process data", e);
} catch (SQLException e) {
    logger.error("Database error during processing", e);
    throw new DataProcessingException("Database error", e);
}

// Good - let it propagate if you can't handle
public void processFile(String path) throws IOException {
    // Caller handles IOException
    Files.readAllLines(Paths.get(path));
}
```

### Inappropriate Use of Inheritance

Overusing inheritance creates tight coupling and fragile class hierarchies. Prefer composition over inheritance for code reuse. Use inheritance only for true "is-a" relationships with polymorphic behavior.

```java
// Bad - inheritance for code reuse
public class Stack extends ArrayList<Object> {
    public void push(Object item) {
        add(item);
    }
    
    public Object pop() {
        return remove(size() - 1);
    }
    
    // Problem: exposes all ArrayList methods, breaking Stack contract
    // stack.add(3, item) - adds at index, not stack behavior
}

// Good - composition
public class Stack {
    private final List<Object> elements = new ArrayList<>();
    
    public void push(Object item) {
        elements.add(item);
    }
    
    public Object pop() {
        if (elements.isEmpty()) {
            throw new EmptyStackException();
        }
        return elements.remove(elements.size() - 1);
    }
    
    public boolean isEmpty() {
        return elements.isEmpty();
    }
    
    // Only expose Stack operations, hide implementation
}
```

### Resource Leaks

Failing to close resources (files, connections, streams) causes resource leaks, exhausting system resources over time. Always use try-with-resources or explicit finally blocks.

```java
// Bad - connection never closed on exception
public void queryDatabase(String sql) throws SQLException {
    Connection conn = DriverManager.getConnection(DB_URL);
    Statement stmt = conn.createStatement();
    ResultSet rs = stmt.executeQuery(sql);
    // If exception occurs, resources not closed
    processResults(rs);
    rs.close();
    stmt.close();
    conn.close();
}

// Good - try-with-resources ensures cleanup
public void queryDatabase(String sql) throws SQLException {
    try (Connection conn = DriverManager.getConnection(DB_URL);
         Statement stmt = conn.createStatement();
         ResultSet rs = stmt.executeQuery(sql)) {
        
        processResults(rs);
        // All resources closed automatically, even on exception
    }
}
```

## Advanced Topics & Edge Cases

### Covariance and Contravariance with Generics

Java's generic type system involves variance rules for type safety. Arrays are covariant (String[] is subtype of Object[]), but generics are invariant (List<String> is not subtype of List<Object>). Wildcards provide flexibility: `? extends T` for reading (covariant), `? super T` for writing (contravariant).

```java
// Arrays are covariant - runtime check required
Object[] objects = new String[10];
objects[0] = "Hello";  // OK
objects[1] = 42;       // Compiles, but throws ArrayStoreException at runtime

// Generics are invariant - compile-time safety
List<Object> objects = new ArrayList<String>();  // Compilation error

// Covariant wildcard - can read as Number, cannot add
public double sum(List<? extends Number> numbers) {
    double total = 0;
    for (Number num : numbers) {
        total += num.doubleValue();  // Can read as Number
    }
    // numbers.add(42);  // Compilation error - cannot add
    return total;
}

// Can call with any List of Number subtype
sum(Arrays.asList(1, 2, 3));           // List<Integer>
sum(Arrays.asList(1.5, 2.5, 3.5));     // List<Double>

// Contravariant wildcard - can write Integer, cannot read specific type
public void addIntegers(List<? super Integer> list) {
    list.add(42);      // Can add Integer
    list.add(100);     // Can add Integer
    // Integer value = list.get(0);  // Compilation error - type unknown
    Object obj = list.get(0);         // Can only read as Object
}

// Can call with List<Integer> or List<Number> or List<Object>
addIntegers(new ArrayList<Integer>());
addIntegers(new ArrayList<Number>());
addIntegers(new ArrayList<Object>());
```

### Type Erasure Implications

Generic type information erases at runtime, replaced with bounds or Object. This affects reflection, instance checks, and array creation. Use raw types or wildcards when type information unavailable at runtime.

```java
// Type erasure - both compile to same bytecode
List<String> strings = new ArrayList<>();
List<Integer> integers = new ArrayList<>();

// Cannot distinguish at runtime
System.out.println(strings.getClass() == integers.getClass());  // true

// Cannot create generic arrays directly
// T[] array = new T[10];  // Compilation error

// Workaround for generic array creation
@SuppressWarnings("unchecked")
public <T> T[] createArray(Class<T> type, int size) {
    return (T[]) Array.newInstance(type, size);
}

String[] strings = createArray(String.class, 10);

// Cannot use instanceof with parameterized types
// if (list instanceof List<String>)  // Compilation error
if (list instanceof List<?>) {  // Use wildcard
    // OK, but no type information
}

// Reflection cannot see generic types at runtime
public void printType(List<String> list) {
    Class<?> clazz = list.getClass();
    System.out.println(clazz.getName());  // java.util.ArrayList, no <String>
}
```

### Enum Advanced Usage

Enums can have fields, methods, constructors, and implement interfaces. Each enum constant can override methods for constant-specific behavior. Use enums for strategy pattern implementation.

```java
// Enum with fields and methods
public enum Operation {
    PLUS("+") {
        @Override
        public double apply(double x, double y) {
            return x + y;
        }
    },
    MINUS("-") {
        @Override
        public double apply(double x, double y) {
            return x - y;
        }
    },
    MULTIPLY("*") {
        @Override
        public double apply(double x, double y) {
            return x * y;
        }
    },
    DIVIDE("/") {
        @Override
        public double apply(double x, double y) {
            if (y == 0) {
                throw new ArithmeticException("Division by zero");
            }
            return x / y;
        }
    };
    
    private final String symbol;
    
    Operation(String symbol) {
        this.symbol = symbol;
    }
    
    public abstract double apply(double x, double y);
    
    public String getSymbol() {
        return symbol;
    }
    
    // Enum with lookup by symbol
    private static final Map<String, Operation> BY_SYMBOL = new HashMap<>();
    
    static {
        for (Operation op : values()) {
            BY_SYMBOL.put(op.symbol, op);
        }
    }
    
    public static Optional<Operation> fromSymbol(String symbol) {
        return Optional.ofNullable(BY_SYMBOL.get(symbol));
    }
}

// Usage
double result = Operation.PLUS.apply(10, 5);  // 15.0
Operation op = Operation.fromSymbol("*").orElseThrow();
result = op.apply(10, 5);  // 50.0
```

### Nested Classes

Java supports static nested classes (associated with outer class), inner classes (tied to outer instance), local classes (defined in methods), and anonymous classes (unnamed inline implementations). Each has different access levels and use cases.

```java
public class OuterClass {
    private int outerField = 10;
    private static int staticField = 20;
    
    // Static nested class - no access to outer instance
    public static class StaticNested {
        public void method() {
            // Can access static members of outer
            System.out.println(staticField);
            // Cannot access outerField - no outer instance
        }
    }
    
    // Inner class - has reference to outer instance
    public class Inner {
        public void method() {
            // Can access all outer members
            System.out.println(outerField);
            System.out.println(staticField);
            System.out.println(OuterClass.this.outerField);  // Explicit outer reference
        }
    }
    
    public void outerMethod() {
        final int localVar = 30;
        
        // Local class - defined inside method
        class LocalClass {
            public void method() {
                // Can access outer members and effectively final local variables
                System.out.println(outerField);
                System.out.println(localVar);
            }
        }
        
        LocalClass local = new LocalClass();
        local.method();
        
        // Anonymous class
        Runnable runnable = new Runnable() {
            @Override
            public void run() {
                System.out.println(outerField);
                System.out.println(localVar);
            }
        };
        
        // Lambda equivalent (Java 8+)
        Runnable lambda = () -> {
            System.out.println(outerField);
            System.out.println(localVar);
        };
    }
}

// Usage
OuterClass.StaticNested nested = new OuterClass.StaticNested();
OuterClass outer = new OuterClass();
OuterClass.Inner inner = outer.new Inner();
```

### Serialization

Java serialization converts objects to byte streams for storage or transmission. Classes implement Serializable marker interface. Control serialization with transient fields, custom writeObject/readObject methods, or Externalizable interface. Consider security implications and prefer explicit serialization formats (JSON, Protobuf) for production systems.

```java
public class User implements Serializable {
    private static final long serialVersionUID = 1L;  // Version control
    
    private String username;
    private String email;
    private transient String password;  // Not serialized
    private transient Cache cache;       // Not serializable, skip
    
    // Custom serialization
    private void writeObject(ObjectOutputStream out) throws IOException {
        out.defaultWriteObject();  // Serialize non-transient fields
        // Custom logic
        out.writeUTF(encryptPassword(password));
    }
    
    private void readObject(ObjectInputStream in) throws IOException, ClassNotFoundException {
        in.defaultReadObject();  // Deserialize non-transient fields
        // Custom logic
        this.password = decryptPassword(in.readUTF());
        this.cache = new Cache();  // Reinitialize transient fields
    }
    
    private String encryptPassword(String password) {
        // Encryption logic
        return password;
    }
    
    private String decryptPassword(String encrypted) {
        // Decryption logic
        return encrypted;
    }
}

// Serialization usage
public void saveUser(User user, String filename) throws IOException {
    try (ObjectOutputStream out = new ObjectOutputStream(
            new FileOutputStream(filename))) {
        out.writeObject(user);
    }
}

public User loadUser(String filename) throws IOException, ClassNotFoundException {
    try (ObjectInputStream in = new ObjectInputStream(
            new FileInputStream(filename))) {
        return (User) in.readObject();
    }
}
```

## Interview & Career Scenarios

Interviewers assess Java fundamentals to verify core understanding before diving into advanced topics. Expect questions testing object-oriented principles, type system knowledge, and practical coding ability. Demonstrate understanding of when to use different language features and their trade-offs. Show awareness of best practices and common pitfalls from real-world experience.

Senior-level interviews expect you to discuss design decisions, performance implications, and maintainability concerns. Be prepared to critique code, identify issues, and propose improvements. Explain how Java features enable writing maintainable, scalable systems. Connect language features to architectural patterns and real-world scenarios.

## Interview Questions

**Q1: Explain the difference between abstract classes and interfaces. When would you use each?**

Abstract classes provide partial implementation with state (fields) and can have constructors, enabling code reuse among related classes in an inheritance hierarchy. Use abstract classes when sharing implementation code among closely related classes, when you need non-public members, or when you need fields that aren't static or final.

Interfaces define contracts without implementation (prior to Java 8) and support multiple inheritance, allowing classes to implement multiple behaviors. Use interfaces when unrelated classes should implement the same behavior, when you want to specify behavior without implementation concerns, or when you need multiple inheritance of type. Java 8+ default methods allow limited implementation in interfaces while maintaining multiple inheritance benefits.

Example: An abstract `Vehicle` class provides shared code for `Car` and `Motorcycle` (related classes), while a `Drawable` interface allows unrelated classes (`Circle`, `Button`, `Image`) to share drawing behavior.

**Q2: What is the difference between == and equals() in Java?**

The `==` operator performs reference equality for objects, checking if two references point to the exact same object in memory. For primitives, `==` compares values directly. The `equals()` method performs value equality, comparing object contents according to class-defined logic.

String comparison demonstrates this: `String a = "hello"` and `String b = "hello"` might be `==` true due to string interning, but `String a = new String("hello")` and `String b = new String("hello")` are `==` false (different objects) yet `equals()` true (same content).

Always override `equals()` and `hashCode()` together when implementing value-based equality. Use `Objects.equals(a, b)` for null-safe comparisons.

**Q3: Explain Java's memory model and the difference between stack and heap.**

The stack stores method call frames containing local variables and method parameters. Each thread has its own stack, and memory allocates/deallocates automatically as methods call and return. Stack memory is fast, limited in size, and follows LIFO order. Primitive types and object references (not objects themselves) live on the stack.

The heap stores all objects and instance variables. All threads share the heap, and garbage collection manages memory. Heap is larger than stack but slower to access. When you `new` an object, it goes on the heap, while the reference variable pointing to it goes on the stack.

Static variables reside in a special memory area (method area/metaspace), shared across all instances. Understanding this distinction helps optimize memory usage and prevent stack overflow (too deep recursion) or out-of-memory errors (too many heap objects).

**Q4: What is polymorphism? Provide examples of compile-time and runtime polymorphism.**

Polymorphism allows treating objects of different types uniformly through common interfaces or base classes, enabling flexible and extensible code.

Compile-time polymorphism (static binding) occurs through method overloading: multiple methods with the same name but different parameters. The compiler determines which method to call based on method signature. Example:
```java
public int add(int a, int b) { return a + b; }
public double add(double a, double b) { return a + b; }
public String add(String a, String b) { return a + b; }
```

Runtime polymorphism (dynamic binding) occurs through method overriding: subclasses provide specific implementations of inherited methods. The JVM determines which method to call based on actual object type at runtime. Example:
```java
Animal animal = new Dog();  // Reference type Animal, object type Dog
animal.makeSound();  // Calls Dog's makeSound(), not Animal's
```

Runtime polymorphism enables strategy pattern, plugin architectures, and extensible frameworks where behavior varies based on actual object type.

**Q5: Explain the purpose of the final keyword and its different uses.**

The `final` keyword creates immutability guarantees at three levels:

Final variables cannot be reassigned after initialization, creating constants. For primitives, the value cannot change. For references, the reference cannot point to a different object, but the object itself remains mutable. Use final for configuration values, constants, and enhancing thread safety.

Final methods cannot be overridden by subclasses, preserving behavior and enabling compiler optimizations. Use final methods for core algorithms that shouldn't be modified or when method behavior is critical to class correctness.

Final classes cannot be extended, preventing all inheritance. Use final classes for immutable classes, security-sensitive classes, or when inheritance would violate class invariants. Examples: `String`, `Integer`, `Math`.

Final parameters prevent reassignment within methods, clarifying intent and preventing accidental modification in complex methods.

**Q6: What are Java generics? How does type erasure work?**

Generics enable writing type-safe code that works with multiple types without sacrificing compile-time type checking. Generic classes and methods use type parameters (T, E, K, V) as placeholders replaced with concrete types at compile time, eliminating casts and catching type errors early.

Type erasure removes generic type information at runtime for backward compatibility with pre-generics Java. The compiler replaces type parameters with their bounds (or Object if unbounded) and inserts casts where needed. This means `List<String>` and `List<Integer>` become the same `List` class at runtime.

Type erasure implications: cannot create generic arrays directly, cannot use instanceof with parameterized types, cannot access type parameters in static contexts. Workarounds include using `Class<T>` parameters for reflection-based array creation, wildcards for runtime checks, and understanding that generic type information only exists at compile time.

**Q7: Explain the difference between Comparable and Comparator interfaces.**

`Comparable<T>` defines natural ordering by implementing `compareTo()` within the class itself. Objects compare themselves to others of the same type. Use Comparable when there's one obvious natural ordering. Example: `String` implements Comparable for alphabetical ordering.

```java
public class Person implements Comparable<Person> {
    private String name;
    private int age;
    
    @Override
    public int compareTo(Person other) {
        return this.name.compareTo(other.name);  // Natural order by name
    }
}
```

`Comparator<T>` defines external comparison logic in separate classes, allowing multiple orderings and sorting classes you cannot modify. Use Comparator when you need multiple sort orders or when sorting classes you don't own.

```java
Comparator<Person> byAge = (p1, p2) -> Integer.compare(p1.getAge(), p2.getAge());
Comparator<Person> byName = Comparator.comparing(Person::getName);
Collections.sort(people, byAge);
```

Modern Java provides `Comparator` builder methods: `comparing()`, `thenComparing()`, `reversed()`, `nullsFirst()`, enabling fluent comparison chains.

**Q8: What is the difference between String, StringBuilder, and StringBuffer?**

`String` is immutable: every modification creates a new String object. Strings are thread-safe by immutability, cached in string pool for memory efficiency, but inefficient for repeated modifications since each operation allocates new objects.

`StringBuilder` is mutable and not thread-safe, providing efficient string construction through internal character array that grows as needed. Use StringBuilder for string building in single-threaded contexts, especially in loops or when concatenating many strings.

`StringBuffer` is mutable and thread-safe, synchronizing all methods for concurrent access. Use StringBuffer only when multiple threads access the same string builder, otherwise StringBuilder performs better without synchronization overhead.

```java
// Bad: creates N string objects
String result = "";
for (String s : strings) {
    result += s;  // New string each iteration
}

// Good: single StringBuilder, efficient
StringBuilder sb = new StringBuilder();
for (String s : strings) {
    sb.append(s);
}
String result = sb.toString();
```

**Q9: Explain autoboxing and unboxing. What are the performance implications?**

Autoboxing automatically converts primitives to wrapper objects (int to Integer), while unboxing converts wrappers to primitives. This convenience feature allows using primitives where objects are required, like collections.

```java
List<Integer> numbers = new ArrayList<>();
numbers.add(42);  // Autoboxing: int 42 -> Integer object
int value = numbers.get(0);  // Unboxing: Integer object -> int 42
```

Performance implications: each autoboxing creates a heap object with overhead compared to stack-allocated primitives. Repeated boxing/unboxing in loops or hot paths causes allocation pressure and garbage collection overhead. Wrapper objects consume more memory (Integer uses ~16 bytes vs int's 4 bytes).

Cache: Java caches Integer objects from -128 to 127, making `==` comparison work for small values but fail for larger ones. Always use `equals()` for wrapper comparisons.

Optimization: use primitives in performance-critical code, specialized primitive collections (IntArrayList), and avoid unnecessary boxing in tight loops.

**Q10: What are the access modifiers in Java? Explain their visibility rules.**

Java provides four access levels from most to least restrictive:

`private`: visible only within the defining class. Use for internal implementation details, fields that should be accessed through getters/setters, and helper methods.

Package-private (default, no keyword): visible to classes in the same package. Use for implementation classes not meant for external use, package-level utilities, and collaborating classes within a package.

`protected`: visible to subclasses and classes in the same package. Use for methods/fields intended as extension points for subclasses while hiding from general use.

`public`: visible everywhere. Use for public API methods and classes, constants, and any member meant for external consumption.

Top-level classes can only be public or package-private. Inner classes can use all four modifiers. Proper access control enforces encapsulation: expose minimum necessary interface, hide implementation, prevent misuse of internal APIs.

**Q11: Explain the difference between shallow copy and deep copy.**

Shallow copy duplicates an object but not the objects it references. The copy shares the same referenced objects as the original. Modifying a mutable field in one affects the other. Java's clone() performs shallow copy by default.

```java
class Person {
    String name;
    Address address;  // Reference type
}

Person original = new Person("John", new Address("NY"));
Person shallow = original.clone();  // Shallow copy
shallow.address.setCity("LA");  // Modifies original's address too!
```

Deep copy duplicates an object and recursively copies all objects it references, creating a completely independent copy. Modifications to the copy don't affect the original.

```java
public Person deepCopy() {
    return new Person(
        this.name,  // String immutable, sharing OK
        this.address.clone()  // Clone address too
    );
}
```

Implement deep copy through copy constructors, serialization/deserialization, or copying libraries. Consider immutability to avoid copy complexity: immutable objects can be safely shared.

**Q12: What is the purpose of the static keyword? When should you use static members?**

Static members belong to the class rather than instances, shared across all objects. One copy exists per class, not per object.

Static variables store class-level state: configuration constants, counters, caches, singleton instances. Initialize in static blocks for complex setup.

Static methods perform operations not requiring instance data: utility functions, factory methods, singleton getters. Static methods can only access static members directly; accessing instance members requires an object reference.

Static initialization blocks execute once when the class loads, useful for complex static variable initialization, loading native libraries, or registering plugins.

```java
public class MathUtils {
    public static final double PI = 3.14159;  // Constant
    private static int callCount = 0;  // Shared counter
    
    static {
        // Static initialization
        System.loadLibrary("nativemath");
    }
    
    public static double square(double x) {
        callCount++;  // Track usage across all callers
        return x * x;
    }
}
```

Use static members for: mathematical utilities, factory methods, constants, singleton patterns. Avoid overusing static for testability—static members create global state difficult to mock in tests.

**Q13: Explain method overloading and overriding. What are the rules for each?**

Method overloading (compile-time polymorphism) allows multiple methods with the same name but different parameters in the same class. The compiler selects the method based on argument types and count. Overloading must differ in parameter list (type, number, or order); return type alone doesn't distinguish overloads.

```java
public int add(int a, int b) { return a + b; }
public double add(double a, double b) { return a + b; }
public String add(String a, String b) { return a + b; }
// Overloaded: different parameter types
```

Method overriding (runtime polymorphism) occurs when a subclass provides a specific implementation of an inherited method. The overriding method must have the same signature, compatible return type (covariant return types allowed), and cannot be more restrictive in access. The @Override annotation catches mistakes at compile time.

```java
class Animal {
    public void makeSound() { System.out.println("Generic sound"); }
}

class Dog extends Animal {
    @Override
    public void makeSound() { System.out.println("Bark"); }
}
```

Overriding rules: same method signature, equal or broader access modifier, covariant return types allowed, cannot override final methods, cannot throw broader checked exceptions. Overloading rules: different parameters, return type can differ, access modifiers can differ.

**Q14: What are nested classes? Explain the different types.**

Java supports four types of nested classes, each with different purposes and access rules:

Static nested classes associate with the outer class but don't require an outer instance. They can access outer class's static members only. Use for helper classes closely related to the outer class that don't need instance access.

Inner classes (non-static member classes) have an implicit reference to an outer instance, accessing all outer members including private. Each inner instance ties to a specific outer instance. Use for implementing helper objects that need access to outer state.

Local classes are defined inside methods, accessing method parameters and local variables (must be effectively final). Use for one-off implementations needed within a method.

Anonymous classes are unnamed classes defined and instantiated in one expression, typically implementing interfaces or extending classes inline. Common before lambdas for event handlers and callbacks.

```java
public class Outer {
    private int outerField;
    
    // Static nested
    public static class StaticNested { }
    
    // Inner class
    public class Inner {
        void accessOuter() { outerField = 10; }
    }
    
    public void method() {
        // Local class
        class Local { }
        
        // Anonymous class
        Runnable r = new Runnable() {
            public void run() { }
        };
    }
}
```

**Q15: Explain the concept of immutability. How do you create an immutable class?**

Immutable objects cannot change state after construction, providing thread safety, simplicity, and cache-ability. Immutable objects are inherently thread-safe, can be freely shared without defensive copying, and work well as map keys or set elements.

Create an immutable class by: declaring the class final to prevent subclassing, making all fields private and final, providing no setters, initializing all fields in the constructor, performing defensive copying of mutable parameters, and returning defensive copies of mutable fields.

```java
public final class ImmutablePerson {
    private final String name;
    private final int age;
    private final List<String> hobbies;
    
    public ImmutablePerson(String name, int age, List<String> hobbies) {
        this.name = name;
        this.age = age;
        // Defensive copy prevents external modification
        this.hobbies = new ArrayList<>(hobbies);
    }
    
    public String getName() { return name; }
    public int getAge() { return age; }
    
    public List<String> getHobbies() {
        // Return unmodifiable view
        return Collections.unmodifiableList(hobbies);
    }
    
    // Operations return new instances
    public ImmutablePerson withAge(int newAge) {
        return new ImmutablePerson(this.name, newAge, this.hobbies);
    }
}
```

Benefits: thread safety without synchronization, safe for use as keys, cacheable, easier reasoning about code. Trade-offs: memory allocation for each modification, not suitable for frequently changing data. Java's String, Integer, LocalDate are immutable.

---

These comprehensive notes cover Java fundamentals with production-grade depth suitable for senior developers, including real-world applications, advanced edge cases, common pitfalls, and interview preparation.
