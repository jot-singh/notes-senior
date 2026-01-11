# Behavioral Design Patterns

## Overview
Behavioral patterns are concerned with algorithms and the assignment of responsibilities between objects. They describe patterns of communication between objects and how the flow of control is distributed.

---

## Strategy Pattern

### Intent
Define a family of algorithms, encapsulate each one, and make them interchangeable. Strategy lets the algorithm vary independently from clients that use it.

### When to Use
- Need different variants of an algorithm
- Want to avoid conditional statements for selecting behaviors
- Have related classes that differ only in behavior

### Structure

```
┌─────────────────────────────────────────────────────────────────┐
│                       STRATEGY PATTERN                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌────────────────────────────────────────┐                     │
│  │              Context                    │                     │
│  ├────────────────────────────────────────┤                     │
│  │ - strategy: Strategy                    │                     │
│  │ + setStrategy(Strategy)                 │                     │
│  │ + executeStrategy()                     │──────┐              │
│  └────────────────────────────────────────┘      │              │
│                                                   │              │
│                                                   ▼              │
│                              ┌─────────────────────────────┐    │
│                              │      <<interface>>          │    │
│                              │        Strategy             │    │
│                              │   + execute()               │    │
│                              └─────────────────────────────┘    │
│                                          △                       │
│                      ┌───────────────────┼───────────────────┐  │
│                      │                   │                   │  │
│              ┌───────────────┐   ┌───────────────┐   ┌───────────────┐
│              │ StrategyA     │   │ StrategyB     │   │ StrategyC     │
│              └───────────────┘   └───────────────┘   └───────────────┘
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Implementation

```java
// Strategy interface
@FunctionalInterface
public interface PricingStrategy {
    BigDecimal calculatePrice(BigDecimal basePrice, Customer customer);
}

// Concrete strategies
public class RegularPricing implements PricingStrategy {
    @Override
    public BigDecimal calculatePrice(BigDecimal basePrice, Customer customer) {
        return basePrice;
    }
}

public class PremiumMemberPricing implements PricingStrategy {
    private final BigDecimal discountPercentage;
    
    public PremiumMemberPricing(BigDecimal discountPercentage) {
        this.discountPercentage = discountPercentage;
    }
    
    @Override
    public BigDecimal calculatePrice(BigDecimal basePrice, Customer customer) {
        BigDecimal discount = basePrice.multiply(discountPercentage)
            .divide(BigDecimal.valueOf(100), RoundingMode.HALF_UP);
        return basePrice.subtract(discount);
    }
}

public class LoyaltyPointsPricing implements PricingStrategy {
    private final int pointsPerDollar;
    
    public LoyaltyPointsPricing(int pointsPerDollar) {
        this.pointsPerDollar = pointsPerDollar;
    }
    
    @Override
    public BigDecimal calculatePrice(BigDecimal basePrice, Customer customer) {
        int pointsAvailable = customer.getLoyaltyPoints();
        BigDecimal maxDiscount = BigDecimal.valueOf(pointsAvailable)
            .divide(BigDecimal.valueOf(pointsPerDollar), 2, RoundingMode.DOWN);
        
        BigDecimal discount = maxDiscount.min(basePrice.multiply(new BigDecimal("0.20")));
        return basePrice.subtract(discount);
    }
}

public class BulkPricing implements PricingStrategy {
    private final Map<Integer, BigDecimal> tierDiscounts;
    
    public BulkPricing() {
        this.tierDiscounts = Map.of(
            10, new BigDecimal("5"),    // 5% off for 10+ items
            50, new BigDecimal("10"),   // 10% off for 50+ items
            100, new BigDecimal("15")   // 15% off for 100+ items
        );
    }
    
