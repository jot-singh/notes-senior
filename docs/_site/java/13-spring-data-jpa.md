# Spring Data & JPA

## Concept Overview

Spring Data JPA simplifies data access by reducing boilerplate code through repository abstractions. JPA (Java Persistence API) provides ORM mapping between Java objects and relational databases. Together they enable rapid development of data access layers with powerful query capabilities.

## Core Mechanics & Implementation

### Entity Mapping

```java
@Entity
@Table(name = "users")
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(nullable = false, unique = true)
    private String email;
    
    @Column(nullable = false)
    private String name;
    
    @Enumerated(EnumType.STRING)
    private UserStatus status;
    
    @CreatedDate
    private LocalDateTime createdAt;
    
    @LastModifiedDate
    private LocalDateTime updatedAt;
    
    @Version
    private Long version;  // Optimistic locking
    
    @OneToMany(mappedBy = "user", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<Order> orders = new ArrayList<>();
    
    @ManyToMany
    @JoinTable(
        name = "user_roles",
        joinColumns = @JoinColumn(name = "user_id"),
        inverseJoinColumns = @JoinColumn(name = "role_id"))
    private Set<Role> roles = new HashSet<>();
    
    // Helper methods for bidirectional relationships
    public void addOrder(Order order) {
        orders.add(order);
        order.setUser(this);
    }
    
    public void removeOrder(Order order) {
        orders.remove(order);
        order.setUser(null);
    }
}

@Entity
@Table(name = "orders")
public class Order {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id", nullable = false)
    private User user;
    
    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<OrderItem> items = new ArrayList<>();
    
    private BigDecimal total;
    
    @Enumerated(EnumType.STRING)
    private OrderStatus status;
}
```

### Repository Pattern

```java
// Basic repository
public interface UserRepository extends JpaRepository<User, Long> {
    
    // Query derivation from method name
    Optional<User> findByEmail(String email);
    List<User> findByStatus(UserStatus status);
    List<User> findByNameContainingIgnoreCase(String name);
    List<User> findByCreatedAtAfter(LocalDateTime date);
    boolean existsByEmail(String email);
    long countByStatus(UserStatus status);
    
    // Custom JPQL query
    @Query("SELECT u FROM User u WHERE u.status = :status AND u.createdAt > :date")
    List<User> findActiveUsersCreatedAfter(
        @Param("status") UserStatus status,
        @Param("date") LocalDateTime date);
    
    // Native SQL query
    @Query(value = "SELECT * FROM users WHERE email LIKE %:domain", nativeQuery = true)
    List<User> findByEmailDomain(@Param("domain") String domain);
    
    // Modifying queries
    @Modifying
    @Query("UPDATE User u SET u.status = :status WHERE u.id = :id")
    int updateStatus(@Param("id") Long id, @Param("status") UserStatus status);
    
    @Modifying
    @Query("DELETE FROM User u WHERE u.status = :status AND u.createdAt < :date")
    int deleteInactiveUsers(@Param("status") UserStatus status, 
                           @Param("date") LocalDateTime date);
}

// Pagination and sorting
public interface OrderRepository extends JpaRepository<Order, Long> {
    
    Page<Order> findByUserId(Long userId, Pageable pageable);
    
    @Query("SELECT o FROM Order o WHERE o.status = :status")
    Page<Order> findByStatus(@Param("status") OrderStatus status, Pageable pageable);
    
    List<Order> findTop10ByOrderByCreatedAtDesc();
}

// Usage
Page<Order> orders = orderRepository.findByUserId(
    userId, 
    PageRequest.of(0, 20, Sort.by("createdAt").descending())
);
```

### Specifications for Dynamic Queries

```java
public class UserSpecifications {
    
    public static Specification<User> hasStatus(UserStatus status) {
        return (root, query, cb) -> 
            status == null ? null : cb.equal(root.get("status"), status);
    }
    
    public static Specification<User> nameLike(String name) {
        return (root, query, cb) -> 
            name == null ? null : cb.like(cb.lower(root.get("name")), 
                "%" + name.toLowerCase() + "%");
    }
    
    public static Specification<User> createdAfter(LocalDateTime date) {
        return (root, query, cb) -> 
            date == null ? null : cb.greaterThan(root.get("createdAt"), date);
    }
}

// Repository
public interface UserRepository extends JpaRepository<User, Long>, 
        JpaSpecificationExecutor<User> {
}

// Usage - dynamic query building
Specification<User> spec = Specification
    .where(UserSpecifications.hasStatus(status))
    .and(UserSpecifications.nameLike(name))
    .and(UserSpecifications.createdAfter(date));

List<User> users = userRepository.findAll(spec);
Page<User> pagedUsers = userRepository.findAll(spec, pageable);
```

### Projections

```java
// Interface projection (closed)
public interface UserSummary {
    Long getId();
    String getName();
    String getEmail();
}

// Interface projection (open)
public interface UserWithOrderCount {
    String getName();
    
    @Value("#{target.orders.size()}")
    int getOrderCount();
}

// Class projection (DTO)
public record UserDto(Long id, String name, String email) {}

// Repository methods
public interface UserRepository extends JpaRepository<User, Long> {
    
    List<UserSummary> findByStatus(UserStatus status);
    
    @Query("SELECT new com.example.dto.UserDto(u.id, u.name, u.email) FROM User u")
    List<UserDto> findAllUserDtos();
    
    <T> List<T> findByStatus(UserStatus status, Class<T> type);
}
```

