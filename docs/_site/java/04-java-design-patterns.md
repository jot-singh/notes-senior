# Java Design Patterns

## Concept Overview

Design patterns are reusable solutions to common software design problems. They provide a shared vocabulary for developers, promote best practices, and help create maintainable, flexible code. Patterns fall into three categories: Creational (object creation), Structural (composition), and Behavioral (object interaction).

## Core Patterns & Implementation

### Creational Patterns

#### Singleton

Ensures single instance with global access. Use for configuration, connection pools, caches.

```java
// Thread-safe lazy initialization (recommended)
public class Singleton {
    private static volatile Singleton instance;
    
    private Singleton() {}
    
    public static Singleton getInstance() {
        if (instance == null) {
            synchronized (Singleton.class) {
                if (instance == null) {
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
}

// Enum singleton (safest, handles serialization)
public enum DatabaseConnection {
    INSTANCE;
    
    public void query(String sql) { /* ... */ }
}
```

#### Factory Method

Defines interface for creating objects, subclasses decide which class to instantiate.

```java
public interface Notification {
    void send(String message);
}

public class EmailNotification implements Notification {
    public void send(String message) { /* email logic */ }
}

public class SMSNotification implements Notification {
    public void send(String message) { /* SMS logic */ }
}

public class NotificationFactory {
    public static Notification create(String type) {
        return switch (type) {
            case "EMAIL" -> new EmailNotification();
            case "SMS" -> new SMSNotification();
            default -> throw new IllegalArgumentException("Unknown type: " + type);
        };
    }
}
```

#### Builder

Constructs complex objects step by step. Ideal for objects with many optional parameters.

```java
public class HttpRequest {
    private final String url;
    private final String method;
    private final Map<String, String> headers;
    private final String body;
    
    private HttpRequest(Builder builder) {
        this.url = builder.url;
        this.method = builder.method;
        this.headers = Map.copyOf(builder.headers);
        this.body = builder.body;
    }
    
    public static class Builder {
        private final String url;
        private String method = "GET";
        private Map<String, String> headers = new HashMap<>();
        private String body;
        
        public Builder(String url) { this.url = url; }
        public Builder method(String method) { this.method = method; return this; }
        public Builder header(String key, String value) { headers.put(key, value); return this; }
        public Builder body(String body) { this.body = body; return this; }
        public HttpRequest build() { return new HttpRequest(this); }
    }
}

// Usage
HttpRequest request = new HttpRequest.Builder("https://api.example.com")
    .method("POST")
    .header("Content-Type", "application/json")
    .body("{\"name\":\"John\"}")
    .build();
```

### Structural Patterns

#### Adapter

Converts interface of a class to one clients expect.

```java
// Legacy payment system
public class LegacyPayment {
    public void processPayment(String cardNumber, double amount) { /* ... */ }
}

// New interface
public interface PaymentGateway {
    void pay(PaymentRequest request);
}

// Adapter
public class PaymentAdapter implements PaymentGateway {
    private final LegacyPayment legacy;
    
    public PaymentAdapter(LegacyPayment legacy) { this.legacy = legacy; }
    
    @Override
    public void pay(PaymentRequest request) {
        legacy.processPayment(request.getCardNumber(), request.getAmount());
    }
}
```

#### Decorator

Adds behavior to objects dynamically without altering their structure.

```java
public interface Coffee {
    double getCost();
    String getDescription();
}

public class SimpleCoffee implements Coffee {
    public double getCost() { return 2.0; }
    public String getDescription() { return "Coffee"; }
}

public abstract class CoffeeDecorator implements Coffee {
    protected final Coffee coffee;
    public CoffeeDecorator(Coffee coffee) { this.coffee = coffee; }
}

public class MilkDecorator extends CoffeeDecorator {
    public MilkDecorator(Coffee coffee) { super(coffee); }
    public double getCost() { return coffee.getCost() + 0.5; }
    public String getDescription() { return coffee.getDescription() + ", Milk"; }
}

// Usage: Coffee order = new MilkDecorator(new SimpleCoffee());
```

#### Proxy

Provides surrogate to control access to another object.

```java
public interface ImageService {
    byte[] loadImage(String id);
}

public class ImageServiceImpl implements ImageService {
    public byte[] loadImage(String id) { /* expensive operation */ }
}

public class CachingImageProxy implements ImageService {
    private final ImageService service;
    private final Map<String, byte[]> cache = new ConcurrentHashMap<>();
    
    public CachingImageProxy(ImageService service) { this.service = service; }
    
    @Override
    public byte[] loadImage(String id) {
        return cache.computeIfAbsent(id, service::loadImage);
    }
}
```