    @Override
    public BigDecimal calculatePrice(BigDecimal basePrice, Customer customer) {
        int quantity = customer.getCartQuantity();
        
        BigDecimal discountPercent = tierDiscounts.entrySet().stream()
            .filter(e -> quantity >= e.getKey())
            .max(Map.Entry.comparingByKey())
            .map(Map.Entry::getValue)
            .orElse(BigDecimal.ZERO);
        
        BigDecimal discount = basePrice.multiply(discountPercent)
            .divide(BigDecimal.valueOf(100), RoundingMode.HALF_UP);
        return basePrice.subtract(discount);
    }
}

// Context
public class ShoppingCart {
    private final List<CartItem> items = new ArrayList<>();
    private PricingStrategy pricingStrategy;
    
    public ShoppingCart() {
        this.pricingStrategy = new RegularPricing(); // Default strategy
    }
    
    public void setPricingStrategy(PricingStrategy strategy) {
        this.pricingStrategy = strategy;
    }
    
    public void addItem(CartItem item) {
        items.add(item);
    }
    
    public BigDecimal calculateTotal(Customer customer) {
        return items.stream()
            .map(item -> pricingStrategy.calculatePrice(item.getPrice(), customer)
                .multiply(BigDecimal.valueOf(item.getQuantity())))
            .reduce(BigDecimal.ZERO, BigDecimal::add);
    }
}

// Strategy factory
public class PricingStrategyFactory {
    public static PricingStrategy getStrategy(Customer customer) {
        if (customer.isPremiumMember()) {
            return new PremiumMemberPricing(new BigDecimal("15"));
        }
        if (customer.getLoyaltyPoints() > 1000) {
            return new LoyaltyPointsPricing(100);
        }
        if (customer.getCartQuantity() >= 10) {
            return new BulkPricing();
        }
        return new RegularPricing();
    }
}

// Usage
ShoppingCart cart = new ShoppingCart();
cart.addItem(new CartItem("Laptop", new BigDecimal("999.99"), 1));
cart.addItem(new CartItem("Mouse", new BigDecimal("49.99"), 2));

// Set strategy based on customer type
cart.setPricingStrategy(PricingStrategyFactory.getStrategy(customer));
BigDecimal total = cart.calculateTotal(customer);
```

### Java Lambda Implementation

```java
// Using functional interfaces
ShoppingCart cart = new ShoppingCart();

// Strategy as lambda
cart.setPricingStrategy((price, customer) -> 
    customer.isPremiumMember() 
        ? price.multiply(new BigDecimal("0.85"))  // 15% off
        : price
);
```

---

## Observer Pattern

### Intent
Define a one-to-many dependency between objects so that when one object changes state, all its dependents are notified and updated automatically.

### When to Use
- Changes in one object require changes in others (unknown how many)
- Object should notify others without knowing who they are
- Need loose coupling between related objects

### Implementation

```java
// Observer interface
public interface EventListener<T> {
    void onEvent(T event);
}

// Event types
public record OrderCreatedEvent(String orderId, String customerId, BigDecimal total) {}
public record OrderShippedEvent(String orderId, String trackingNumber) {}
public record OrderDeliveredEvent(String orderId, LocalDateTime deliveredAt) {}

// Subject / Event Publisher
public class EventPublisher<T> {
    private final Map<Class<?>, List<EventListener<?>>> listeners = new ConcurrentHashMap<>();
    
    public <E extends T> void subscribe(Class<E> eventType, EventListener<E> listener) {
        listeners.computeIfAbsent(eventType, k -> new CopyOnWriteArrayList<>())
            .add(listener);
    }
    
    public <E extends T> void unsubscribe(Class<E> eventType, EventListener<E> listener) {
        List<EventListener<?>> eventListeners = listeners.get(eventType);
        if (eventListeners != null) {
            eventListeners.remove(listener);
        }
    }
    
    @SuppressWarnings("unchecked")
    public <E extends T> void publish(E event) {
        List<EventListener<?>> eventListeners = listeners.get(event.getClass());
        if (eventListeners != null) {
            for (EventListener<?> listener : eventListeners) {
                ((EventListener<E>) listener).onEvent(event);
            }
        }
    }
}

