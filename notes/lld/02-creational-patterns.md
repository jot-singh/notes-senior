# Creational Design Patterns

## Overview
Creational patterns deal with object creation mechanisms, trying to create objects in a manner suitable to the situation. They abstract the instantiation process, making the system independent of how objects are created, composed, and represented.

---

## Singleton Pattern

### Intent
Ensure a class has only one instance and provide a global point of access to it.

### When to Use
- Configuration managers
- Connection pools
- Caches
- Logger instances
- Thread pools

### Implementation Approaches

```java
// 1. Eager Initialization (Thread-safe, simple)
public class EagerSingleton {
    private static final EagerSingleton INSTANCE = new EagerSingleton();
    
    private EagerSingleton() {}
    
    public static EagerSingleton getInstance() {
        return INSTANCE;
    }
}

// 2. Lazy Initialization with Double-Checked Locking (Thread-safe, lazy)
public class LazySingleton {
    private static volatile LazySingleton instance;
    
    private LazySingleton() {}
    
    public static LazySingleton getInstance() {
        if (instance == null) {                    // First check (no locking)
            synchronized (LazySingleton.class) {
                if (instance == null) {            // Second check (with locking)
                    instance = new LazySingleton();
                }
            }
        }
        return instance;
    }
}

// 3. Bill Pugh Singleton (Thread-safe, lazy, recommended)
public class BillPughSingleton {
    private BillPughSingleton() {}
    
    private static class SingletonHelper {
        private static final BillPughSingleton INSTANCE = new BillPughSingleton();
    }
    
    public static BillPughSingleton getInstance() {
        return SingletonHelper.INSTANCE;
    }
}

// 4. Enum Singleton (Best - handles serialization & reflection)
public enum DatabaseConnection {
    INSTANCE;
    
    private Connection connection;
    
    DatabaseConnection() {
        // Initialize connection
    }
    
    public Connection getConnection() {
        return connection;
    }
    
    public void query(String sql) {
        // Execute query
    }
}
```

### Real-World Example: Configuration Manager

```java
public class ConfigurationManager {
    private static volatile ConfigurationManager instance;
    private final Properties properties;
    
    private ConfigurationManager() {
        properties = new Properties();
        loadConfiguration();
    }
    
    private void loadConfiguration() {
        try (InputStream input = getClass().getClassLoader()
                .getResourceAsStream("config.properties")) {
            if (input != null) {
                properties.load(input);
            }
        } catch (IOException e) {
            throw new RuntimeException("Failed to load configuration", e);
        }
    }
    
    public static ConfigurationManager getInstance() {
        if (instance == null) {
            synchronized (ConfigurationManager.class) {
                if (instance == null) {
                    instance = new ConfigurationManager();
                }
            }
        }
        return instance;
    }
    
    public String get(String key) {
        return properties.getProperty(key);
    }
    
    public String get(String key, String defaultValue) {
        return properties.getProperty(key, defaultValue);
    }
    
    public int getInt(String key, int defaultValue) {
        String value = properties.getProperty(key);
        return value != null ? Integer.parseInt(value) : defaultValue;
    }
}

// Usage
String dbUrl = ConfigurationManager.getInstance().get("database.url");
int maxConnections = ConfigurationManager.getInstance().getInt("pool.maxSize", 10);
```

### Pitfalls
- **Testing difficulty**: Hard to mock, use dependency injection instead
- **Hidden dependencies**: Makes code less transparent
- **Global state**: Can cause issues in concurrent environments

---

## Factory Method Pattern

### Intent
Define an interface for creating an object, but let subclasses decide which class to instantiate.

### When to Use
- When a class can't anticipate the type of objects it needs to create
- When subclasses should specify the objects they create
- When you want to localize object creation logic

### Structure

