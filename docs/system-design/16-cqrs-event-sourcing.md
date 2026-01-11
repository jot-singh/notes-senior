# CQRS and Event Sourcing

## What You'll Learn

- Command Query Responsibility Segregation pattern
- Event Sourcing fundamentals
- Event Store implementation
- Projections and read models
- Combining CQRS with Event Sourcing

## Why This Matters

Traditional CRUD systems struggle with complex domains, high read/write ratios, and audit requirements. CQRS separates reads from writes, allowing independent scaling. Event Sourcing stores state changes as immutable events, providing perfect audit trail and time-travel debugging. Companies like Netflix, Uber, and Microsoft use these patterns for scalability and business intelligence. Understanding when to apply CQRS+ES versus traditional approaches is crucial for senior architects.

## CQRS (Command Query Responsibility Segregation)

Separate models for reading and writing data. Commands modify state, queries retrieve data.

### Basic CQRS Implementation

```java
// Command side - handles writes
public class OrderCommandService {
    private final OrderRepository orderRepository;
    private final EventBus eventBus;
    
    public void createOrder(CreateOrderCommand command) {
        // Validate command
        if (command.getItems().isEmpty()) {
            throw new InvalidCommandException("Order must have items");
        }
        
        // Create aggregate
        Order order = new Order(
            command.getUserId(),
            command.getItems(),
            command.getShippingAddress()
        );
        
        // Persist
        orderRepository.save(order);
        
        // Publish event for read model
        eventBus.publish(new OrderCreatedEvent(
            order.getId(),
            order.getUserId(),
            order.getTotal(),
            Instant.now()
        ));
    }
    
    public void cancelOrder(CancelOrderCommand command) {
        Order order = orderRepository.findById(command.getOrderId())
            .orElseThrow(() -> new OrderNotFoundException());
        
        order.cancel(command.getReason());
        orderRepository.save(order);
        
        eventBus.publish(new OrderCancelledEvent(
            order.getId(),
            command.getReason(),
            Instant.now()
        ));
    }
}

// Query side - handles reads
public class OrderQueryService {
    private final OrderReadRepository readRepository;
    
    public OrderSummary getOrderSummary(String orderId) {
        // Read from optimized read model
        return readRepository.findOrderSummary(orderId);
    }
    
    public List<OrderListItem> getUserOrders(String userId, Pagination page) {
        // Query denormalized view
        return readRepository.findByUserId(userId, page);
    }
    
    public OrderStatistics getStatistics(LocalDate startDate, LocalDate endDate) {
        // Complex query on pre-aggregated data
        return readRepository.getStatistics(startDate, endDate);
    }
}

// Read model updater (event handler)
@Component
public class OrderReadModelUpdater {
    private final OrderReadRepository readRepository;
    
    @EventHandler
    public void on(OrderCreatedEvent event) {
        OrderSummary summary = new OrderSummary();
        summary.setOrderId(event.getOrderId());
        summary.setUserId(event.getUserId());
        summary.setTotal(event.getTotal());
        summary.setStatus("CREATED");
        summary.setCreatedAt(event.getTimestamp());
        
        readRepository.save(summary);
    }
    
    @EventHandler
    public void on(OrderCancelledEvent event) {
        OrderSummary summary = readRepository.findById(event.getOrderId());
        summary.setStatus("CANCELLED");
        summary.setCancelReason(event.getReason());
        summary.setUpdatedAt(event.getTimestamp());
        
        readRepository.save(summary);
    }
}
```

### Separate Databases for Read/Write

```python
# Write model - normalized, transactional
class OrderWriteModel:
    def __init__(self, write_db):
        self.db = write_db
    
    def create_order(self, user_id, items):
        with self.db.transaction():
            order_id = self.db.insert('orders', {
                'user_id': user_id,
                'status': 'pending',
                'created_at': datetime.now()
            })
            
            for item in items:
                self.db.insert('order_items', {
                    'order_id': order_id,
                    'product_id': item['product_id'],
                    'quantity': item['quantity'],
                    'price': item['price']
                })
            
            return order_id

# Read model - denormalized, optimized for queries
class OrderReadModel:
    def __init__(self, read_db):
        self.db = read_db  # Could be MongoDB, Elasticsearch, etc.
    
    def get_order_details(self, order_id):
        # Single document with all order information
        return self.db.collection('order_views').find_one({
            '_id': order_id
        })
    
    def search_orders(self, filters):
        # Full-text search, faceted navigation
        return self.db.collection('order_views').find(filters)
    
    def update_from_event(self, event):
        """Update read model from domain event"""
        if event.type == 'OrderCreated':
            self.db.collection('order_views').insert_one({
                '_id': event.order_id,
                'user_id': event.user_id,
                'items': event.items,
                'total': event.total,
                'status': 'pending',
                'created_at': event.timestamp
            })
        elif event.type == 'OrderShipped':
            self.db.collection('order_views').update_one(
                {'_id': event.order_id},
                {'$set': {
                    'status': 'shipped',
                    'tracking_number': event.tracking_number
                }}
            )
```