// Concrete observers
public class EmailNotificationService implements EventListener<OrderCreatedEvent> {
    private final EmailClient emailClient;
    
    public EmailNotificationService(EmailClient emailClient) {
        this.emailClient = emailClient;
    }
    
    @Override
    public void onEvent(OrderCreatedEvent event) {
        emailClient.send(
            event.customerId(),
            "Order Confirmation",
            "Your order " + event.orderId() + " has been placed. Total: $" + event.total()
        );
    }
}

public class InventoryService implements EventListener<OrderCreatedEvent> {
    private final InventoryRepository repository;
    
    public InventoryService(InventoryRepository repository) {
        this.repository = repository;
    }
    
    @Override
    public void onEvent(OrderCreatedEvent event) {
        // Reserve inventory for the order
        repository.reserveInventory(event.orderId());
    }
}

public class AnalyticsService implements EventListener<OrderCreatedEvent> {
    private final MetricsCollector metrics;
    
    public AnalyticsService(MetricsCollector metrics) {
        this.metrics = metrics;
    }
    
    @Override
    public void onEvent(OrderCreatedEvent event) {
        metrics.increment("orders.created");
        metrics.recordValue("orders.total", event.total().doubleValue());
    }
}

public class ShippingTracker implements EventListener<OrderShippedEvent> {
    @Override
    public void onEvent(OrderShippedEvent event) {
        System.out.println("Order " + event.orderId() + 
            " shipped with tracking: " + event.trackingNumber());
    }
}

// Order service that publishes events
public class OrderService {
    private final OrderRepository orderRepository;
    private final EventPublisher<Object> eventPublisher;
    
    public OrderService(OrderRepository orderRepository, EventPublisher<Object> eventPublisher) {
        this.orderRepository = orderRepository;
        this.eventPublisher = eventPublisher;
    }
    
    public Order createOrder(CreateOrderRequest request) {
        Order order = Order.builder()
            .customerId(request.getCustomerId())
            .items(request.getItems())
            .status(OrderStatus.CREATED)
            .build();
        
        order = orderRepository.save(order);
        
        // Publish event - observers handle side effects
        eventPublisher.publish(new OrderCreatedEvent(
            order.getId(),
            order.getCustomerId(),
            order.getTotal()
        ));
        
        return order;
    }
    
    public void shipOrder(String orderId, String trackingNumber) {
        Order order = orderRepository.findById(orderId)
            .orElseThrow(() -> new OrderNotFoundException(orderId));
        
        order.setStatus(OrderStatus.SHIPPED);
        order.setTrackingNumber(trackingNumber);
        orderRepository.save(order);
        
        eventPublisher.publish(new OrderShippedEvent(orderId, trackingNumber));
    }
}

// Setup and usage
EventPublisher<Object> publisher = new EventPublisher<>();

// Register observers
publisher.subscribe(OrderCreatedEvent.class, new EmailNotificationService(emailClient));
publisher.subscribe(OrderCreatedEvent.class, new InventoryService(inventoryRepo));
publisher.subscribe(OrderCreatedEvent.class, new AnalyticsService(metrics));
publisher.subscribe(OrderShippedEvent.class, new ShippingTracker());

OrderService orderService = new OrderService(orderRepo, publisher);
orderService.createOrder(request); // All observers notified automatically
```

### Using Java's Built-in Observer Support

```java
// Using PropertyChangeSupport (Java standard library)
public class Stock {
    private final PropertyChangeSupport support = new PropertyChangeSupport(this);
    private String symbol;
    private BigDecimal price;
    
    public void addPropertyChangeListener(PropertyChangeListener listener) {
        support.addPropertyChangeListener(listener);
    }
    
    public void removePropertyChangeListener(PropertyChangeListener listener) {
        support.removePropertyChangeListener(listener);
    }
    
    public void setPrice(BigDecimal newPrice) {
        BigDecimal oldPrice = this.price;
        this.price = newPrice;
        support.firePropertyChange("price", oldPrice, newPrice);
    }
}