```
┌─────────────────────────────────────────────────────────────────┐
│                     FACTORY METHOD PATTERN                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────────────┐         ┌─────────────────────┐        │
│  │     <<interface>>   │         │     <<interface>>   │        │
│  │       Creator       │         │       Product       │        │
│  ├─────────────────────┤         ├─────────────────────┤        │
│  │ + factoryMethod()   │────────>│ + operation()       │        │
│  │ + someOperation()   │         └─────────────────────┘        │
│  └─────────────────────┘                   △                    │
│            △                               │                    │
│            │                    ┌──────────┴──────────┐         │
│  ┌─────────┴─────────┐          │                     │         │
│  │                   │   ┌──────────────┐   ┌──────────────┐    │
│  │ ConcreteCreatorA  │   │ConcreteProductA│ │ConcreteProductB│   │
│  │ ConcreteCreatorB  │   └──────────────┘   └──────────────┘    │
│  └───────────────────┘                                          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Implementation

```java
// Product interface
public interface Notification {
    void send(String message);
    String getType();
}

// Concrete products
public class EmailNotification implements Notification {
    private final String email;
    
    public EmailNotification(String email) {
        this.email = email;
    }
    
    @Override
    public void send(String message) {
        System.out.println("Sending email to " + email + ": " + message);
    }
    
    @Override
    public String getType() {
        return "EMAIL";
    }
}

public class SMSNotification implements Notification {
    private final String phoneNumber;
    
    public SMSNotification(String phoneNumber) {
        this.phoneNumber = phoneNumber;
    }
    
    @Override
    public void send(String message) {
        System.out.println("Sending SMS to " + phoneNumber + ": " + message);
    }
    
    @Override
    public String getType() {
        return "SMS";
    }
}

public class PushNotification implements Notification {
    private final String deviceToken;
    
    public PushNotification(String deviceToken) {
        this.deviceToken = deviceToken;
    }
    
    @Override
    public void send(String message) {
        System.out.println("Sending push to " + deviceToken + ": " + message);
    }
    
    @Override
    public String getType() {
        return "PUSH";
    }
}

// Factory
public class NotificationFactory {
    
    public static Notification create(String type, String destination) {
        return switch (type.toUpperCase()) {
            case "EMAIL" -> new EmailNotification(destination);
            case "SMS" -> new SMSNotification(destination);
            case "PUSH" -> new PushNotification(destination);
            default -> throw new IllegalArgumentException("Unknown notification type: " + type);
        };
    }
}

// Usage
Notification notification = NotificationFactory.create("EMAIL", "user@example.com");
notification.send("Welcome to our platform!");
```

### Real-World Example: Document Parser Factory

```java
public interface DocumentParser {
    Document parse(InputStream input);
    boolean supports(String fileExtension);
}

public class PDFParser implements DocumentParser {
    @Override
    public Document parse(InputStream input) {
        // PDF parsing logic
        return new Document("PDF content");
    }
    
    @Override
    public boolean supports(String fileExtension) {
        return "pdf".equalsIgnoreCase(fileExtension);
    }
}

public class WordParser implements DocumentParser {
    @Override
    public Document parse(InputStream input) {
        // Word parsing logic
        return new Document("Word content");
    }
    
    @Override
    public boolean supports(String fileExtension) {
        return "docx".equalsIgnoreCase(fileExtension) || 
               "doc".equalsIgnoreCase(fileExtension);
    }
}

public class DocumentParserFactory {
    private final List<DocumentParser> parsers;
    
    public DocumentParserFactory() {
        this.parsers = List.of(
            new PDFParser(),
            new WordParser(),
            new ExcelParser()
        );
    }
    
    public DocumentParser getParser(String filename) {
        String extension = getFileExtension(filename);
        return parsers.stream()
            .filter(p -> p.supports(extension))
            .findFirst()
            .orElseThrow(() -> new UnsupportedOperationException(
                "No parser found for: " + extension));
    }
    
