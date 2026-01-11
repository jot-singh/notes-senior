# Structural Design Patterns

## Overview
Structural patterns deal with object composition, creating relationships between objects to form larger structures while keeping them flexible and efficient. They help ensure that if one part of a system changes, the entire structure doesn't need to change.

---

## Adapter Pattern

### Intent
Convert the interface of a class into another interface clients expect. Adapter lets classes work together that couldn't otherwise because of incompatible interfaces.

### When to Use
- Integrating legacy code with new systems
- Working with third-party libraries that have different interfaces
- Creating reusable classes that work with unrelated classes

### Structure

```
┌─────────────────────────────────────────────────────────────────┐
│                       ADAPTER PATTERN                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌────────────┐         ┌──────────────┐                        │
│  │   Client   │────────>│   Target     │                        │
│  └────────────┘         │  Interface   │                        │
│                         └──────────────┘                        │
│                                △                                 │
│                                │                                 │
│                         ┌──────────────┐      ┌──────────────┐  │
│                         │   Adapter    │─────>│   Adaptee    │  │
│                         │              │      │  (Legacy)    │  │
│                         └──────────────┘      └──────────────┘  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Implementation

```java
// Target interface (what client expects)
public interface PaymentProcessor {
    PaymentResult process(PaymentRequest request);
    PaymentStatus checkStatus(String transactionId);
}

// Adaptee (legacy system with different interface)
public class LegacyPaymentGateway {
    public String makePayment(String cardNumber, String expiry, 
                              String cvv, double amount, String currency) {
        // Legacy payment logic
        return "TXN-" + System.currentTimeMillis();
    }
    
    public int getPaymentStatus(String txnId) {
        // Returns: 0=pending, 1=success, 2=failed
        return 1;
    }
}

// Request/Response classes
public record PaymentRequest(
    String cardNumber,
    String expiryDate,
    String cvv,
    BigDecimal amount,
    Currency currency
) {}

public record PaymentResult(
    String transactionId,
    boolean success,
    String message
) {}

public enum PaymentStatus { PENDING, SUCCESS, FAILED }

// Adapter
public class LegacyPaymentAdapter implements PaymentProcessor {
    private final LegacyPaymentGateway legacyGateway;
    
    public LegacyPaymentAdapter(LegacyPaymentGateway legacyGateway) {
        this.legacyGateway = legacyGateway;
    }
    
    @Override
    public PaymentResult process(PaymentRequest request) {
        try {
            String txnId = legacyGateway.makePayment(
                request.cardNumber(),
                request.expiryDate(),
                request.cvv(),
                request.amount().doubleValue(),
                request.currency().getCurrencyCode()
            );
            return new PaymentResult(txnId, true, "Payment processed successfully");
        } catch (Exception e) {
            return new PaymentResult(null, false, "Payment failed: " + e.getMessage());
        }
    }
    
    @Override
    public PaymentStatus checkStatus(String transactionId) {
        int status = legacyGateway.getPaymentStatus(transactionId);
        return switch (status) {
            case 0 -> PaymentStatus.PENDING;
            case 1 -> PaymentStatus.SUCCESS;
            default -> PaymentStatus.FAILED;
        };
    }
}

// Usage
PaymentProcessor processor = new LegacyPaymentAdapter(new LegacyPaymentGateway());
PaymentResult result = processor.process(new PaymentRequest(
    "4111111111111111", "12/25", "123", 
    new BigDecimal("99.99"), Currency.getInstance("USD")
));
```

### Real-World Example: Database Adapter

```java
// Target interface
public interface UserRepository {
    User findById(Long id);
    List<User> findByEmail(String email);
    User save(User user);
    void delete(Long id);
}

// Adaptee - External API
public class ExternalUserService {
    public JsonObject getUserById(String id) { /* returns JSON */ }
    public JsonArray searchUsers(Map<String, String> criteria) { /* returns JSON array */ }
    public JsonObject createUser(JsonObject userData) { /* creates and returns JSON */ }
    public void removeUser(String id) { /* deletes user */ }
}

// Adapter
public class ExternalUserAdapter implements UserRepository {
    private final ExternalUserService externalService;
    private final ObjectMapper mapper;
    
    public ExternalUserAdapter(ExternalUserService externalService) {
        this.externalService = externalService;
        this.mapper = new ObjectMapper();
    }
    