// Observer
Stock stock = new Stock("AAPL");
stock.addPropertyChangeListener(evt -> {
    System.out.println("Price changed from " + evt.getOldValue() + 
        " to " + evt.getNewValue());
});
```

---

## Command Pattern

### Intent
Encapsulate a request as an object, allowing parameterization of clients with different requests, queuing of requests, and support for undoable operations.

### When to Use
- Parameterize objects with operations
- Queue, log, or schedule operations
- Support undo/redo functionality
- Structure a system around high-level operations

### Implementation

```java
// Command interface
public interface Command {
    void execute();
    void undo();
    String getDescription();
}

// Receiver
public class TextEditor {
    private StringBuilder content = new StringBuilder();
    private int cursorPosition = 0;
    
    public void insert(String text, int position) {
        content.insert(position, text);
        cursorPosition = position + text.length();
    }
    
    public void delete(int start, int length) {
        content.delete(start, start + length);
        cursorPosition = start;
    }
    
    public String getContent() {
        return content.toString();
    }
    
    public int getCursorPosition() {
        return cursorPosition;
    }
    
    public void setCursorPosition(int position) {
        this.cursorPosition = position;
    }
}

// Concrete commands
public class InsertCommand implements Command {
    private final TextEditor editor;
    private final String text;
    private final int position;
    
    public InsertCommand(TextEditor editor, String text, int position) {
        this.editor = editor;
        this.text = text;
        this.position = position;
    }
    
    @Override
    public void execute() {
        editor.insert(text, position);
    }
    
    @Override
    public void undo() {
        editor.delete(position, text.length());
    }
    
    @Override
    public String getDescription() {
        return "Insert '" + text + "' at position " + position;
    }
}

public class DeleteCommand implements Command {
    private final TextEditor editor;
    private final int start;
    private final int length;
    private String deletedText;
    
    public DeleteCommand(TextEditor editor, int start, int length) {
        this.editor = editor;
        this.start = start;
        this.length = length;
    }
    
    @Override
    public void execute() {
        deletedText = editor.getContent().substring(start, start + length);
        editor.delete(start, length);
    }
    
    @Override
    public void undo() {
        editor.insert(deletedText, start);
    }
    
    @Override
    public String getDescription() {
        return "Delete " + length + " characters at position " + start;
    }
}

// Invoker with undo/redo support
public class CommandManager {
    private final Deque<Command> undoStack = new ArrayDeque<>();
    private final Deque<Command> redoStack = new ArrayDeque<>();
    private final int maxHistorySize;
    
    public CommandManager(int maxHistorySize) {
        this.maxHistorySize = maxHistorySize;
    }
    
    public void execute(Command command) {
        command.execute();
        undoStack.push(command);
        redoStack.clear(); // Clear redo stack on new command
        
        // Trim history if needed
        while (undoStack.size() > maxHistorySize) {
            undoStack.removeLast();
        }
    }
    
    public boolean canUndo() {
        return !undoStack.isEmpty();
    }
    
    public boolean canRedo() {
        return !redoStack.isEmpty();
    }
    
    public void undo() {
        if (canUndo()) {
            Command command = undoStack.pop();
            command.undo();
            redoStack.push(command);
        }
    }
    
    public void redo() {
        if (canRedo()) {
            Command command = redoStack.pop();
            command.execute();
            undoStack.push(command);
        }
    }
    
    public List<String> getHistory() {
        return undoStack.stream()
            .map(Command::getDescription)
            .collect(Collectors.toList());
    }
}

// Macro command (composite)
public class MacroCommand implements Command {
    private final List<Command> commands;
    private final String name;
    
    public MacroCommand(String name, List<Command> commands) {
        this.name = name;
        this.commands = new ArrayList<>(commands);
    }
    
    @Override
    public void execute() {
        commands.forEach(Command::execute);
    }
    