    private String getFileExtension(String filename) {
        int lastDot = filename.lastIndexOf('.');
        return lastDot > 0 ? filename.substring(lastDot + 1) : "";
    }
}
```

---

## Abstract Factory Pattern

### Intent
Provide an interface for creating families of related objects without specifying their concrete classes.

### When to Use
- System should be independent of how products are created
- System should be configured with one of multiple families of products
- Products are designed to work together (need to enforce this)

### Structure

```
┌─────────────────────────────────────────────────────────────────┐
│                    ABSTRACT FACTORY PATTERN                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────────────────────────────────────────┐            │
│  │            <<interface>>                         │            │
│  │            UIFactory                             │            │
│  ├─────────────────────────────────────────────────┤            │
│  │ + createButton(): Button                         │            │
│  │ + createTextField(): TextField                   │            │
│  │ + createCheckbox(): Checkbox                     │            │
│  └─────────────────────────────────────────────────┘            │
│                         △                                        │
│           ┌─────────────┴─────────────┐                         │
│           │                           │                         │
│  ┌─────────────────┐         ┌─────────────────┐                │
│  │  WindowsFactory │         │   MacFactory    │                │
│  ├─────────────────┤         ├─────────────────┤                │
│  │ + createButton()│         │ + createButton()│                │
│  │ + createTextField()       │ + createTextField()              │
│  │ + createCheckbox()        │ + createCheckbox()               │
│  └─────────────────┘         └─────────────────┘                │
│           │                           │                         │
│           ▼                           ▼                         │
│  Creates Windows UI           Creates Mac UI                    │
│  Components                   Components                        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Implementation

```java
// Abstract products
public interface Button {
    void render();
    void onClick(Runnable action);
}

public interface TextField {
    void render();
    String getValue();
    void setValue(String value);
}

public interface Checkbox {
    void render();
    boolean isChecked();
}

// Concrete products - Windows
public class WindowsButton implements Button {
    @Override
    public void render() {
        System.out.println("Rendering Windows style button");
    }
    
    @Override
    public void onClick(Runnable action) {
        System.out.println("Windows button clicked");
        action.run();
    }
}

public class WindowsTextField implements TextField {
    private String value = "";
    
    @Override
    public void render() {
        System.out.println("Rendering Windows style text field");
    }
    
    @Override
    public String getValue() { return value; }
    
    @Override
    public void setValue(String value) { this.value = value; }
}

// Concrete products - Mac
public class MacButton implements Button {
    @Override
    public void render() {
        System.out.println("Rendering Mac style button");
    }
    
    @Override
    public void onClick(Runnable action) {
        System.out.println("Mac button clicked");
        action.run();
    }
}

public class MacTextField implements TextField {
    private String value = "";
    
    @Override
    public void render() {
        System.out.println("Rendering Mac style text field");
    }
    
    @Override
    public String getValue() { return value; }
    
    @Override
    public void setValue(String value) { this.value = value; }
}

// Abstract factory
public interface UIFactory {
    Button createButton();
    TextField createTextField();
    Checkbox createCheckbox();
}

// Concrete factories
public class WindowsUIFactory implements UIFactory {
    @Override
    public Button createButton() { return new WindowsButton(); }
    
    @Override
    public TextField createTextField() { return new WindowsTextField(); }
    
    @Override
    public Checkbox createCheckbox() { return new WindowsCheckbox(); }
}

public class MacUIFactory implements UIFactory {
    @Override
    public Button createButton() { return new MacButton(); }
    
    @Override
    public TextField createTextField() { return new MacTextField(); }
    
    @Override
    public Checkbox createCheckbox() { return new MacCheckbox(); }
}

// Client code
public class Application {
    private final Button button;
    private final TextField textField;
    
    public Application(UIFactory factory) {
        this.button = factory.createButton();
        this.textField = factory.createTextField();
    }
    
    public void render() {
        button.render();
        textField.render();
    }
}

// Usage
UIFactory factory = System.getProperty("os.name").contains("Windows") 
    ? new WindowsUIFactory() 
    : new MacUIFactory();
    
Application app = new Application(factory);
app.render();
```

---

## Builder Pattern

### Intent
Separate the construction of a complex object from its representation, allowing the same construction process to create different representations.

### When to Use
- Object has many optional parameters
- Object creation involves many steps
- You want immutable objects with many fields
- You want to avoid telescoping constructors

### Implementation