    @Override
    public User findById(Long id) {
        JsonObject json = externalService.getUserById(id.toString());
        return mapToUser(json);
    }
    
    @Override
    public List<User> findByEmail(String email) {
        JsonArray results = externalService.searchUsers(Map.of("email", email));
        return results.stream()
            .map(this::mapToUser)
            .collect(Collectors.toList());
    }
    
    @Override
    public User save(User user) {
        JsonObject userData = mapToJson(user);
        JsonObject result = externalService.createUser(userData);
        return mapToUser(result);
    }
    
    @Override
    public void delete(Long id) {
        externalService.removeUser(id.toString());
    }
    
    private User mapToUser(JsonObject json) {
        return new User(
            Long.parseLong(json.getString("id")),
            json.getString("name"),
            json.getString("email")
        );
    }
    
    private JsonObject mapToJson(User user) {
        // Convert User to JsonObject
    }
}
```

---

## Decorator Pattern

### Intent
Attach additional responsibilities to an object dynamically. Decorators provide a flexible alternative to subclassing for extending functionality.

### When to Use
- Add responsibilities to individual objects without affecting other objects
- Need to add/remove responsibilities at runtime
- Subclassing is impractical (too many combinations)

### Structure

```
┌─────────────────────────────────────────────────────────────────┐
│                      DECORATOR PATTERN                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │                     <<interface>>                           │ │
│  │                       Component                             │ │
│  │                     + operation()                           │ │
│  └────────────────────────────────────────────────────────────┘ │
│                              △                                   │
│               ┌──────────────┴──────────────┐                   │
│               │                             │                   │
│  ┌────────────────────┐        ┌────────────────────┐          │
│  │ ConcreteComponent  │        │     Decorator      │          │
│  │                    │        │ - component        │          │
│  │ + operation()      │        │ + operation()      │          │
│  └────────────────────┘        └────────────────────┘          │
│                                         △                       │
│                          ┌──────────────┴──────────────┐        │
│                          │                             │        │
│               ┌────────────────────┐     ┌────────────────────┐ │
│               │  DecoratorA        │     │    DecoratorB      │ │
│               │  + operation()     │     │    + operation()   │ │
│               └────────────────────┘     └────────────────────┘ │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Implementation