    @Override
    public void undo() {
        // Undo in reverse order
        List<Command> reversed = new ArrayList<>(commands);
        Collections.reverse(reversed);
        reversed.forEach(Command::undo);
    }
    
    @Override
    public String getDescription() {
        return "Macro: " + name;
    }
}

// Usage
TextEditor editor = new TextEditor();
CommandManager manager = new CommandManager(100);

manager.execute(new InsertCommand(editor, "Hello ", 0));
manager.execute(new InsertCommand(editor, "World", 6));
System.out.println(editor.getContent()); // "Hello World"

manager.undo();
System.out.println(editor.getContent()); // "Hello "

manager.redo();
System.out.println(editor.getContent()); // "Hello World"
```

### Command Pattern for Task Queue

```java
// Async command execution with queue
public class CommandQueue {
    private final BlockingQueue<Command> queue = new LinkedBlockingQueue<>();
    private final ExecutorService executor;
    private volatile boolean running = true;
    
    public CommandQueue(int workerCount) {
        this.executor = Executors.newFixedThreadPool(workerCount);
        for (int i = 0; i < workerCount; i++) {
            executor.submit(this::processCommands);
        }
    }
    
    public void submit(Command command) {
        queue.offer(command);
    }
    
    private void processCommands() {
        while (running) {
            try {
                Command command = queue.poll(100, TimeUnit.MILLISECONDS);
                if (command != null) {
                    command.execute();
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                break;
            }
        }
    }
    
    public void shutdown() {
        running = false;
        executor.shutdown();
    }
}
```

---

## Template Method Pattern

### Intent
Define the skeleton of an algorithm in a method, deferring some steps to subclasses. Template Method lets subclasses redefine certain steps without changing the algorithm's structure.

### When to Use
- Implement invariant parts of algorithm once
- Common behavior among subclasses should be centralized
- Control subclass extensions

### Implementation

```java
// Abstract class with template method
public abstract class DataProcessor {
    
    // Template method - defines the algorithm skeleton
    public final ProcessingResult process(DataSource source) {
        try {
            // Step 1: Validate
            validate(source);
            
            // Step 2: Extract data
            RawData rawData = extractData(source);
            
            // Step 3: Transform data
            ProcessedData processedData = transformData(rawData);
            
            // Step 4: Load/Save data
            String destination = loadData(processedData);
            
            // Step 5: Post-process (optional hook)
            postProcess(destination);
            
            return ProcessingResult.success(destination);
            
        } catch (ValidationException e) {
            return ProcessingResult.failure("Validation failed: " + e.getMessage());
        } catch (Exception e) {
            handleError(e);
            return ProcessingResult.failure("Processing failed: " + e.getMessage());
        }
    }
    
    // Abstract methods - must be implemented by subclasses
    protected abstract void validate(DataSource source) throws ValidationException;
    protected abstract RawData extractData(DataSource source);
    protected abstract ProcessedData transformData(RawData rawData);
    protected abstract String loadData(ProcessedData data);
    
    // Hook methods - can be overridden
    protected void postProcess(String destination) {
        // Default: do nothing
    }
    
    protected void handleError(Exception e) {
        // Default: log error
        System.err.println("Error during processing: " + e.getMessage());
    }
}

// Concrete implementation for CSV
public class CsvDataProcessor extends DataProcessor {
    private final CsvParser csvParser;
    private final DataWarehouse warehouse;
    
    public CsvDataProcessor(CsvParser csvParser, DataWarehouse warehouse) {
        this.csvParser = csvParser;
        this.warehouse = warehouse;
    }
    
    @Override
    protected void validate(DataSource source) throws ValidationException {
        if (!source.getPath().endsWith(".csv")) {
            throw new ValidationException("File must be a CSV file");
        }
        if (source.getSize() > 100_000_000) {
            throw new ValidationException("File too large (max 100MB)");
        }
    }
    
    @Override
    protected RawData extractData(DataSource source) {
        List<String[]> rows = csvParser.parse(source.getPath());
        return new RawData(rows.get(0), rows.subList(1, rows.size()));
    }
    