```java
// Product
public class HttpRequest {
    private final String url;
    private final String method;
    private final Map<String, String> headers;
    private final Map<String, String> queryParams;
    private final String body;
    private final Duration timeout;
    private final int retryCount;
    
    private HttpRequest(Builder builder) {
        this.url = builder.url;
        this.method = builder.method;
        this.headers = Map.copyOf(builder.headers);
        this.queryParams = Map.copyOf(builder.queryParams);
        this.body = builder.body;
        this.timeout = builder.timeout;
        this.retryCount = builder.retryCount;
    }
    
    // Getters
    public String getUrl() { return url; }
    public String getMethod() { return method; }
    public Map<String, String> getHeaders() { return headers; }
    public String getBody() { return body; }
    
    public static class Builder {
        // Required parameters
        private final String url;
        
        // Optional parameters with defaults
        private String method = "GET";
        private Map<String, String> headers = new HashMap<>();
        private Map<String, String> queryParams = new HashMap<>();
        private String body;
        private Duration timeout = Duration.ofSeconds(30);
        private int retryCount = 3;
        
        public Builder(String url) {
            this.url = Objects.requireNonNull(url, "URL cannot be null");
        }
        
        public Builder method(String method) {
            this.method = method;
            return this;
        }
        
        public Builder header(String key, String value) {
            this.headers.put(key, value);
            return this;
        }
        
        public Builder headers(Map<String, String> headers) {
            this.headers.putAll(headers);
            return this;
        }
        
        public Builder queryParam(String key, String value) {
            this.queryParams.put(key, value);
            return this;
        }
        
        public Builder body(String body) {
            this.body = body;
            return this;
        }
        
        public Builder timeout(Duration timeout) {
            this.timeout = timeout;
            return this;
        }
        
        public Builder retryCount(int retryCount) {
            this.retryCount = retryCount;
            return this;
        }
        
        public HttpRequest build() {
            validate();
            return new HttpRequest(this);
        }
        
        private void validate() {
            if (("POST".equals(method) || "PUT".equals(method)) && body == null) {
                throw new IllegalStateException("Body required for " + method + " requests");
            }
        }
    }
}

// Usage
HttpRequest request = new HttpRequest.Builder("https://api.example.com/users")
    .method("POST")
    .header("Content-Type", "application/json")
    .header("Authorization", "Bearer token123")
    .body("{\"name\": \"John\", \"email\": \"john@example.com\"}")
    .timeout(Duration.ofSeconds(10))
    .retryCount(5)
    .build();
```

### Real-World Example: SQL Query Builder

```java
public class SqlQueryBuilder {
    private final StringBuilder query;
    private final List<Object> parameters;
    private String table;
    private final List<String> selectColumns = new ArrayList<>();
    private final List<String> whereConditions = new ArrayList<>();
    private final List<String> orderByColumns = new ArrayList<>();
    private Integer limit;
    private Integer offset;
    
    public SqlQueryBuilder() {
        this.query = new StringBuilder();
        this.parameters = new ArrayList<>();
    }
    
    public SqlQueryBuilder select(String... columns) {
        selectColumns.addAll(Arrays.asList(columns));
        return this;
    }
    
    public SqlQueryBuilder from(String table) {
        this.table = table;
        return this;
    }
    
    public SqlQueryBuilder where(String condition, Object... params) {
        whereConditions.add(condition);
        parameters.addAll(Arrays.asList(params));
        return this;
    }
    
    public SqlQueryBuilder and(String condition, Object... params) {
        return where(condition, params);
    }
    
    public SqlQueryBuilder orderBy(String column, String direction) {
        orderByColumns.add(column + " " + direction);
        return this;
    }
    
    public SqlQueryBuilder limit(int limit) {
        this.limit = limit;
        return this;
    }
    
    public SqlQueryBuilder offset(int offset) {
        this.offset = offset;
        return this;
    }
    
    public String build() {
        query.append("SELECT ");
        query.append(selectColumns.isEmpty() ? "*" : String.join(", ", selectColumns));
        query.append(" FROM ").append(table);
        
        if (!whereConditions.isEmpty()) {
            query.append(" WHERE ").append(String.join(" AND ", whereConditions));
        }
        
        if (!orderByColumns.isEmpty()) {
            query.append(" ORDER BY ").append(String.join(", ", orderByColumns));
        }
        
        if (limit != null) {
            query.append(" LIMIT ").append(limit);
        }
        
        if (offset != null) {
            query.append(" OFFSET ").append(offset);
        }
        
        return query.toString();
    }
    
    public List<Object> getParameters() {
        return Collections.unmodifiableList(parameters);
    }
}

// Usage
String query = new SqlQueryBuilder()
    .select("id", "name", "email", "created_at")
    .from("users")
    .where("status = ?", "active")
    .and("created_at > ?", LocalDate.now().minusDays(30))
    .orderBy("created_at", "DESC")
    .limit(10)
    .offset(20)
    .build();
// Result: SELECT id, name, email, created_at FROM users WHERE status = ? AND created_at > ? ORDER BY created_at DESC LIMIT 10 OFFSET 20
```