### Transaction Management

```java
@Service
@Transactional(readOnly = true)  // Default for class
public class OrderService {
    
    private final OrderRepository orderRepository;
    private final UserRepository userRepository;
    
    @Transactional  // Read-write transaction
    public Order createOrder(Long userId, List<OrderItem> items) {
        User user = userRepository.findById(userId)
            .orElseThrow(() -> new UserNotFoundException(userId));
        
        Order order = new Order();
        order.setUser(user);
        items.forEach(order::addItem);
        order.calculateTotal();
        
        return orderRepository.save(order);
    }
    
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void auditOrder(Long orderId, String action) {
        // Runs in separate transaction
        auditRepository.save(new AuditLog(orderId, action));
    }
    
    @Transactional(isolation = Isolation.SERIALIZABLE)
    public void processPayment(Long orderId) {
        // Highest isolation level for critical operations
    }
}
```

### Fetch Optimization

```java
@Entity
public class User {
    // Lazy loading (default for collections)
    @OneToMany(mappedBy = "user", fetch = FetchType.LAZY)
    private List<Order> orders;
    
    // Eager loading (use sparingly)
    @ManyToOne(fetch = FetchType.EAGER)
    private Department department;
}

// Entity Graph for selective eager loading
@Entity
@NamedEntityGraph(
    name = "User.withOrders",
    attributeNodes = @NamedAttributeNode("orders"))
public class User { }

public interface UserRepository extends JpaRepository<User, Long> {
    
    @EntityGraph("User.withOrders")
    Optional<User> findWithOrdersById(Long id);
    
    @EntityGraph(attributePaths = {"orders", "roles"})
    @Query("SELECT u FROM User u WHERE u.id = :id")
    Optional<User> findWithOrdersAndRolesById(@Param("id") Long id);
}

// Join fetch in JPQL
@Query("SELECT u FROM User u LEFT JOIN FETCH u.orders WHERE u.id = :id")
Optional<User> findByIdWithOrders(@Param("id") Long id);
```

### Auditing

```java
@Configuration
@EnableJpaAuditing
public class JpaConfig {
    
    @Bean
    public AuditorAware<String> auditorProvider() {
        return () -> Optional.ofNullable(SecurityContextHolder.getContext())
            .map(SecurityContext::getAuthentication)
            .map(Authentication::getName);
    }
}

@MappedSuperclass
@EntityListeners(AuditingEntityListener.class)
public abstract class BaseEntity {
    
    @CreatedDate
    @Column(updatable = false)
    private LocalDateTime createdAt;
    
    @LastModifiedDate
    private LocalDateTime updatedAt;
    
    @CreatedBy
    @Column(updatable = false)
    private String createdBy;
    
    @LastModifiedBy
    private String updatedBy;
}
```

## Best Practices

- Use `FetchType.LAZY` as default, eager fetch selectively with EntityGraph
- Avoid N+1 queries with JOIN FETCH or EntityGraph
- Use projections for read-only queries returning subset of columns
- Prefer `@Version` for optimistic locking
- Keep transactions short, avoid long-running transactions
- Use batch processing for bulk operations
- Index frequently queried columns

## Interview Questions

**Q1: What is the N+1 problem and how to solve it?**

Loading collection triggers N additional queries for N parent entities. Solutions: JOIN FETCH in JPQL, @EntityGraph for selective loading, @BatchSize annotation, or use projections. Monitor with query logging.

**Q2: Explain FetchType.LAZY vs EAGER.**

LAZY: loads when accessed, default for collections, prevents loading unnecessary data. EAGER: loads immediately with parent, can cause performance issues, use sparingly. Override with EntityGraph or JOIN FETCH for specific queries.

**Q3: What is optimistic locking?**

Prevents concurrent modification without database locks. @Version field incremented on update. If version mismatch (someone else modified), throws OptimisticLockException. Caller handles retry or conflict resolution.

**Q4: How do query methods work in Spring Data?**

Method names parsed to build queries: findBy, countBy, deleteBy + property names + operators (And, Or, Between, Like). @Query for JPQL/SQL. Specifications for dynamic queries. Supports pagination and sorting.

**Q5: What is the difference between save() and saveAndFlush()?**

save(): schedules insert/update, executes at transaction commit or before queries needing fresh data. saveAndFlush(): immediately writes to database, useful when you need generated ID or must ensure write before subsequent queries.

**Q6: Explain cascade types.**

CascadeType controls operation propagation to related entities. PERSIST: save parent saves children. REMOVE: delete parent deletes children. MERGE/REFRESH: propagate merge/refresh. ALL: all operations. Use carefully to avoid unintended deletes.

**Q7: What is an EntityGraph?**

Defines which associations to fetch eagerly for specific query, overriding entity fetch types. More flexible than entity-level fetch configuration. Named or ad-hoc. Prevents N+1 without changing entity mappings.

**Q8: How to handle large batch operations?**

Use @Modifying queries for bulk updates/deletes. For inserts, batch with EntityManager.flush()/clear() every N records. Configure batch_size in Hibernate. Consider native queries or JDBC template for very large operations.