## Event Sourcing

Store state changes as immutable events instead of current state.

### Event Store Implementation

```java
// Event store
public class EventStore {
    private final JdbcTemplate jdbcTemplate;
    private final ObjectMapper objectMapper;
    
    public void appendEvent(String aggregateId, DomainEvent event, long expectedVersion) {
        String eventType = event.getClass().getSimpleName();
        String eventData = objectMapper.writeValueAsString(event);
        
        try {
            jdbcTemplate.update(
                "INSERT INTO event_store (aggregate_id, event_type, event_data, version, timestamp) " +
                "VALUES (?, ?, ?, ?, ?)",
                aggregateId,
                eventType,
                eventData,
                expectedVersion + 1,
                Instant.now()
            );
        } catch (DuplicateKeyException e) {
            // Optimistic concurrency check failed
            throw new ConcurrencyException("Expected version " + expectedVersion);
        }
    }
    
    public List<DomainEvent> getEvents(String aggregateId) {
        return jdbcTemplate.query(
            "SELECT event_type, event_data FROM event_store " +
            "WHERE aggregate_id = ? ORDER BY version",
            (rs, rowNum) -> deserializeEvent(
                rs.getString("event_type"),
                rs.getString("event_data")
            ),
            aggregateId
        );
    }
    
    public List<DomainEvent> getEventsSince(long position) {
        return jdbcTemplate.query(
            "SELECT aggregate_id, event_type, event_data FROM event_store " +
            "WHERE id > ? ORDER BY id",
            (rs, rowNum) -> deserializeEvent(
                rs.getString("event_type"),
                rs.getString("event_data")
            ),
            position
        );
    }
    
    private DomainEvent deserializeEvent(String eventType, String eventData) {
        try {
            Class<?> eventClass = Class.forName("com.example.events." + eventType);
            return (DomainEvent) objectMapper.readValue(eventData, eventClass);
        } catch (Exception e) {
            throw new RuntimeException("Failed to deserialize event", e);
        }
    }
}

// Event store schema
/*
CREATE TABLE event_store (
    id BIGSERIAL PRIMARY KEY,
    aggregate_id VARCHAR(255) NOT NULL,
    event_type VARCHAR(255) NOT NULL,
    event_data JSONB NOT NULL,
    version INTEGER NOT NULL,
    timestamp TIMESTAMP NOT NULL,
    UNIQUE(aggregate_id, version)
);

CREATE INDEX idx_aggregate_id ON event_store(aggregate_id);
CREATE INDEX idx_timestamp ON event_store(timestamp);
*/
```

### Aggregate with Event Sourcing