```java
// Component interface
public interface DataSource {
    void writeData(String data);
    String readData();
}

// Concrete component
public class FileDataSource implements DataSource {
    private final String filename;
    
    public FileDataSource(String filename) {
        this.filename = filename;
    }
    
    @Override
    public void writeData(String data) {
        try (FileWriter writer = new FileWriter(filename)) {
            writer.write(data);
        } catch (IOException e) {
            throw new RuntimeException("Failed to write file", e);
        }
    }
    
    @Override
    public String readData() {
        try {
            return Files.readString(Path.of(filename));
        } catch (IOException e) {
            throw new RuntimeException("Failed to read file", e);
        }
    }
}

// Base decorator
public abstract class DataSourceDecorator implements DataSource {
    protected final DataSource wrappee;
    
    protected DataSourceDecorator(DataSource source) {
        this.wrappee = source;
    }
    
    @Override
    public void writeData(String data) {
        wrappee.writeData(data);
    }
    
    @Override
    public String readData() {
        return wrappee.readData();
    }
}

// Concrete decorators
public class EncryptionDecorator extends DataSourceDecorator {
    private final String secretKey;
    
    public EncryptionDecorator(DataSource source, String secretKey) {
        super(source);
        this.secretKey = secretKey;
    }
    
    @Override
    public void writeData(String data) {
        String encrypted = encrypt(data);
        super.writeData(encrypted);
    }
    
    @Override
    public String readData() {
        String data = super.readData();
        return decrypt(data);
    }
    
    private String encrypt(String data) {
        // AES encryption logic
        return Base64.getEncoder().encodeToString(data.getBytes());
    }
    
    private String decrypt(String data) {
        // AES decryption logic
        return new String(Base64.getDecoder().decode(data));
    }
}

public class CompressionDecorator extends DataSourceDecorator {
    
    public CompressionDecorator(DataSource source) {
        super(source);
    }
    
    @Override
    public void writeData(String data) {
        String compressed = compress(data);
        super.writeData(compressed);
    }
    
    @Override
    public String readData() {
        String data = super.readData();
        return decompress(data);
    }
    
    private String compress(String data) {
        // GZIP compression
        try (ByteArrayOutputStream bos = new ByteArrayOutputStream();
             GZIPOutputStream gzip = new GZIPOutputStream(bos)) {
            gzip.write(data.getBytes());
            gzip.close();
            return Base64.getEncoder().encodeToString(bos.toByteArray());
        } catch (IOException e) {
            throw new RuntimeException("Compression failed", e);
        }
    }
    
    private String decompress(String data) {
        // GZIP decompression
        byte[] compressed = Base64.getDecoder().decode(data);
        try (GZIPInputStream gzip = new GZIPInputStream(
                new ByteArrayInputStream(compressed))) {
            return new String(gzip.readAllBytes());
        } catch (IOException e) {
            throw new RuntimeException("Decompression failed", e);
        }
    }
}

public class LoggingDecorator extends DataSourceDecorator {
    private static final Logger log = LoggerFactory.getLogger(LoggingDecorator.class);
    
    public LoggingDecorator(DataSource source) {
        super(source);
    }
    
    @Override
    public void writeData(String data) {
        log.info("Writing data: {} bytes", data.length());
        long start = System.currentTimeMillis();
        super.writeData(data);
        log.info("Write completed in {} ms", System.currentTimeMillis() - start);
    }
    
    @Override
    public String readData() {
        log.info("Reading data...");
        long start = System.currentTimeMillis();
        String data = super.readData();
        log.info("Read {} bytes in {} ms", data.length(), System.currentTimeMillis() - start);
        return data;
    }
}

// Usage - decorators can be stacked
DataSource source = new FileDataSource("data.txt");
source = new CompressionDecorator(source);
source = new EncryptionDecorator(source, "secret-key");
source = new LoggingDecorator(source);

// Write: logging -> encryption -> compression -> file
source.writeData("sensitive user data");

// Read: file -> decompression -> decryption -> logging
String data = source.readData();
```

### Java I/O Streams - Real Decorator Example

```java
// Java's I/O uses decorator pattern extensively
InputStream fileStream = new FileInputStream("data.txt");
InputStream bufferedStream = new BufferedInputStream(fileStream);
DataInputStream dataStream = new DataInputStream(bufferedStream);

// Or chained
DataInputStream in = new DataInputStream(
    new BufferedInputStream(
        new GZIPInputStream(
            new FileInputStream("data.gz")
        )
    )
);
```

---

## Facade Pattern

### Intent
Provide a unified interface to a set of interfaces in a subsystem. Facade defines a higher-level interface that makes the subsystem easier to use.

### When to Use
- Simplify complex subsystem
- Decouple client from subsystem components
- Layer your subsystems with entry points

### Structure

```
┌─────────────────────────────────────────────────────────────────┐
│                        FACADE PATTERN                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌────────────┐                                                  │
│  │   Client   │                                                  │
│  └─────┬──────┘                                                  │
│        │                                                         │
│        ▼                                                         │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │                         FACADE                               ││
│  │                  + simpleOperation()                         ││
│  └─────────────────────────────────────────────────────────────┘│
│        │                                                         │
│        ├────────────────┬────────────────┬──────────────────┐   │
│        ▼                ▼                ▼                  ▼   │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐  │
│  │Subsystem1│    │Subsystem2│    │Subsystem3│    │Subsystem4│  │
│  └──────────┘    └──────────┘    └──────────┘    └──────────┘  │
│                                                                  │
│                     COMPLEX SUBSYSTEM                            │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Implementation

```java
// Complex subsystem classes
public class InventoryService {
    public boolean checkStock(String productId, int quantity) {
        // Check warehouse inventory
        return true;
    }
    
    public void reserveStock(String productId, int quantity) {
        // Reserve items in inventory
    }
    
    public void releaseStock(String productId, int quantity) {
        // Release reserved items
    }
}

public class PaymentService {
    public String authorize(String cardNumber, BigDecimal amount) {
        // Authorize payment with bank
        return "AUTH-" + System.currentTimeMillis();
    }
    
    public boolean capture(String authorizationId, BigDecimal amount) {
        // Capture authorized payment
        return true;
    }
    