---

## Prototype Pattern

### Intent
Create new objects by copying an existing object (prototype) rather than creating from scratch.

### When to Use
- Object creation is expensive (database calls, network requests)
- System should be independent of how products are created
- Need many similar objects with slight variations

### Implementation

```java
// Prototype interface
public interface Prototype<T> {
    T clone();
}

// Concrete prototype
public class GameCharacter implements Prototype<GameCharacter> {
    private String name;
    private int health;
    private int attack;
    private int defense;
    private List<String> abilities;
    private Map<String, Integer> stats;
    
    public GameCharacter(String name, int health, int attack, int defense) {
        this.name = name;
        this.health = health;
        this.attack = attack;
        this.defense = defense;
        this.abilities = new ArrayList<>();
        this.stats = new HashMap<>();
    }
    
    // Copy constructor for deep cloning
    private GameCharacter(GameCharacter source) {
        this.name = source.name;
        this.health = source.health;
        this.attack = source.attack;
        this.defense = source.defense;
        this.abilities = new ArrayList<>(source.abilities);
        this.stats = new HashMap<>(source.stats);
    }
    
    @Override
    public GameCharacter clone() {
        return new GameCharacter(this);
    }
    
    // Setters for customization after cloning
    public GameCharacter withName(String name) {
        this.name = name;
        return this;
    }
    
    public GameCharacter withHealth(int health) {
        this.health = health;
        return this;
    }
    
    public void addAbility(String ability) {
        abilities.add(ability);
    }
}

// Prototype registry
public class CharacterRegistry {
    private final Map<String, GameCharacter> prototypes = new HashMap<>();
    
    public CharacterRegistry() {
        // Pre-load prototypes
        GameCharacter warrior = new GameCharacter("Warrior", 100, 15, 10);
        warrior.addAbility("Slash");
        warrior.addAbility("Block");
        prototypes.put("warrior", warrior);
        
        GameCharacter mage = new GameCharacter("Mage", 60, 25, 5);
        mage.addAbility("Fireball");
        mage.addAbility("Ice Shield");
        prototypes.put("mage", mage);
        
        GameCharacter archer = new GameCharacter("Archer", 80, 20, 7);
        archer.addAbility("Arrow Shot");
        archer.addAbility("Dodge");
        prototypes.put("archer", archer);
    }
    
    public GameCharacter create(String type) {
        GameCharacter prototype = prototypes.get(type.toLowerCase());
        if (prototype == null) {
            throw new IllegalArgumentException("Unknown character type: " + type);
        }
        return prototype.clone();
    }
    
    public void register(String type, GameCharacter prototype) {
        prototypes.put(type.toLowerCase(), prototype);
    }
}

// Usage
CharacterRegistry registry = new CharacterRegistry();

GameCharacter player1 = registry.create("warrior").withName("Hero");
GameCharacter player2 = registry.create("mage").withName("Merlin");
GameCharacter enemy1 = registry.create("warrior").withName("Orc").withHealth(50);
```

---

## Comparison Table

| Pattern | Intent | Use When |
|---------|--------|----------|
| **Singleton** | One instance globally | Shared resources (config, pools) |
| **Factory Method** | Defer instantiation to subclasses | Unknown types at compile time |
| **Abstract Factory** | Create families of related objects | Multiple product families |
| **Builder** | Step-by-step complex construction | Many optional parameters |
| **Prototype** | Clone existing objects | Expensive object creation |

---

## Quick Reference

```java
// Singleton
MySingleton.getInstance().doSomething();

// Factory Method
Product p = factory.createProduct(type);

// Abstract Factory
UIFactory factory = new WindowsUIFactory();
Button btn = factory.createButton();

// Builder
Request req = new Request.Builder(url).method("POST").build();

// Prototype
GameCharacter clone = prototype.clone();
```

---

## Related Topics
- [[01-lld-fundamentals]] - SOLID principles
- [[03-structural-patterns]] - Adapter, Decorator, Facade
- [[04-behavioral-patterns]] - Strategy, Observer, Command