```java
// Event-sourced aggregate
public class Order {
    private String id;
    private String userId;
    private OrderStatus status;
    private List<OrderItem> items = new ArrayList<>();
    private long version = 0;
    
    // List of uncommitted events
    private List<DomainEvent> uncommittedEvents = new ArrayList<>();
    
    // Constructor for new aggregate
    public Order(String userId, List<OrderItem> items) {
        apply(new OrderCreatedEvent(
            UUID.randomUUID().toString(),
            userId,
            items,
            Instant.now()
        ));
    }
    
    // Constructor to rebuild from events
    public static Order fromEvents(List<DomainEvent> events) {
        Order order = new Order();
        for (DomainEvent event : events) {
            order.applyEvent(event);
            order.version++;
        }
        return order;
    }
    
    // Command handler
    public void ship(String trackingNumber) {
        if (status != OrderStatus.PAID) {
            throw new IllegalStateException("Cannot ship unpaid order");
        }
        
        apply(new OrderShippedEvent(
            id,
            trackingNumber,
            Instant.now()
        ));
    }
    
    // Apply event and add to uncommitted
    private void apply(DomainEvent event) {
        applyEvent(event);
        uncommittedEvents.add(event);
    }
    
    // Apply event to state (used for both new events and replay)
    private void applyEvent(DomainEvent event) {
        if (event instanceof OrderCreatedEvent) {
            OrderCreatedEvent e = (OrderCreatedEvent) event;
            this.id = e.getOrderId();
            this.userId = e.getUserId();
            this.items = new ArrayList<>(e.getItems());
            this.status = OrderStatus.PENDING;
        } else if (event instanceof OrderShippedEvent) {
            OrderShippedEvent e = (OrderShippedEvent) event;
            this.status = OrderStatus.SHIPPED;
        }
    }
    
    public List<DomainEvent> getUncommittedEvents() {
        return new ArrayList<>(uncommittedEvents);
    }
    
    public void clearUncommittedEvents() {
        uncommittedEvents.clear();
    }
    
    public long getVersion() {
        return version;
    }
}

// Repository for event-sourced aggregates
public class EventSourcedOrderRepository {
    private final EventStore eventStore;
    
    public void save(Order order) {
        List<DomainEvent> events = order.getUncommittedEvents();
        
        for (DomainEvent event : events) {
            eventStore.appendEvent(order.getId(), event, order.getVersion());
        }
        
        order.clearUncommittedEvents();
    }
    
    public Order findById(String orderId) {
        List<DomainEvent> events = eventStore.getEvents(orderId);
        
        if (events.isEmpty()) {
            throw new AggregateNotFoundException(orderId);
        }
        
        return Order.fromEvents(events);
    }
}
```

## Projections

Transform event stream into queryable views.

```javascript
// Projection builder
class OrderProjection {
    constructor(eventStore, database) {
        this.eventStore = eventStore;
        this.database = database;
    }
    
    async buildProjection() {
        // Rebuild entire projection from scratch
        await this.database.truncate('order_projections');
        
        let position = 0;
        while (true) {
            const events = await this.eventStore.getEventsSince(position, 1000);
            
            if (events.length === 0) break;
            
            for (const event of events) {
                await this.handleEvent(event);
                position = event.position;
            }
        }
    }
    
    async handleEvent(event) {
        switch (event.type) {
            case 'OrderCreated':
                await this.database.insert('order_projections', {
                    order_id: event.orderId,
                    user_id: event.userId,
                    total: event.total,
                    status: 'pending',
                    created_at: event.timestamp
                });
                break;
                
            case 'OrderShipped':
                await this.database.update('order_projections',
                    { order_id: event.orderId },
                    { 
                        status: 'shipped',
                        shipped_at: event.timestamp,
                        tracking_number: event.trackingNumber
                    }
                );
                break;
                
            case 'OrderCancelled':
                await this.database.update('order_projections',
                    { order_id: event.orderId },
                    { 
                        status: 'cancelled',
                        cancelled_at: event.timestamp
                    }
                );
                break;
        }
    }
    
    // Real-time projection updates
    subscribeToEvents() {
        this.eventStore.subscribe((event) => {
            this.handleEvent(event);
        });
    }
}

// Multiple projections for different views
class OrderStatisticsProjection {
    async handleEvent(event) {
        if (event.type === 'OrderCreated') {
            // Update daily statistics
            const date = event.timestamp.toDateString();
            await this.database.query(`
                INSERT INTO daily_order_stats (date, order_count, revenue)
                VALUES (?, 1, ?)
                ON DUPLICATE KEY UPDATE 
                    order_count = order_count + 1,
                    revenue = revenue + ?
            `, [date, event.total, event.total]);
        }
    }
}
```

## Snapshotting

Optimize performance by periodically saving aggregate state.