    @Override
    protected ProcessedData transformData(RawData rawData) {
        // Transform CSV rows to domain objects
        List<DataRecord> records = rawData.getRows().stream()
            .map(this::rowToRecord)
            .filter(this::isValidRecord)
            .collect(Collectors.toList());
        return new ProcessedData(records);
    }
    
    @Override
    protected String loadData(ProcessedData data) {
        String tableId = warehouse.createTable("imported_" + System.currentTimeMillis());
        warehouse.insertBatch(tableId, data.getRecords());
        return tableId;
    }
    
    @Override
    protected void postProcess(String destination) {
        warehouse.createIndex(destination, "id");
        warehouse.analyzeTable(destination);
    }
}

// Concrete implementation for JSON
public class JsonDataProcessor extends DataProcessor {
    private final ObjectMapper mapper;
    private final ElasticsearchClient esClient;
    
    @Override
    protected void validate(DataSource source) throws ValidationException {
        if (!source.getPath().endsWith(".json")) {
            throw new ValidationException("File must be a JSON file");
        }
    }
    
    @Override
    protected RawData extractData(DataSource source) {
        try {
            JsonNode root = mapper.readTree(new File(source.getPath()));
            return new RawData(root);
        } catch (IOException e) {
            throw new RuntimeException("Failed to parse JSON", e);
        }
    }
    
    @Override
    protected ProcessedData transformData(RawData rawData) {
        List<DataRecord> records = StreamSupport
            .stream(rawData.getJsonNode().spliterator(), false)
            .map(node -> mapper.convertValue(node, DataRecord.class))
            .collect(Collectors.toList());
        return new ProcessedData(records);
    }
    
    @Override
    protected String loadData(ProcessedData data) {
        String indexName = "data-" + LocalDate.now();
        esClient.bulkIndex(indexName, data.getRecords());
        return indexName;
    }
}

// Usage
DataProcessor csvProcessor = new CsvDataProcessor(csvParser, warehouse);
ProcessingResult result = csvProcessor.process(new DataSource("/data/sales.csv"));

DataProcessor jsonProcessor = new JsonDataProcessor(mapper, esClient);
ProcessingResult result2 = jsonProcessor.process(new DataSource("/data/events.json"));
```

---

## State Pattern

### Intent
Allow an object to alter its behavior when its internal state changes. The object will appear to change its class.

### When to Use
- Object behavior depends on its state and must change at runtime
- Operations have large conditional statements that depend on state
- State-specific behavior should be encapsulated

### Implementation

```java
// State interface
public interface OrderState {
    void next(OrderContext context);
    void previous(OrderContext context);
    void cancel(OrderContext context);
    String getStatus();
}

// Concrete states
public class PendingState implements OrderState {
    @Override
    public void next(OrderContext context) {
        context.setState(new ConfirmedState());
    }
    
    @Override
    public void previous(OrderContext context) {
        System.out.println("Order is already in initial state");
    }
    
    @Override
    public void cancel(OrderContext context) {
        context.setState(new CancelledState());
    }
    
    @Override
    public String getStatus() {
        return "PENDING";
    }
}

public class ConfirmedState implements OrderState {
    @Override
    public void next(OrderContext context) {
        context.setState(new ShippedState());
    }
    
    @Override
    public void previous(OrderContext context) {
        context.setState(new PendingState());
    }
    
    @Override
    public void cancel(OrderContext context) {
        context.setState(new CancelledState());
    }
    
    @Override
    public String getStatus() {
        return "CONFIRMED";
    }
}

public class ShippedState implements OrderState {
    @Override
    public void next(OrderContext context) {
        context.setState(new DeliveredState());
    }
    
    @Override
    public void previous(OrderContext context) {
        System.out.println("Cannot revert shipped order");
    }
    
    @Override
    public void cancel(OrderContext context) {
        System.out.println("Cannot cancel shipped order");
    }
    