    public boolean refund(String transactionId, BigDecimal amount) {
        // Process refund
        return true;
    }
}

public class ShippingService {
    public BigDecimal calculateShipping(Address from, Address to, double weight) {
        // Calculate shipping cost
        return new BigDecimal("9.99");
    }
    
    public String createShipment(Order order, Address destination) {
        // Create shipping label
        return "TRACK-" + System.currentTimeMillis();
    }
}

public class NotificationService {
    public void sendOrderConfirmation(String email, Order order) {
        // Send confirmation email
    }
    
    public void sendShippingNotification(String email, String trackingNumber) {
        // Send shipping update
    }
}

public class OrderRepository {
    public Order save(Order order) {
        // Persist order
        return order;
    }
    
    public Order findById(String orderId) {
        // Fetch order
        return null;
    }
}

// Facade - simplifies complex order processing
public class OrderFacade {
    private final InventoryService inventoryService;
    private final PaymentService paymentService;
    private final ShippingService shippingService;
    private final NotificationService notificationService;
    private final OrderRepository orderRepository;
    
    public OrderFacade(InventoryService inventoryService,
                       PaymentService paymentService,
                       ShippingService shippingService,
                       NotificationService notificationService,
                       OrderRepository orderRepository) {
        this.inventoryService = inventoryService;
        this.paymentService = paymentService;
        this.shippingService = shippingService;
        this.notificationService = notificationService;
        this.orderRepository = orderRepository;
    }
    
    // Simple interface for complex operation
    public OrderResult placeOrder(OrderRequest request) {
        try {
            // 1. Validate inventory
            for (OrderItem item : request.getItems()) {
                if (!inventoryService.checkStock(item.getProductId(), item.getQuantity())) {
                    return OrderResult.failure("Product " + item.getProductId() + " out of stock");
                }
            }
            
            // 2. Reserve inventory
            for (OrderItem item : request.getItems()) {
                inventoryService.reserveStock(item.getProductId(), item.getQuantity());
            }
            
            // 3. Calculate total with shipping
            BigDecimal subtotal = calculateSubtotal(request.getItems());
            BigDecimal shipping = shippingService.calculateShipping(
                getWarehouseAddress(), 
                request.getShippingAddress(),
                calculateWeight(request.getItems())
            );
            BigDecimal total = subtotal.add(shipping);
            
            // 4. Process payment
            String authId = paymentService.authorize(request.getCardNumber(), total);
            if (authId == null) {
                releaseInventory(request.getItems());
                return OrderResult.failure("Payment authorization failed");
            }
            
            boolean captured = paymentService.capture(authId, total);
            if (!captured) {
                releaseInventory(request.getItems());
                return OrderResult.failure("Payment capture failed");
            }
            
            // 5. Create order
            Order order = Order.builder()
                .items(request.getItems())
                .shippingAddress(request.getShippingAddress())
                .total(total)
                .status(OrderStatus.CONFIRMED)
                .build();
            order = orderRepository.save(order);
            
            // 6. Create shipment
            String trackingNumber = shippingService.createShipment(
                order, 
                request.getShippingAddress()
            );
            
            // 7. Send notifications
            notificationService.sendOrderConfirmation(request.getEmail(), order);
            notificationService.sendShippingNotification(request.getEmail(), trackingNumber);
            
            return OrderResult.success(order.getId(), trackingNumber);
            
        } catch (Exception e) {
            // Handle rollback
            return OrderResult.failure("Order processing failed: " + e.getMessage());
        }
    }
    
    private void releaseInventory(List<OrderItem> items) {
        for (OrderItem item : items) {
            inventoryService.releaseStock(item.getProductId(), item.getQuantity());
        }
    }
    
    private BigDecimal calculateSubtotal(List<OrderItem> items) {
        return items.stream()
            .map(item -> item.getPrice().multiply(BigDecimal.valueOf(item.getQuantity())))
            .reduce(BigDecimal.ZERO, BigDecimal::add);
    }
}

// Client code - simple!
OrderFacade orderFacade = new OrderFacade(/*...dependencies...*/);

OrderResult result = orderFacade.placeOrder(new OrderRequest(
    customer.getEmail(),
    items,
    shippingAddress,
    cardNumber
));