```python
class SnapshotStore:
    def __init__(self, database):
        self.db = database
    
    def save_snapshot(self, aggregate_id, aggregate, version):
        """Save aggregate snapshot"""
        self.db.execute("""
            INSERT INTO snapshots (aggregate_id, snapshot_data, version, created_at)
            VALUES (?, ?, ?, ?)
            ON CONFLICT (aggregate_id) DO UPDATE
            SET snapshot_data = ?, version = ?, created_at = ?
        """, (
            aggregate_id,
            pickle.dumps(aggregate),
            version,
            datetime.now(),
            pickle.dumps(aggregate),
            version,
            datetime.now()
        ))
    
    def get_snapshot(self, aggregate_id):
        """Get latest snapshot"""
        result = self.db.query_one("""
            SELECT snapshot_data, version
            FROM snapshots
            WHERE aggregate_id = ?
        """, (aggregate_id,))
        
        if result:
            return pickle.loads(result['snapshot_data']), result['version']
        return None, 0

class EventSourcedRepository:
    def __init__(self, event_store, snapshot_store):
        self.event_store = event_store
        self.snapshot_store = snapshot_store
        self.snapshot_frequency = 100  # Snapshot every 100 events
    
    def load(self, aggregate_id):
        """Load aggregate from snapshot + events"""
        # Get latest snapshot
        aggregate, version = self.snapshot_store.get_snapshot(aggregate_id)
        
        if aggregate is None:
            aggregate = Order()
            version = 0
        
        # Load events since snapshot
        events = self.event_store.get_events_after(aggregate_id, version)
        
        # Replay events
        for event in events:
            aggregate.apply_event(event)
            version = event.version
        
        return aggregate
    
    def save(self, aggregate):
        """Save events and maybe snapshot"""
        events = aggregate.get_uncommitted_events()
        
        for event in events:
            self.event_store.append(aggregate.id, event)
        
        # Snapshot if threshold reached
        if aggregate.version % self.snapshot_frequency == 0:
            self.snapshot_store.save_snapshot(
                aggregate.id,
                aggregate,
                aggregate.version
            )
        
        aggregate.clear_uncommitted_events()
```

## Event Versioning

Handle schema evolution in events.

```java
// Event upcasting
public interface EventUpcaster {
    DomainEvent upcast(DomainEvent event);
    boolean canUpcast(DomainEvent event);
}

public class OrderCreatedV1ToV2Upcaster implements EventUpcaster {
    @Override
    public boolean canUpcast(DomainEvent event) {
        return event instanceof OrderCreatedEventV1;
    }
    
    @Override
    public DomainEvent upcast(DomainEvent event) {
        OrderCreatedEventV1 v1 = (OrderCreatedEventV1) event;
        
        // V2 adds shipping address field
        return new OrderCreatedEventV2(
            v1.getOrderId(),
            v1.getUserId(),
            v1.getItems(),
            getDefaultShippingAddress(v1.getUserId()), // Add missing field
            v1.getTimestamp()
        );
    }
}

public class EventStoreWithUpcasting {
    private final EventStore eventStore;
    private final List<EventUpcaster> upcasters;
    
    public List<DomainEvent> getEvents(String aggregateId) {
        List<DomainEvent> events = eventStore.getEvents(aggregateId);
        
        // Apply upcasters
        return events.stream()
            .map(this::upcastEvent)
            .collect(Collectors.toList());
    }
    
    private DomainEvent upcastEvent(DomainEvent event) {
        for (EventUpcaster upcaster : upcasters) {
            if (upcaster.canUpcast(event)) {
                event = upcaster.upcast(event);
            }
        }
        return event;
    }
}
```

## Best Practices

**Immutable Events**: Never modify published events. Use upcasting for schema evolution.

**Idempotent Projections**: Projections may process same event multiple times. Ensure idempotency.

**Event Versioning**: Include version in event type. Plan for schema evolution from day one.

**Snapshot Strategy**: Snapshot after N events or based on aggregate complexity.

**Separate Event Store**: Don't mix events with application data. Events are append-only, immutable.

**Monitor Projection Lag**: Track how far behind projections are from event stream.

## Anti-Patterns

❌ **CQRS Everywhere**: Most CRUD operations don't need CQRS complexity

❌ **Event Sourcing Without CQRS**: Event sourcing creates write-optimized store. Need read models.

❌ **Mutable Events**: Changing past events breaks audit trail and replays

❌ **No Snapshots**: Replaying thousands of events is slow

❌ **Synchronous Projections**: Don't block command processing on projection updates

## Real-World Examples

**Microsoft**: Azure Event Hubs and CosmosDB support event sourcing at scale.

**Netflix**: Uses event sourcing for user viewing history. Perfect for analytics and recommendations.

**Uber**: Trip state managed with event sourcing. Easy to replay trips and debug issues.

**Stack Overflow**: Uses CQRS to separate read-heavy queries from writes. Elasticsearch for reads, SQL for writes.

## Related Topics

- [Distributed Transactions](15-distributed-transactions.md)
- [Message Queues](06-message-queues.md)
- [Database Scaling Strategies](03-database-scaling-strategies.md)
- [Monitoring & Observability](13-monitoring-observability.md)