    @Override
    public String getStatus() {
        return "SHIPPED";
    }
}

public class DeliveredState implements OrderState {
    @Override
    public void next(OrderContext context) {
        System.out.println("Order already delivered");
    }
    
    @Override
    public void previous(OrderContext context) {
        System.out.println("Cannot revert delivered order");
    }
    
    @Override
    public void cancel(OrderContext context) {
        System.out.println("Cannot cancel delivered order");
    }
    
    @Override
    public String getStatus() {
        return "DELIVERED";
    }
}

public class CancelledState implements OrderState {
    @Override
    public void next(OrderContext context) {
        System.out.println("Order is cancelled, cannot proceed");
    }
    
    @Override
    public void previous(OrderContext context) {
        System.out.println("Order is cancelled, cannot revert");
    }
    
    @Override
    public void cancel(OrderContext context) {
        System.out.println("Order is already cancelled");
    }
    
    @Override
    public String getStatus() {
        return "CANCELLED";
    }
}

// Context
public class OrderContext {
    private OrderState state;
    private final String orderId;
    private final List<StateChangeEvent> history = new ArrayList<>();
    
    public OrderContext(String orderId) {
        this.orderId = orderId;
        this.state = new PendingState();
        recordStateChange();
    }
    
    public void setState(OrderState state) {
        this.state = state;
        recordStateChange();
    }
    
    public void nextState() {
        state.next(this);
    }
    
    public void previousState() {
        state.previous(this);
    }
    
    public void cancel() {
        state.cancel(this);
    }
    
    public String getStatus() {
        return state.getStatus();
    }
    
    private void recordStateChange() {
        history.add(new StateChangeEvent(state.getStatus(), Instant.now()));
    }
    
    public List<StateChangeEvent> getHistory() {
        return Collections.unmodifiableList(history);
    }
}

// Usage
OrderContext order = new OrderContext("ORD-123");
System.out.println(order.getStatus()); // PENDING

order.nextState();
System.out.println(order.getStatus()); // CONFIRMED

order.nextState();
System.out.println(order.getStatus()); // SHIPPED

order.cancel(); // "Cannot cancel shipped order"

order.nextState();
System.out.println(order.getStatus()); // DELIVERED
```

---

## Chain of Responsibility Pattern

### Intent
Avoid coupling the sender of a request to its receiver by giving more than one object a chance to handle the request. Chain the receiving objects and pass the request along until an object handles it.

### When to Use
- More than one object may handle a request
- Handler isn't known beforehand
- Set of handlers should be specified dynamically

### Implementation

```java
// Handler interface
public abstract class RequestHandler {
    protected RequestHandler next;
    
    public RequestHandler setNext(RequestHandler next) {
        this.next = next;
        return next;
    }
    
    public final Response handle(Request request) {
        if (canHandle(request)) {
            return doHandle(request);
        }
        if (next != null) {
            return next.handle(request);
        }
        return Response.unhandled("No handler found for request");
    }
    
    protected abstract boolean canHandle(Request request);
    protected abstract Response doHandle(Request request);
}

// Concrete handlers
public class AuthenticationHandler extends RequestHandler {
    private final TokenValidator tokenValidator;
    
    public AuthenticationHandler(TokenValidator tokenValidator) {
        this.tokenValidator = tokenValidator;
    }
    
    @Override
    protected boolean canHandle(Request request) {
        return true; // Always check authentication first
    }
    
    @Override
    protected Response doHandle(Request request) {
        String token = request.getHeader("Authorization");
        if (token == null || !tokenValidator.isValid(token)) {
            return Response.unauthorized("Invalid or missing token");
        }
        
        // Set user context
        request.setAttribute("user", tokenValidator.getUser(token));
        
        // Pass to next handler
        if (next != null) {
            return next.handle(request);
        }
        return Response.ok();
    }
}

public class RateLimitHandler extends RequestHandler {
    private final RateLimiter rateLimiter;
    