if (result.isSuccess()) {
    System.out.println("Order placed: " + result.getOrderId());
}
```

---

## Proxy Pattern

### Intent
Provide a surrogate or placeholder for another object to control access to it.

### Types of Proxies
- **Virtual Proxy**: Lazy initialization of expensive objects
- **Protection Proxy**: Access control
- **Remote Proxy**: Local representative of remote object
- **Caching Proxy**: Cache results of expensive operations

### Implementation

```java
// Subject interface
public interface ImageService {
    byte[] loadImage(String imageId);
    void saveImage(String imageId, byte[] data);
    void deleteImage(String imageId);
}

// Real subject
public class CloudImageService implements ImageService {
    private final S3Client s3Client;
    private final String bucketName;
    
    public CloudImageService(S3Client s3Client, String bucketName) {
        this.s3Client = s3Client;
        this.bucketName = bucketName;
    }
    
    @Override
    public byte[] loadImage(String imageId) {
        // Expensive operation - downloads from cloud
        ResponseBytes<GetObjectResponse> response = s3Client.getObjectAsBytes(
            GetObjectRequest.builder()
                .bucket(bucketName)
                .key(imageId)
                .build()
        );
        return response.asByteArray();
    }
    
    @Override
    public void saveImage(String imageId, byte[] data) {
        s3Client.putObject(
            PutObjectRequest.builder()
                .bucket(bucketName)
                .key(imageId)
                .build(),
            RequestBody.fromBytes(data)
        );
    }
    
    @Override
    public void deleteImage(String imageId) {
        s3Client.deleteObject(
            DeleteObjectRequest.builder()
                .bucket(bucketName)
                .key(imageId)
                .build()
        );
    }
}

// Caching proxy
public class CachingImageProxy implements ImageService {
    private final ImageService realService;
    private final Cache<String, byte[]> cache;
    private final int maxCacheSize;
    
    public CachingImageProxy(ImageService realService, int maxCacheSize) {
        this.realService = realService;
        this.maxCacheSize = maxCacheSize;
        this.cache = Caffeine.newBuilder()
            .maximumSize(maxCacheSize)
            .expireAfterAccess(Duration.ofMinutes(30))
            .build();
    }
    
    @Override
    public byte[] loadImage(String imageId) {
        return cache.get(imageId, id -> {
            System.out.println("Cache miss, loading from cloud: " + id);
            return realService.loadImage(id);
        });
    }
    
    @Override
    public void saveImage(String imageId, byte[] data) {
        realService.saveImage(imageId, data);
        cache.put(imageId, data); // Update cache
    }
    
    @Override
    public void deleteImage(String imageId) {
        realService.deleteImage(imageId);
        cache.invalidate(imageId); // Remove from cache
    }
}

// Protection proxy
public class SecuredImageProxy implements ImageService {
    private final ImageService realService;
    private final AuthenticationService authService;
    private final Set<String> writeRoles = Set.of("ADMIN", "EDITOR");
    
    public SecuredImageProxy(ImageService realService, 
                            AuthenticationService authService) {
        this.realService = realService;
        this.authService = authService;
    }
    
    @Override
    public byte[] loadImage(String imageId) {
        // Anyone authenticated can read
        checkAuthenticated();
        return realService.loadImage(imageId);
    }
    
    @Override
    public void saveImage(String imageId, byte[] data) {
        checkHasWriteAccess();
        realService.saveImage(imageId, data);
    }
    
    @Override
    public void deleteImage(String imageId) {
        checkHasWriteAccess();
        realService.deleteImage(imageId);
    }
    
    private void checkAuthenticated() {
        if (!authService.isAuthenticated()) {
            throw new SecurityException("Authentication required");
        }
    }
    
    private void checkHasWriteAccess() {
        checkAuthenticated();
        String userRole = authService.getCurrentUserRole();
        if (!writeRoles.contains(userRole)) {
            throw new SecurityException("Write access denied for role: " + userRole);
        }
    }
}

// Logging proxy
public class LoggingImageProxy implements ImageService {
    private final ImageService realService;
    private static final Logger log = LoggerFactory.getLogger(LoggingImageProxy.class);
    
    public LoggingImageProxy(ImageService realService) {
        this.realService = realService;
    }
    