### Behavioral Patterns

#### Strategy

Defines family of algorithms, encapsulates each, makes them interchangeable.

```java
public interface PricingStrategy {
    double calculatePrice(double basePrice);
}

public class RegularPricing implements PricingStrategy {
    public double calculatePrice(double basePrice) { return basePrice; }
}

public class PremiumDiscount implements PricingStrategy {
    public double calculatePrice(double basePrice) { return basePrice * 0.8; }
}

public class ShoppingCart {
    private PricingStrategy strategy;
    
    public void setStrategy(PricingStrategy strategy) { this.strategy = strategy; }
    
    public double checkout(double basePrice) {
        return strategy.calculatePrice(basePrice);
    }
}
```

#### Observer

Defines one-to-many dependency; when one object changes, dependents are notified.

```java
public interface EventListener {
    void onEvent(String event);
}

public class EventManager {
    private final Map<String, List<EventListener>> listeners = new HashMap<>();
    
    public void subscribe(String eventType, EventListener listener) {
        listeners.computeIfAbsent(eventType, k -> new ArrayList<>()).add(listener);
    }
    
    public void notify(String eventType, String data) {
        listeners.getOrDefault(eventType, Collections.emptyList())
            .forEach(l -> l.onEvent(data));
    }
}
```

#### Template Method

Defines skeleton of algorithm, letting subclasses override specific steps.

```java
public abstract class DataProcessor {
    // Template method
    public final void process() {
        readData();
        processData();
        writeData();
    }
    
    protected abstract void readData();
    protected abstract void processData();
    protected void writeData() { /* default implementation */ }
}

public class CSVProcessor extends DataProcessor {
    protected void readData() { /* read CSV */ }
    protected void processData() { /* process CSV */ }
}
```

## Best Practices

- **Favor composition over inheritance**: Use Strategy, Decorator instead of deep hierarchies
- **Program to interfaces**: Depend on abstractions, not concrete implementations
- **SOLID principles**: Patterns often implement SOLID (Strategy = OCP, Factory = DIP)
- **Don't over-engineer**: Apply patterns when they solve actual problems
- **Document pattern usage**: Help future developers understand design decisions

## Interview Questions

**Q1: When would you use Factory vs Builder pattern?**

Factory creates objects of a type based on parameters—use when creation logic is complex or type varies at runtime. Builder constructs complex objects step-by-step—use when objects have many optional parameters or require validation before construction.

**Q2: Explain the Singleton pattern pitfalls.**

Global state complicates testing, creates hidden dependencies. Thread safety issues with lazy initialization. Serialization can create multiple instances. Difficult to subclass. Prefer dependency injection over singletons for better testability.

**Q3: How does Decorator differ from inheritance?**

Decorator adds behavior at runtime, can combine multiple decorators, follows OCP. Inheritance adds behavior at compile-time, creates rigid hierarchies, violates LSP if overused. Decorator = "has-a" relationship; inheritance = "is-a" relationship.

**Q4: What is the Strategy pattern?**

Encapsulates interchangeable algorithms/behaviors behind common interface. Context delegates to strategy object. Enables runtime algorithm switching, eliminates conditionals, follows OCP. Examples: sorting algorithms, payment methods, compression strategies.

**Q5: Explain Observer pattern implementation.**

Subject maintains list of observers, notifies all when state changes. Observers implement common interface with update method. Enables loose coupling—subject doesn't know concrete observer types. Used in event systems, MVC, reactive programming.

**Q6: When to use Proxy pattern?**

Lazy loading (virtual proxy), access control (protection proxy), caching (smart proxy), logging/monitoring (logging proxy), remote objects (remote proxy). Controls access to real object without clients knowing.

**Q7: What is Template Method pattern?**

Defines algorithm skeleton in base class, subclasses override specific steps. Promotes code reuse, follows Hollywood Principle ("don't call us, we'll call you"). Common in frameworks—framework calls your code at specific points.

**Q8: How to implement thread-safe Singleton?**

Double-checked locking with volatile, enum singleton (safest), or static inner class holder. Enum handles serialization automatically. Avoid synchronized getInstance() as it's slow. Consider if singleton is actually needed—dependency injection often better.