    public RateLimitHandler(RateLimiter rateLimiter) {
        this.rateLimiter = rateLimiter;
    }
    
    @Override
    protected boolean canHandle(Request request) {
        return true; // Always check rate limit
    }
    
    @Override
    protected Response doHandle(Request request) {
        String clientId = request.getClientId();
        if (!rateLimiter.allowRequest(clientId)) {
            return Response.tooManyRequests("Rate limit exceeded");
        }
        
        if (next != null) {
            return next.handle(request);
        }
        return Response.ok();
    }
}

public class ValidationHandler extends RequestHandler {
    private final Map<String, Validator<?>> validators;
    
    public ValidationHandler(Map<String, Validator<?>> validators) {
        this.validators = validators;
    }
    
    @Override
    protected boolean canHandle(Request request) {
        return validators.containsKey(request.getPath());
    }
    
    @Override
    protected Response doHandle(Request request) {
        Validator<?> validator = validators.get(request.getPath());
        ValidationResult result = validator.validate(request.getBody());
        
        if (!result.isValid()) {
            return Response.badRequest(result.getErrors());
        }
        
        if (next != null) {
            return next.handle(request);
        }
        return Response.ok();
    }
}

public class BusinessLogicHandler extends RequestHandler {
    private final Map<String, RequestProcessor> processors;
    
    public BusinessLogicHandler(Map<String, RequestProcessor> processors) {
        this.processors = processors;
    }
    
    @Override
    protected boolean canHandle(Request request) {
        return processors.containsKey(request.getPath());
    }
    
    @Override
    protected Response doHandle(Request request) {
        RequestProcessor processor = processors.get(request.getPath());
        return processor.process(request);
    }
}

// Chain setup
public class RequestPipeline {
    private final RequestHandler chain;
    
    public RequestPipeline(TokenValidator tokenValidator,
                          RateLimiter rateLimiter,
                          Map<String, Validator<?>> validators,
                          Map<String, RequestProcessor> processors) {
        
        RequestHandler auth = new AuthenticationHandler(tokenValidator);
        RequestHandler rateLimit = new RateLimitHandler(rateLimiter);
        RequestHandler validation = new ValidationHandler(validators);
        RequestHandler business = new BusinessLogicHandler(processors);
        
        // Build chain
        auth.setNext(rateLimit)
            .setNext(validation)
            .setNext(business);
        
        this.chain = auth;
    }
    
    public Response process(Request request) {
        return chain.handle(request);
    }
}

// Usage
RequestPipeline pipeline = new RequestPipeline(
    tokenValidator, rateLimiter, validators, processors
);

Response response = pipeline.process(incomingRequest);
```

---

## Comparison Table

| Pattern | Intent | Key Use Case |
|---------|--------|--------------|
| **Strategy** | Interchangeable algorithms | Pricing, sorting, validation |
| **Observer** | Event notification | Pub/sub, event handling |
| **Command** | Encapsulate request | Undo/redo, task queues |
| **Template Method** | Algorithm skeleton | Data processing, workflows |
| **State** | State-dependent behavior | Order status, workflows |
| **Chain of Responsibility** | Pass request along chain | Filters, middleware |

---

## Quick Reference

```java
// Strategy - swap algorithms
context.setStrategy(new ConcreteStrategy());
context.executeStrategy();

// Observer - notify subscribers
publisher.subscribe(listener);
publisher.publish(event);

// Command - queue/undo operations
manager.execute(new ConcreteCommand());
manager.undo();

// Template Method - define skeleton
abstract class Template {
    final void execute() { step1(); step2(); } // fixed
    abstract void step1(); // customizable
}

// State - change behavior with state
context.setState(new NewState());

// Chain - pass along handlers
handler1.setNext(handler2);
handler1.handle(request);
```

---

## Related Topics
- [[01-lld-fundamentals]] - SOLID principles
- [[02-creational-patterns]] - Factory, Builder, Singleton
- [[03-structural-patterns]] - Adapter, Decorator, Facade