    @Override
    public byte[] loadImage(String imageId) {
        log.info("Loading image: {}", imageId);
        long start = System.currentTimeMillis();
        try {
            byte[] result = realService.loadImage(imageId);
            log.info("Loaded image: {} ({} bytes) in {} ms", 
                imageId, result.length, System.currentTimeMillis() - start);
            return result;
        } catch (Exception e) {
            log.error("Failed to load image: {}", imageId, e);
            throw e;
        }
    }
    
    @Override
    public void saveImage(String imageId, byte[] data) {
        log.info("Saving image: {} ({} bytes)", imageId, data.length);
        realService.saveImage(imageId, data);
        log.info("Image saved: {}", imageId);
    }
    
    @Override
    public void deleteImage(String imageId) {
        log.info("Deleting image: {}", imageId);
        realService.deleteImage(imageId);
        log.info("Image deleted: {}", imageId);
    }
}

// Chain proxies together
ImageService service = new CloudImageService(s3Client, "my-bucket");
service = new CachingImageProxy(service, 100);
service = new SecuredImageProxy(service, authService);
service = new LoggingImageProxy(service);
```

---

## Composite Pattern

### Intent
Compose objects into tree structures to represent part-whole hierarchies. Composite lets clients treat individual objects and compositions uniformly.

### When to Use
- Represent hierarchies of objects
- Clients should treat all objects in the composite uniformly
- File systems, UI components, organization structures

### Implementation

```java
// Component
public interface FileSystemComponent {
    String getName();
    long getSize();
    void display(String indent);
    List<FileSystemComponent> search(String name);
}

// Leaf
public class File implements FileSystemComponent {
    private final String name;
    private final long size;
    
    public File(String name, long size) {
        this.name = name;
        this.size = size;
    }
    
    @Override
    public String getName() { return name; }
    
    @Override
    public long getSize() { return size; }
    
    @Override
    public void display(String indent) {
        System.out.println(indent + "📄 " + name + " (" + formatSize(size) + ")");
    }
    
    @Override
    public List<FileSystemComponent> search(String searchName) {
        if (name.contains(searchName)) {
            return List.of(this);
        }
        return List.of();
    }
    
    private String formatSize(long bytes) {
        if (bytes < 1024) return bytes + " B";
        if (bytes < 1024 * 1024) return (bytes / 1024) + " KB";
        return (bytes / (1024 * 1024)) + " MB";
    }
}

// Composite
public class Directory implements FileSystemComponent {
    private final String name;
    private final List<FileSystemComponent> children = new ArrayList<>();
    
    public Directory(String name) {
        this.name = name;
    }
    
    public void add(FileSystemComponent component) {
        children.add(component);
    }
    
    public void remove(FileSystemComponent component) {
        children.remove(component);
    }
    
    @Override
    public String getName() { return name; }
    
    @Override
    public long getSize() {
        return children.stream()
            .mapToLong(FileSystemComponent::getSize)
            .sum();
    }
    
    @Override
    public void display(String indent) {
        System.out.println(indent + "📁 " + name + "/");
        for (FileSystemComponent child : children) {
            child.display(indent + "  ");
        }
    }
    
    @Override
    public List<FileSystemComponent> search(String searchName) {
        List<FileSystemComponent> results = new ArrayList<>();
        if (name.contains(searchName)) {
            results.add(this);
        }
        for (FileSystemComponent child : children) {
            results.addAll(child.search(searchName));
        }
        return results;
    }
}

// Usage
Directory root = new Directory("project");

Directory src = new Directory("src");
src.add(new File("Main.java", 2048));
src.add(new File("Utils.java", 1024));

Directory test = new Directory("test");
test.add(new File("MainTest.java", 1536));

root.add(src);
root.add(test);
root.add(new File("README.md", 512));
root.add(new File("pom.xml", 4096));

root.display("");
// Output:
// 📁 project/
//   📁 src/
//     📄 Main.java (2 KB)
//     📄 Utils.java (1 KB)
//   📁 test/
//     📄 MainTest.java (1 KB)
//   📄 README.md (512 B)
//   📄 pom.xml (4 KB)

System.out.println("Total size: " + root.getSize());
List<FileSystemComponent> found = root.search("Main");
```

---

## Bridge Pattern

### Intent
Decouple an abstraction from its implementation so that the two can vary independently.

### When to Use
- Avoid permanent binding between abstraction and implementation
- Both abstraction and implementation should be extensible
- Changes in implementation shouldn't affect clients

### Implementation

```java
// Implementation interface
public interface MessageSender {
    void send(String message, String recipient);
}

// Concrete implementations
public class EmailSender implements MessageSender {
    private final EmailClient emailClient;
    
    public EmailSender(EmailClient emailClient) {
        this.emailClient = emailClient;
    }
    
    @Override
    public void send(String message, String recipient) {
        emailClient.sendEmail(recipient, "Notification", message);
    }
}

public class SMSSender implements MessageSender {
    private final SMSGateway smsGateway;
    
    public SMSSender(SMSGateway smsGateway) {
        this.smsGateway = smsGateway;
    }
    
    @Override
    public void send(String message, String recipient) {
        smsGateway.sendSMS(recipient, message);
    }
}

public class SlackSender implements MessageSender {
    private final SlackClient slackClient;
    
    public SlackSender(SlackClient slackClient) {
        this.slackClient = slackClient;
    }
    
    @Override
    public void send(String message, String recipient) {
        slackClient.postMessage(recipient, message);
    }
}

// Abstraction
public abstract class Notification {
    protected final MessageSender sender;
    
    protected Notification(MessageSender sender) {
        this.sender = sender;
    }
    
    public abstract void notify(String recipient);
}

// Refined abstractions
public class AlertNotification extends Notification {
    private final String alertMessage;
    private final AlertLevel level;
    
    public AlertNotification(MessageSender sender, String alertMessage, AlertLevel level) {
        super(sender);
        this.alertMessage = alertMessage;
        this.level = level;
    }
    
    @Override
    public void notify(String recipient) {
        String formattedMessage = String.format("[%s ALERT] %s", level, alertMessage);
        sender.send(formattedMessage, recipient);
    }
}

public class ReminderNotification extends Notification {
    private final String task;
    private final LocalDateTime dueDate;
    
    public ReminderNotification(MessageSender sender, String task, LocalDateTime dueDate) {
        super(sender);
        this.task = task;
        this.dueDate = dueDate;
    }
    
    @Override
    public void notify(String recipient) {
        String formattedMessage = String.format(
            "Reminder: '%s' is due on %s", 
            task, 
            dueDate.format(DateTimeFormatter.ISO_LOCAL_DATE_TIME)
        );
        sender.send(formattedMessage, recipient);
    }
}

public class PromotionalNotification extends Notification {
    private final String title;
    private final String description;
    private final String promoCode;
    
    public PromotionalNotification(MessageSender sender, String title, 
                                   String description, String promoCode) {
        super(sender);
        this.title = title;
        this.description = description;
        this.promoCode = promoCode;
    }
    
    @Override
    public void notify(String recipient) {
        String formattedMessage = String.format(
            "🎉 %s\n%s\nUse code: %s", 
            title, description, promoCode
        );
        sender.send(formattedMessage, recipient);
    }
}

// Usage - mix and match independently
MessageSender emailSender = new EmailSender(emailClient);
MessageSender smsSender = new SMSSender(smsGateway);
MessageSender slackSender = new SlackSender(slackClient);

// Alert via email
Notification emailAlert = new AlertNotification(emailSender, "Server down!", AlertLevel.CRITICAL);
emailAlert.notify("admin@company.com");

// Alert via Slack
Notification slackAlert = new AlertNotification(slackSender, "Server down!", AlertLevel.CRITICAL);
slackAlert.notify("#ops-channel");

// Promo via SMS
Notification smsPromo = new PromotionalNotification(smsSender, "50% Off!", "Limited time offer", "SAVE50");
smsPromo.notify("+1234567890");
```

---

## Comparison Table

| Pattern | Intent | Key Characteristic |
|---------|--------|-------------------|
| **Adapter** | Convert interface | Wrapper that translates |
| **Decorator** | Add responsibilities | Wraps to enhance |
| **Facade** | Simplify subsystem | Single entry point |
| **Proxy** | Control access | Same interface, different behavior |
| **Composite** | Tree structures | Uniform treatment |
| **Bridge** | Decouple abstraction | Two hierarchies |

---

## Related Topics
- [[01-lld-fundamentals]] - SOLID principles
- [[02-creational-patterns]] - Factory, Builder, Singleton
- [[04-behavioral-patterns]] - Strategy, Observer, Command
